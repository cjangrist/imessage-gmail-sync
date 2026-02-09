# 04 — Conversation Threading & Email Formatting

## Purpose

Two modules work together:
- `src/threading_engine.py` — generates deterministic `Message-ID`, `In-Reply-To`, and `References` headers that force Gmail to group all messages from one iMessage chat into a single conversation thread.
- `src/formatter.py` — converts iMessage data into human-friendly, multipart RFC 2822 emails that look good in Gmail's web UI and mobile app.

---

## Part A: Threading Engine (`threading_engine.py`)

### Gmail Threading Rules (What We Must Satisfy)

Gmail groups messages into a conversation when **all** of these hold:

1. **Subject matches** — identical, or with `Re:` / `Fwd:` prefix
2. **`In-Reply-To` header** — references a `Message-ID` already in the thread
3. **`References` header** — contains `Message-ID`s from the thread chain
4. **Same participants** — sender must be part of the existing thread (exception: `In-Reply-To` is set)

IMAP APPEND with explicit headers bypasses the "within 1 week" heuristic that applies to SMTP-delivered mail.

### Deterministic Message-ID Scheme

Every iMessage gets a globally unique, deterministic, reproducible `Message-ID`:

```
<imsg-{chat_id}-{message_rowid}@imessage-gmail-sync>
```

Examples:
```
<imsg-42-12847@imessage-gmail-sync>     # chat 42, message ROWID 12847
<imsg-42-12848@imessage-gmail-sync>     # next message in same chat
<imsg-7-501@imessage-gmail-sync>        # different chat
```

Why this works:
- **Deterministic**: same input always produces same Message-ID → idempotent re-syncs
- **Globally unique**: `(chat_id, rowid)` is a unique composite key in `chat.db`
- **Valid RFC 2822**: `<local-part@domain>` format
- **Searchable**: IMAP `SEARCH HEADER Message-ID <imsg-42-12847@...>` for dedup

### Thread Chain Construction

For each chat, messages are processed **in chronological order** (ascending by ROWID). The threading headers build up a chain:

```
Message 1 (ROWID 100) — FIRST message in chat:
  Message-ID: <imsg-42-100@imessage-gmail-sync>
  Subject: iMessage: Family Group
  In-Reply-To: (absent)
  References: (absent)

Message 2 (ROWID 105) — SECOND message:
  Message-ID: <imsg-42-105@imessage-gmail-sync>
  Subject: Re: iMessage: Family Group
  In-Reply-To: <imsg-42-100@imessage-gmail-sync>
  References: <imsg-42-100@imessage-gmail-sync>

Message 3 (ROWID 110) — THIRD message:
  Message-ID: <imsg-42-110@imessage-gmail-sync>
  Subject: Re: iMessage: Family Group
  In-Reply-To: <imsg-42-105@imessage-gmail-sync>
  References: <imsg-42-100@imessage-gmail-sync> <imsg-42-105@imessage-gmail-sync>

Message N:
  In-Reply-To: <Message-ID of message N-1>
  References: <Message-ID of message 1> <Message-ID of message 2> ... <Message-ID of message N-1>
```

### References Header Truncation

Gmail and RFC 2822 have no hard limit on `References` length, but for chats with 1000+ messages the header gets unwieldy. Strategy:

- Keep the **first** Message-ID (thread anchor) always
- Keep the **last 20** Message-IDs (recent context)
- Drop middle entries if chain exceeds 25 entries

This preserves Gmail's ability to thread correctly (it primarily uses the first and most recent entries).

### Implementation

```python
# src/threading_engine.py
"""Deterministic email threading headers for iMessage → Gmail conversations."""

DOMAIN = "imessage-gmail-sync"
MAX_REFERENCES_COUNT = 25
KEEP_RECENT_COUNT = 20


def generate_message_id(chat_id: int, message_rowid: int) -> str:
    """
    Generate a deterministic RFC 2822 Message-ID for an iMessage.

    Returns: "<imsg-{chat_id}-{message_rowid}@imessage-gmail-sync>"
    """
    return f"<imsg-{chat_id}-{message_rowid}@{DOMAIN}>"


def build_thread_headers(
    chat_id: int,
    message_rowid: int,
    previous_message_ids: list[str],
    chat_display_name: str,
) -> dict:
    """
    Build Message-ID, In-Reply-To, References, and Subject headers.

    Args:
        chat_id: The iMessage chat ID
        message_rowid: This message's ROWID
        previous_message_ids: List of Message-IDs for all prior messages
                              in this chat (in chronological order).
                              Empty list for the first message.
        chat_display_name: Human-readable chat name for Subject line

    Returns dict with keys:
        "Message-ID": str
        "Subject": str
        "In-Reply-To": str or None
        "References": str or None
    """
    current_message_id = generate_message_id(chat_id, message_rowid)
    is_first_message = len(previous_message_ids) == 0

    subject = (
        f"iMessage: {chat_display_name}"
        if is_first_message
        else f"Re: iMessage: {chat_display_name}"
    )

    in_reply_to = None if is_first_message else previous_message_ids[-1]

    references = None
    if not is_first_message:
        refs = list(previous_message_ids)
        if len(refs) > MAX_REFERENCES_COUNT:
            # Keep first (anchor) + last N (recent)
            refs = [refs[0]] + refs[-(KEEP_RECENT_COUNT):]
        references = " ".join(refs)

    return {
        "Message-ID": current_message_id,
        "Subject": subject,
        "In-Reply-To": in_reply_to,
        "References": references,
    }
```

