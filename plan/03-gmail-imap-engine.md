# 03 — Gmail IMAP Engine

## Purpose

`src/gmail_imap.py` manages the IMAP connection to Gmail. It handles authentication, label/folder creation, message injection via APPEND, marking messages as read, archiving (removing from INBOX), and querying for existing messages (dedup).

## Why IMAP APPEND (Not SMTP)

| SMTP (`smtplib`) | IMAP APPEND (`imapclient`) |
|---|---|
| Triggers Gmail's full delivery pipeline | Bypasses delivery pipeline entirely |
| Spam filtering, inbox placement rules fire | No filtering — message placed directly |
| **Push notifications fire on all devices** | **No notifications — completely silent** |
| Message appears as "new" incoming email | Message appears as if it was always there |
| Requires SMTP credentials (port 587) | Uses same IMAP credentials (port 993) |
| Cannot set labels at injection time | Can set Gmail labels via `X-GM-LABELS` |
| Cannot mark as read at injection time | Can APPEND with `\Seen` flag |

SMTP is a non-starter for bulk sync. Even 20 messages would fire 20 notification sounds.

## Connection Management

```python
# src/gmail_imap.py
"""IMAP engine for injecting messages into Gmail without notifications."""

import imaplib
import logging
from contextlib import contextmanager
from imapclient import IMAPClient

logger = logging.getLogger(__name__)

GMAIL_IMAP_HOST = "imap.gmail.com"
GMAIL_IMAP_PORT = 993


@contextmanager
def gmail_connection(email_address: str, app_password: str):
    """
    Context manager for authenticated Gmail IMAP connection.

    Uses IMAPClient with SSL on port 993.
    Yields the connected+authenticated IMAPClient instance.
    Logs out on exit.

    Example:
        with gmail_connection(email, password) as client:
            client.select_folder("INBOX")
            ...
    """
    client = IMAPClient(GMAIL_IMAP_HOST, port=GMAIL_IMAP_PORT, ssl=True)
    try:
        client.login(email_address, app_password)
        logger.debug("IMAP login successful for %s", email_address)
        yield client
    finally:
        try:
            client.logout()
        except Exception:
            pass
```

## Gmail App Password Authentication

Gmail requires an **App Password** when 2FA is enabled (which it should be). This is a 16-character password generated at https://myaccount.google.com/apppasswords.

- IMAP must be enabled in Gmail settings (Settings → Forwarding and POP/IMAP → Enable IMAP)
- The app password is stored in `~/.config/imessage-gmail-sync/config.toml` (file permissions `0600`)

### Connection Test Function

```python
def test_gmail_connection(email_address: str, app_password: str) -> dict:
    """
    Test IMAP connectivity and authentication.

    Returns dict with:
        connected: bool
        authenticated: bool
        imap_enabled: bool
        capabilities: list[str]
        error: str or None
    """
    ...
```

## Label (Folder) Management

Gmail labels are IMAP folders. The default label is `"iMessage"` (configurable in TOML).

```python
def ensure_label_exists(client: IMAPClient, label_name: str) -> bool:
    """
    Create the Gmail label if it doesn't already exist.

    Uses client.list_folders() to check existence.
    Uses client.create_folder(label_name) if missing.
    Returns True if created, False if already existed.
    """
    ...
```

### Gmail Folder Naming

- Gmail labels with `/` become nested: `"iMessage/Archive"` → nested label
- We use a flat label: `"iMessage"` by default
- Special Gmail folders: `[Gmail]/All Mail`, `[Gmail]/Trash`, `[Gmail]/Drafts`, `INBOX`
- We APPEND to `[Gmail]/All Mail` (or the label directly) and then apply the label

## Message Injection: The Core Operation

### APPEND Workflow

```
1. Build RFC 2822 message bytes (see 04-threading-and-formatting.md)
2. APPEND to the iMessage label folder with flags (\Seen)
3. Verify append succeeded (check returned UID)
4. Optionally: remove INBOX label (archive) if message landed in INBOX
```

### Implementation

```python
def append_message(
    client: IMAPClient,
    label_name: str,
    message_bytes: bytes,
    message_timestamp: str,
    flags: tuple = ("\\Seen",),
) -> int:
    """
    Inject a fully-formed RFC 2822 email into Gmail via IMAP APPEND.

    Args:
        client: Authenticated IMAPClient
        label_name: Gmail label/folder to append into (e.g., "iMessage")
        message_bytes: Complete RFC 2822 email as bytes
        message_timestamp: ISO 8601 timestamp for INTERNALDATE
        flags: IMAP flags to set. Default (\Seen) marks as read.

    Returns:
        The UID of the appended message.

    Notes:
        - APPEND to a Gmail label folder places the message in that label
          AND in [Gmail]/All Mail automatically.
        - Setting \Seen flag means the message appears as already-read.
        - The message does NOT appear in INBOX (unless label_name is "INBOX").
        - No push notifications are triggered.
    """
    ...
```

### APPEND Destination Strategy

**Option A: APPEND directly to the label folder** (Recommended)
```python
client.append(label_name, message_bytes, flags, msg_time)
```
- Message gets the label AND goes to All Mail
- Does NOT land in INBOX → no "new mail" badge
- Already marked as `\Seen` → no unread count bump

**Option B: APPEND to INBOX, then label + archive**
```python
# More steps, not recommended
uid = client.append("INBOX", message_bytes, flags, msg_time)
client.select_folder("INBOX")
client.add_gmail_labels([uid], [label_name])
client.remove_gmail_labels([uid], ["\\Inbox"])  # archive
```

We use **Option A** because it's atomic and simpler.

## Dedup: Checking for Already-Synced Messages

Before syncing, we can check if a message was already injected by searching for our deterministic `Message-ID` header.

```python
def find_message_by_message_id(
    client: IMAPClient,
    label_name: str,
    message_id_header: str,
) -> int | None:
    """
    Search for an existing message with the given Message-ID header.

    Uses IMAP SEARCH with HEADER criterion:
        SEARCH HEADER Message-ID "<imsg-42-12847@imessage-gmail-sync>"

    Returns the UID if found, None otherwise.
    This is the secondary dedup gate (primary is local SQLite state).
    """
    client.select_folder(label_name, readonly=True)
    results = client.search(["HEADER", "Message-ID", message_id_header])
    if results:
        logger.debug("Found existing message for %s: UID %d", message_id_header, results[0])
        return results[0]
    return None
```

## Finding Last Synced Timestamp via IMAP

For the "sync from last sync" default mode, if local state is missing we fall back to IMAP:

```python
def get_last_synced_timestamp(
    client: IMAPClient,
    label_name: str,
) -> str | None:
    """
    Find the most recent message in the iMessage label and return its date.

    Selects the label folder, searches for all messages, fetches INTERNALDATE
    of the most recent one.

    Returns ISO 8601 timestamp string, or None if label is empty.
    This is the fallback for when local SQLite state is unavailable.
    """
    client.select_folder(label_name, readonly=True)
    all_uids = client.search(["ALL"])
    if not all_uids:
        return None

    # Fetch INTERNALDATE of the last (most recent) UID
    latest_uid = max(all_uids)
    fetch_data = client.fetch([latest_uid], ["INTERNALDATE"])
    internal_date = fetch_data[latest_uid][b"INTERNALDATE"]
    return internal_date.isoformat()
```

## Batch Operations

For performance, we batch APPEND operations and use a single IMAP session:

```python
def sync_messages_batch(
    client: IMAPClient,
    label_name: str,
    messages: list[tuple[bytes, str]],  # (message_bytes, timestamp) pairs
) -> list[int]:
    """
    Append a batch of messages to Gmail.

    Opens a single IMAP session, appends all messages sequentially,
    returns list of assigned UIDs.

    Rate limiting: Gmail IMAP has undocumented rate limits.
    We insert a small delay (0.1s) between appends to avoid
    triggering "Too many simultaneous connections" errors.
    Gmail allows ~15 IMAP connections and rate-limits APPEND
    to roughly 500/hour for app password auth.
    """
    ...
```

## Gmail IMAP Quirks and Limits

| Quirk | Impact | Mitigation |
|-------|--------|------------|
| Max 15 simultaneous IMAP connections | Single connection is fine for us | Use context manager, always logout |
| APPEND rate limit ~500/hour | Large initial backfills may hit this | Batch with delays; log progress; resume via state |
| `[Gmail]/All Mail` is the canonical store | APPENDing to a label auto-copies to All Mail | No action needed — this is desired behavior |
| Gmail may rewrite `Message-ID` on SMTP delivery | Not applicable — IMAP APPEND preserves headers exactly | Our threading headers survive intact |
| IMAP SEARCH is slow on large folders | Dedup search could be slow | Primary dedup via local SQLite; IMAP is fallback only |
| `X-GM-LABELS` extension is Gmail-specific | Works only with Gmail | We only target Gmail anyway |
| IDLE command for push notifications | Could use for future real-time | Not used in v1 (noted as future work) |

## Error Handling

| Error | Cause | Response |
|-------|-------|----------|
| `AUTHENTICATIONFAILED` | Wrong password or IMAP disabled | Clear error message + setup instructions |
| `NO [ALERT] Application-specific password required` | 2FA on, no app password | Direct to app password generation URL |
| `BYE Connection dropped` | Network issue / timeout | Retry with exponential backoff (3 attempts) |
| `NO [OVERQUOTA]` | Gmail storage full | Error message, abort sync |
| `BAD Could not parse command` | Malformed message bytes | Log the message, skip, continue batch |
| `NO [TRYCREATE]` | Label doesn't exist | Create label, retry append |

## Testing During Development

```python
# Quick connection test
with gmail_connection("user@gmail.com", "abcd efgh ijkl mnop") as client:
    caps = client.capabilities()
    print(f"Connected. Capabilities: {caps}")
    folders = client.list_folders()
    print(f"Folders: {[f[2] for f in folders]}")

# Test append (dry run — append to Drafts, then delete)
msg = b"From: test@test.com\r\nTo: test@test.com\r\nSubject: Test\r\n\r\nBody"
with gmail_connection(email, password) as client:
    uid = client.append("[Gmail]/Drafts", msg, ("\\Seen",))
    print(f"Appended as UID: {uid}")
    # Clean up
    client.select_folder("[Gmail]/Drafts")
    client.delete_messages([uid])
    client.expunge()
```