### Chat Display Name Resolution

```python
def resolve_chat_display_name(chat: dict, messages: list[dict]) -> str:
    """
    Determine a human-friendly display name for a chat.

    Priority:
    1. chat["name"] if non-null (explicit group name)
    2. For 1:1 chats: the other party's sender ID (phone/email)
    3. For unnamed groups: "Group: participant1, participant2, ..." (max 4, then +N more)
    """
    if chat.get("name"):
        return chat["name"]

    # Collect unique senders that aren't "me"
    other_senders = list(dict.fromkeys(
        msg["sender"] for msg in messages if not msg["is_from_me"] and msg.get("sender")
    ))

    if len(other_senders) == 1:
        return other_senders[0]  # 1:1 chat

    if len(other_senders) <= 4:
        return "Group: " + ", ".join(other_senders)

    shown = ", ".join(other_senders[:4])
    remaining = len(other_senders) - 4
    return f"Group: {shown} +{remaining} more"
```

### Resuming Thread After Previous Sync

When syncing new messages for a chat that was previously synced, we need the `Message-ID` of the last message from the previous sync to set up `In-Reply-To`. This comes from the local SQLite state DB:

```python
def get_thread_anchor_message_id(chat_id: int, last_synced_rowid: int) -> str:
    """
    Get the Message-ID that new messages should reply-to.
    This is the Message-ID of the last message we synced for this chat.
    """
    return generate_message_id(chat_id, last_synced_rowid)
```

---

## Part B: Email Formatter (`formatter.py`)

### Design Goal

Make synced iMessages look as close to the iMessage experience as possible inside Gmail. Not a raw data dump — a **readable conversation**.

### Email Structure

Each iMessage becomes one email in the Gmail thread. The email is **multipart/alternative** with both plaintext and HTML parts (Gmail displays HTML when available).

### Subject Line

```
First message:  "iMessage: Mom"
Replies:        "Re: iMessage: Mom"

First message:  "iMessage: Family Group"
Replies:        "Re: iMessage: Family Group"
```

### From / To Headers

```
From: "Mom via iMessage" <user@gmail.com>
To: user@gmail.com
```

Key decisions:
- `From` display name encodes who sent the message: `"{sender_name} via iMessage"`
- `From` email address is always the user's own Gmail address (required — we're APPENDing to their own mailbox, not actually sending)
- `To` is always the user's own Gmail address (enforced by config validation)
- For messages from the user themselves: `From: "Me via iMessage" <user@gmail.com>`

### Date Header

```
Date: Sat, 08 Feb 2026 18:45:00 -0800
```

Uses the `created_at` timestamp from `imsg` — the email appears at the correct time in the Gmail conversation, maintaining chronological order.

### HTML Body Template

```html
<div style="font-family: -apple-system, BlinkMacSystemFont, 'SF Pro', 'Helvetica Neue', sans-serif; max-width: 600px;">

  <!-- Sender badge -->
  <div style="display: inline-block; background: {color}; color: white; padding: 4px 12px;
              border-radius: 16px; font-size: 13px; font-weight: 500; margin-bottom: 6px;">
    {sender_display_name}
  </div>

  <!-- Message bubble -->
  <div style="background: {bubble_color}; border-radius: 18px; padding: 10px 16px;
              margin-bottom: 4px; display: inline-block; max-width: 80%;">
    <p style="margin: 0; font-size: 15px; line-height: 1.4; color: {text_color};">
      {message_text}
    </p>
  </div>

  <!-- Timestamp -->
  <div style="font-size: 11px; color: #8e8e93; margin-top: 2px;">
    {formatted_timestamp}
  </div>

  <!-- Attachment notice (if any) -->
  {attachment_section}

  <!-- Reactions (if any) -->
  {reactions_section}

</div>
```

### Color Scheme

Messages from the user (is_from_me=true):
- Bubble: `#007AFF` (iMessage blue)
- Text: `#FFFFFF`
- Sender badge: `#007AFF`

Messages from others:
- Bubble: `#E9E9EB` (iMessage gray)
- Text: `#000000`
- Sender badge: `#8E8E93` (system gray)

For group chats, each unique sender gets a rotating color from a palette to visually distinguish participants:

```python
SENDER_COLORS = [
    "#FF3B30",  # red
    "#FF9500",  # orange
    "#FFCC00",  # yellow
    "#34C759",  # green
    "#5AC8FA",  # teal
    "#AF52DE",  # purple
    "#FF2D55",  # pink
]
```

### Plaintext Body (Fallback)

```
[Mom] Hey are we still on for dinner?
— Feb 8, 2026 6:45 PM

📎 IMG_1234.heic (2.3 MB)
```

Simple, readable, no HTML dependency. Used by email clients that don't render HTML.

### Attachment Handling (v1: Metadata Only)

We don't embed or attach files in v1. Instead, we show attachment metadata:

```html
<div style="background: #f2f2f7; border-radius: 12px; padding: 8px 12px; margin-top: 6px;">
  <span style="font-size: 13px; color: #636366;">
    📎 {transfer_name} ({formatted_size})
  </span>
</div>
```

### Reactions Display

iMessage reactions (tapbacks) shown as inline badges:

```html
<div style="font-size: 12px; color: #636366; margin-top: 4px;">
  ❤️ Liked by Mom · 😂 Laughed by Dad
</div>
```

### Full Message Builder

```python
# src/formatter.py
"""Format iMessage data into human-friendly RFC 2822 emails."""

import email.message
import email.utils
from datetime import datetime
from typing import Optional


def build_email_message(
    sender_display_name: str,
    user_email: str,
    message_text: Optional[str],
    message_timestamp: str,
    is_from_me: bool,
    attachments: list[dict],
    reactions: list[dict],
    thread_headers: dict,
) -> bytes:
    """
    Build a complete RFC 2822 multipart/alternative email.

    Args:
        sender_display_name: Who sent this (e.g., "Mom", "+15551234567")
        user_email: The authenticated Gmail address (used for From/To)
        message_text: iMessage text body (can be None for attachment-only)
        message_timestamp: ISO 8601 timestamp from imsg
        is_from_me: True if the user sent this message
        attachments: List of attachment metadata dicts from imsg
        reactions: List of reaction dicts from imsg
        thread_headers: Dict from threading_engine.build_thread_headers()

    Returns:
        Complete RFC 2822 email as bytes, ready for IMAP APPEND.
    """
    msg = email.message.EmailMessage()

    # Threading headers
    msg["Message-ID"] = thread_headers["Message-ID"]
    msg["Subject"] = thread_headers["Subject"]
    if thread_headers.get("In-Reply-To"):
        msg["In-Reply-To"] = thread_headers["In-Reply-To"]
    if thread_headers.get("References"):
        msg["References"] = thread_headers["References"]

    # Sender/recipient
    display = "Me" if is_from_me else sender_display_name
    msg["From"] = f'"{display} via iMessage" <{user_email}>'
    msg["To"] = user_email

    # Date (use original iMessage timestamp)
    parsed_time = datetime.fromisoformat(message_timestamp)
    msg["Date"] = email.utils.format_datetime(parsed_time)

    # Custom headers for tooling/search
    msg["X-iMessage-Sync"] = "imessage-gmail-sync"
    msg["X-iMessage-Chat-ID"] = str(thread_headers.get("chat_id", ""))

    # Build body content
    plaintext_body = format_plaintext_body(
        display, message_text, message_timestamp, attachments, reactions
    )
    html_body = format_html_body(
        display, message_text, message_timestamp, is_from_me, attachments, reactions
    )

    # Set as multipart/alternative
    msg.make_alternative()
    msg.add_alternative(plaintext_body, subtype="plain")
    msg.add_alternative(html_body, subtype="html")

    return msg.as_bytes()


def format_size_human(total_bytes: int) -> str:
    """Convert bytes to human-readable: 2456789 → '2.3 MB'"""
    ...


def format_timestamp_human(iso_timestamp: str) -> str:
    """Convert ISO 8601 to human-friendly: 'Feb 8, 2026 6:45 PM'"""
    ...
```

### Group Chat: Multi-Sender Formatting

In group chats, the sender badge is critical — it tells you who said what. The `From` header display name changes per message:

```
Message from Mom:    From: "Mom via iMessage" <user@gmail.com>
Message from Dad:    From: "Dad via iMessage" <user@gmail.com>
Message from you:    From: "Me via iMessage" <user@gmail.com>
```

Gmail's conversation view shows this naturally — each message in the thread shows a different "sender" name, just like a real group email conversation.

### Edge Cases

| Case | Handling |
|------|----------|
| Empty text (attachment-only) | Show "📎 [attachment info]" as body |
| Very long message (>5000 chars) | No truncation — email handles long bodies fine |
| URL-only messages | Auto-linkify URLs in HTML body |
| Emoji-only messages | Display as-is (works in both plaintext and HTML) |
| Null sender | Display as "Unknown" |
| Unicode / non-ASCII | RFC 2822 `email` module handles encoding automatically |
| Line breaks in iMessage | Convert `\n` to `<br>` in HTML, preserve in plaintext |
