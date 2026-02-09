# Gmail Threading Deep Dive — Everything You Need to Know

> Research document for `imessage-gmail-sync`. All findings verified against primary sources
> (Google official docs, RFC specs, Google engineer statements, forensic analysis).
> Last updated: 2026-02-09.

---

## Table of Contents

1. [The 2019 Rule Change That Changed Everything](#1-the-2019-rule-change)
2. [Gmail's Complete Threading Decision Tree](#2-the-complete-decision-tree)
3. [Header-by-Header Breakdown](#3-header-by-header-breakdown)
4. [IMAP APPEND vs SMTP Delivery — Threading Differences](#4-imap-append-vs-smtp)
5. [Gmail's Label System and IMAP Folders](#5-labels-and-imap-folders)
6. [X-GM-THRID and Gmail's Internal Thread ID](#6-x-gm-thrid)
7. [Thread Breaking Conditions](#7-thread-breaking)
8. [The 100-Message Thread Limit](#8-100-message-limit)
9. [Header Size Limits and References Truncation](#9-header-limits)
10. [From Header Behavior (IMAP APPEND vs SMTP)](#10-from-header)
11. [Notification Behavior (IMAP APPEND)](#11-notifications)
12. [Undocumented Quirks and Gotchas](#12-quirks)
13. [Implications for imessage-gmail-sync](#13-implications)
14. [Sources](#14-sources)

---

## 1. The 2019 Rule Change That Changed Everything {#1-the-2019-rule-change}

In **March 2019**, Google fundamentally tightened Gmail's threading algorithm. This is the single most important thing to understand.

### Before 2019

Gmail would thread messages together when **either** of these was true:
- Same subject + same sender/recipient + sent within **one week** of a prior message
- Matching `References` / `In-Reply-To` headers

Subject-only matching was possible. Time proximity was a factor.

### After 2019

Google added the requirement:

> **"An incoming message's `References` header, if present, must reference IDs of previous messages in order to thread."**

This means:
- Two emails with the **same subject** from the **same sender** will **NOT** be threaded unless one explicitly references the other via `References` or `In-Reply-To` headers.
- The one-week time window was effectively eliminated as a standalone criterion.
- Subject matching became **necessary but not sufficient**.

### The Critical "If Present" Clause

The rule says "if present." This means:

| Scenario | Threading Behavior |
|----------|-------------------|
| Message has `References` header with valid IDs | **Threaded** (if subject matches + IDs match existing thread) |
| Message has `References` header with **wrong** IDs | **NOT threaded** (starts new conversation) |
| Message has **no** `References` header at all | Falls back to looser heuristics (subject + sender + timing) |

**For our project:** We ALWAYS set `References` and `In-Reply-To` headers. This means Gmail will use the strict path every time — which is exactly what we want for deterministic threading.

**Source:** [Google Workspace Updates Blog (March 2019)](https://workspaceupdates.googleblog.com/2019/03/threading-changes-in-gmail-conversation-view.html)

---

## 2. Gmail's Complete Threading Decision Tree {#2-the-complete-decision-tree}

When Gmail receives a message (via SMTP or IMAP APPEND), it evaluates threading with this logic:

```
INCOMING MESSAGE
       │
       ▼
┌─────────────────────────────────────────────┐
│ Does Subject match an existing thread?       │
│ (after stripping Re:/Fwd:/R:/FWD: prefixes) │
└─────────────────────────────────────────────┘
       │
    NO ──► NEW THREAD (full stop)
       │
    YES
       │
       ▼
┌─────────────────────────────────────────────┐
│ Is the existing thread already at 100 msgs? │
└─────────────────────────────────────────────┘
       │
    YES ──► NEW THREAD (full stop)
       │
    NO
       │
       ▼
┌─────────────────────────────────────────────┐
│ Does message have a References header?       │
└─────────────────────────────────────────────┘
       │
    YES ──► Does ANY Message-ID in References
    │       match ANY Message-ID in the thread?
    │          │
    │       NO ──► NEW THREAD
    │          │
    │       YES ──► ADD TO THREAD ✓
       │
    NO (no References header at all)
       │
       ▼
┌─────────────────────────────────────────────┐
│ Is the sender already a participant          │
│ in the existing thread?                      │
└─────────────────────────────────────────────┘
       │
    YES ──► ADD TO THREAD ✓ (subject + sender match)
       │
    NO
       │
       ▼
┌─────────────────────────────────────────────┐
│ Does In-Reply-To reference a message         │
│ in the thread?                               │
└─────────────────────────────────────────────┘
       │
    YES ──► ADD TO THREAD ✓ (In-Reply-To override)
       │
    NO ──► NEW THREAD
```

### ALL conditions that must be met for threading (summary)

1. **Subject must match** (after stripping recognized prefixes)
2. **Thread must be under 100 messages**
3. **One of:**
   - `References` header (if present) contains at least one Message-ID from the thread, OR
   - Sender is already a participant in the thread (when no `References` header), OR
   - `In-Reply-To` references a message in the thread (overrides sender requirement)

---

## 3. Header-by-Header Breakdown {#3-header-by-header-breakdown}

### `Message-ID`

**What it is:** A globally unique identifier for this specific email.

**Format (RFC 5322):** `<local-part@domain>`

```
Message-ID: <imsg-42-12847@imessage-gmail-sync>
```

**Gmail's requirements:**
- **Angle brackets are MANDATORY.** Gmail is now actively rejecting messages with malformed Message-IDs. Error: `550-5.7.1 Messages missing a valid Message-ID header are not accepted.`
- Must follow `<local@domain>` format
- Gmail rejects duplicate Message-IDs within the same delivery context (but IMAP APPEND with a duplicate Message-ID will create a second copy — it's a store operation, not delivery)
- If missing, Gmail will generate one (but this breaks our deterministic threading)

**For our project:** We generate deterministic Message-IDs: `<imsg-{chat_id}-{rowid}@imessage-gmail-sync>`. Always with angle brackets. Always unique per message.

### `In-Reply-To`

**What it is:** The Message-ID of the direct parent message being replied to.

**Format:** `<message-id-of-parent@domain>`

```
In-Reply-To: <imsg-42-12846@imessage-gmail-sync>
```

**Gmail's behavior:**
- Used to identify the direct parent in the thread
- **Critical override:** If `In-Reply-To` references a message in an existing thread, Gmail will thread the message even if the **sender was not previously a participant** in that thread
- Should contain exactly ONE Message-ID (the immediate parent)

**For our project:** Every message after the first in a chat sets `In-Reply-To` to the previous message's Message-ID.

### `References`

**What it is:** A space-separated list of Message-IDs tracing the conversation ancestry, oldest first.

**Format:**
```
References: <root-msg-id@domain> <grandparent@domain> <parent@domain>
```

**Gmail's behavior:**
- **Gmail checks ALL entries** — not just the first or last. Any single matching Message-ID in the chain is sufficient to trigger threading (provided subject also matches).
- The first entry should be the thread root (the very first message in the conversation)
- The last entry should be the direct parent (same as `In-Reply-To`)
- Middle entries are the chain of ancestors
- Since 2019: if `References` is present, it **must** contain at least one valid reference to an existing thread message, or threading fails

**Gmail exempts `References` from the standard 32KB per-header limit** (along with `To` and `Cc`). The total header limit is 500KB across all headers combined.

**For our project:** We build the full chain. For long conversations (>25 messages), we keep the first Message-ID (root) + last 20 Message-IDs, dropping the middle. This matches the RFC best practice of "drop the second" entry to compress the chain while preserving root and recent ancestors.

### `Subject`

**What it is:** The email subject line.

**Gmail's prefix stripping rules:**
| Prefix | Recognized | Case-Sensitive |
|--------|-----------|----------------|
| `Re:` | Yes | No (`re:`, `RE:`, `Re:` all work) |
| `Fwd:` | Yes | No |
| `FWD:` | Yes | No |
| `R:` | Yes | No (used in some European locales) |

**What is NOT recognized:**
- `AW:` (German reply prefix) — **may break threading**
- `SV:` (Scandinavian reply prefix) — **may break threading**
- Any arbitrary text like `[EXTERNAL]`, `[SPAM]`, `URGENT:` — **breaks threading**

**Matching rules:**
- After stripping recognized prefixes, the base subjects must match
- Matching appears to be **case-insensitive** (empirically observed, not officially documented)
- Adding ANY text changes the subject: `"test"` → `"test 123"` = thread break

**For our project:** Subject is `"iMessage: {chat_display_name}"` for the first message, `"Re: iMessage: {chat_display_name}"` for all subsequent. Never changes within a chat. Safe.

---

## 4. IMAP APPEND vs SMTP Delivery — Threading Differences {#4-imap-append-vs-smtp}

This is critical for our project since we use IMAP APPEND exclusively.

### What's the Same

Gmail applies the **same threading algorithm** (Subject + References/In-Reply-To + sender) to IMAP APPENDed messages as it does to SMTP-delivered messages. The threading decision is purely header-based and happens regardless of delivery method.

### What's Different

| Aspect | SMTP Delivery | IMAP APPEND |
|--------|--------------|-------------|
| **Spam filtering** | Yes | **No** — bypassed entirely |
| **Classification** | Yes (Primary/Social/Promotions) | **No** — bypassed |
| **Push notifications** | Yes (fires on all devices) | **No** — silent |
| **`\Recent` flag** | Set by server | **Not reliably set** |
| **`From` header rewriting** | Gmail rewrites to match authenticated user | **Preserved as-is** |
| **`Message-ID` rewriting** | Gmail may add/modify | **Preserved as-is** |
| **Thread ID (`X-GM-THRID`)** | Auto-assigned, cannot set | **Auto-assigned, cannot set** |
| **Gmail API equivalent** | `messages.send()` | `messages.insert()` |
| **Rate limits** | Sending limits (500/day consumer) | IMAP limits (~500 APPEND/hour) |

### Key Insight: IMAP APPEND Preserves Headers

When you SMTP-send through Gmail, it rewrites `From` to match the authenticated account and may modify `Message-ID`. When you IMAP APPEND, the message is stored **exactly as provided**, headers intact. This is critical because:

1. Our deterministic `Message-ID` headers survive intact → threading works reliably
2. Our `From` display names (e.g., `"Mom via iMessage"`) survive intact → conversation view shows correct sender per message
3. Our `References` chain survives intact → Gmail can thread correctly

### You Cannot Force a Thread ID via IMAP

Unlike the Gmail REST API's `messages.insert` (which accepts a `threadId` parameter), IMAP APPEND has **no way to explicitly set `X-GM-THRID`**. You rely entirely on Gmail's header-based threading algorithm. This is fine — our headers are deterministic and correct, so Gmail will auto-assign the right thread ID.

**Source:** [Google Developers: Gmail IMAP Extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions), [Google Developers: messages.insert](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages)

---

## 5. Gmail's Label System and IMAP Folders {#5-labels-and-imap-folders}

### Labels Are Not Folders

Gmail stores all messages in a single pool. "Labels" are metadata tags, not containers. What IMAP clients see as "folders" are actually label-based views. A message with labels `Inbox` + `iMessage` appears in both "folders" but is stored only once.

### Special Gmail Folders

| IMAP Folder | Gmail Label | Notes |
|------------|-------------|-------|
| `INBOX` | Inbox | The inbox label; removing it = "archiving" |
| `[Gmail]/All Mail` | All Mail | Every non-Trash/non-Spam message |
| `[Gmail]/Sent Mail` | Sent | Messages you sent |
| `[Gmail]/Drafts` | Drafts | |
| `[Gmail]/Trash` | Trash | |
| `[Gmail]/Spam` | Spam | |
| Any user folder | User label | Created via IMAP `CREATE` or Gmail UI |

### APPEND Destination Strategy

**APPEND to a user label folder (e.g., `"iMessage"`):**
- Message gets that label automatically
- Message also appears in `[Gmail]/All Mail` (always)
- Message does **NOT** get the `Inbox` label → does NOT appear in Inbox
- No notification triggered
- **This is our approach.**

**APPEND to `INBOX`:**
- Message gets the `Inbox` label
- Appears in All Mail
- May trigger unread badge (but not push notification via IMAP APPEND)
- Would need separate STORE to add our label + remove Inbox label
- **Not recommended for us.**

**APPEND to `[Gmail]/All Mail`:**
- Message gets no user-facing label (just exists in All Mail)
- Would need separate STORE with `+X-GM-LABELS` to add our label
- Two operations instead of one
- **More steps, no benefit over direct label append.**

### Applying Labels via IMAP

Two approaches:

1. **IMAP APPEND to the label folder** — message gets that label automatically (one step)
2. **IMAP STORE with X-GM-LABELS** — `STORE <uid> +X-GM-LABELS ("iMessage")` — applies label to existing message

`IMAPClient` supports: `add_gmail_labels(messages, ['iMessage'])`, `remove_gmail_labels(...)`, `set_gmail_labels(...)`.

**For our project:** APPEND directly to the `"iMessage"` label folder. One step. Clean.

### Labels Attach to Messages, Not Threads

This is a common gotcha. When you "label a thread" in Gmail's UI, it applies the label to each individual message in the thread. But new messages added to the thread later do **NOT** automatically inherit the label. Since we APPEND each message individually to the label folder, each message gets the label. No issue for us.

---

## 6. X-GM-THRID and Gmail's Internal Thread ID {#6-x-gm-thrid}

### What It Is

A 64-bit unsigned integer that groups messages into conversations, exactly matching the Gmail web UI. Example: `1278455344230334865`.

### How to Access It

Requires the `X-GM-EXT-1` IMAP capability (Gmail-specific):

```
# FETCH it
a001 FETCH 1:4 (X-GM-THRID)
* 1 FETCH (X-GM-THRID 1266894439832287888)
* 2 FETCH (X-GM-THRID 1266894439832287888)  ← same thread
* 3 FETCH (X-GM-THRID 1266894439832287888)  ← same thread
* 4 FETCH (X-GM-THRID 1267920559strippedid)  ← different thread

# SEARCH by thread ID (find all messages in a thread)
a002 UID SEARCH X-GM-THRID 1266894439832287888
* SEARCH 2 3 4
```

### Key Properties

| Property | Value |
|----------|-------|
| **Writable?** | **NO** — read-only, server-assigned |
| **Settable on APPEND?** | **NO** |
| **Immutable?** | Yes — once assigned, never changes |
| **Thread merging?** | **Never** — Gmail never joins separate threads |
| **Derivation** | Equals the `X-GM-MSGID` of the first message in the thread |
| **Timestamp encoded?** | Yes — drop last 5 hex digits, treat remainder as epoch milliseconds |

### Google Engineer Confirmation (Brandon Long)

> "You should be able to consider X-GM-THRID as immutable."
> "We never join threads."

This means: if our first message for a chat starts a new Gmail thread, all subsequent messages (with correct headers) will join that thread. And that thread ID will never change or merge with another thread.

**For our project:** We can use `X-GM-THRID` as a verification tool. After syncing, FETCH the thread IDs and confirm all messages for one iMessage chat share the same `X-GM-THRID`. This is a post-sync validation step.

**Source:** [IETF imapext mailing list — Brandon Long (Google)](https://mailarchive.ietf.org/arch/msg/imapext/RPT8q63tPY1dWhzbuo-BoEyUh74/), [Metaspike Forensics](https://www.metaspike.com/dates-gmail-message-id-thread-id-timestamps/)

---

## 7. Thread Breaking Conditions {#7-thread-breaking}

Gmail will **break** a thread (start a new conversation) when ANY of these occur:

| Condition | Example | Preventable? |
|-----------|---------|-------------|
| Subject changed | `"iMessage: Mom"` → `"iMessage: Mom - Photos"` | Yes — never change subject |
| 100-message limit | Thread has 100 msgs, 101st arrives | No — Gmail hard limit |
| `References` present but no matching IDs | `References` points to non-existent messages | Yes — always use correct IDs |
| Sender not a participant (no `In-Reply-To`) | New sender, no reply headers | Yes — always set `In-Reply-To` |
| System-altered subject | `[EXTERNAL] iMessage: Mom` | N/A — only SMTP delivery |

### Things That Do NOT Break Threads

| Condition | Still Threads? |
|-----------|---------------|
| Months between messages | **Yes** (no time limit post-2019) |
| Different labels on messages | **Yes** |
| `Re:` / `Fwd:` prefix added | **Yes** |
| Message marked read/unread | **Yes** |
| Message archived | **Yes** |
| Very long `References` chain | **Yes** (Gmail checks all entries) |

**For our project:** Thread breaks can only happen at 100 messages per thread. We handle this by detecting it (tracking message count per thread) and starting a new subject + thread chain when approaching the limit. See [Section 8](#8-100-message-limit).

---

## 8. The 100-Message Thread Limit {#8-100-message-limit}

### The Hard Limit

Gmail enforces a hard maximum of **100 messages per conversation thread.** This is documented by Google:

> "A conversation breaks off into a new conversation when the subject line changes, or the conversation gets to more than 100 emails."

### What Happens at Message 101

When the 101st message arrives with correct threading headers, Gmail **automatically** creates a new, separate thread. The original 100-message thread remains intact. The new message starts a fresh thread.

### Implications for Our Project

Many iMessage chats have hundreds or thousands of messages. A chat with 500 messages would create **5 separate Gmail threads** (100 messages each).

**Handling strategy:**

```
Messages 1-100:     Subject: "iMessage: Mom"
                    References chain: normal
                    → Gmail Thread A

Messages 101-200:   Subject: "iMessage: Mom (cont.)"  [or just reset chain]
                    References chain: starts fresh
                    → Gmail Thread B

Messages 201-300:   Subject: "iMessage: Mom (cont. 2)"
                    ...
```

**Alternative (simpler, recommended):** Don't try to outsmart Gmail. Let the first 100 messages thread naturally. When Gmail splits at 101, the new messages will still have our deterministic `Message-ID`s and will form a new thread. The user sees multiple threads for very long chats — this is acceptable and expected in Gmail.

The subject stays the same (`"Re: iMessage: Mom"`), and when message 101's `References` header points to message 100, Gmail will see that the thread is full and start a new one automatically. **No special handling needed on our end** — just let Gmail do its thing.

---

## 9. Header Size Limits and References Truncation {#9-header-limits}

### Gmail's Header Limits

| Limit | Value | Notes |
|-------|-------|-------|
| Individual header size | **32 KB** | Excludes `To`, `Cc`, `References` |
| `References` header | **Exempt from 32KB** | Has own higher limit (undocumented exact value) |
| Total all headers combined | **500 KB** | Across all headers in the message |
| Line length (RFC 5322) | **998 chars max** per line | Must fold long headers with CRLF + whitespace |
| Line length recommended | **78 chars** per line | Best practice for compatibility |

### References Header Truncation Best Practice

Per RFC 2822 and long-standing Usenet convention:

> **"If there are more than about ten identifiers listed, the writer should eliminate the second one."**

The logic:
1. **First entry** = thread root → **always keep** (Gmail needs this to find the thread anchor)
2. **Last entry** = direct parent → **always keep** (Gmail and other clients use this for In-Reply-To matching)
3. **Middle entries** = grandparents → **can be trimmed** (but any match suffices for Gmail)

**Truncation algorithm (repeat until under limit):**
1. Keep the first Message-ID (root)
2. Drop the second Message-ID (oldest non-root)
3. Keep the rest (recent ancestors + parent)

**For our project:** Our deterministic Message-IDs are ~50 bytes each (`<imsg-42-12847@imessage-gmail-sync>`). At 100 messages per thread (Gmail's limit), the References header would be ~5KB. Well within Gmail's limits. We still truncate at 25 entries for cleanliness (keep first + last 20), but it's not strictly necessary.

---

## 10. From Header Behavior {#10-from-header}

### SMTP: Gmail Rewrites It

When sending via Gmail's SMTP server, Gmail **silently rewrites** the `From` header's email address to match the authenticated user's address. The display name is preserved but the address is changed. This is a standards violation per RFC 5322, but Google does it anyway for anti-spoofing.

### IMAP APPEND: Gmail Preserves It

When using IMAP APPEND, the message is stored **exactly as provided**, including the `From` header. IMAP APPEND is a storage operation, not a delivery operation — it bypasses the SMTP submission pipeline entirely.

**Forensic confirmation:** Metaspike's forensic analysis of Gmail IMAP showed that "the Message-Id header field was preserved" when messages were APPENDed via IMAP, confirming headers are stored as-is.

**For our project:** This is great news. We set:
```
From: "Mom via iMessage" <user@gmail.com>
```

The display name (`"Mom via iMessage"`) and the email address (`user@gmail.com`) are both preserved exactly. In Gmail's conversation view, each message shows the display name — so users see "Mom via iMessage", "Dad via iMessage", "Me via iMessage" per message. This creates a natural conversation appearance.

**Why the address is always `user@gmail.com`:** We're APPENDing to the user's own mailbox. Using a different email address would be misleading (the message wasn't actually sent by that address). Using the user's own address is both honest and functional.

**Source:** [Lee Phillips: Gmail Tampers with Outgoing Email](https://lee-phillips.org/gmailRewriting/), [Metaspike: Forensic Examination](https://www.metaspike.com/forensic-examination-manipulated-email-gmail/)

---

## 11. Notification Behavior {#11-notifications}

### Why IMAP APPEND Doesn't Trigger Notifications

IMAP APPEND bypasses Gmail's entire mail delivery pipeline:

| Pipeline Stage | SMTP Delivery | IMAP APPEND |
|----------------|:---:|:---:|
| Received: header added | Yes | No |
| Spam scanning | Yes | No |
| Tab classification | Yes | No |
| `\Recent` flag set | Yes | **No (unreliable)** |
| Push notification fired | Yes | **No** |
| Unread badge bumped | Yes | **Only if not `\Seen`** |

### The `\Seen` Flag

When we APPEND with `("\\Seen")` flags:
- Message appears as **already read**
- No unread count bump
- No bold/highlighted in Gmail UI
- Combined with appending to label folder (not INBOX), the message is effectively invisible unless the user navigates to the label

### The `\Recent` Flag

Gmail's IMAP doesn't reliably set `\Recent` on APPENDed messages. This is actually beneficial — `\Recent` is what some clients use for "new mail" alerts. Without it, there's zero chance of notification leakage.

### Double Protection

Our messages get two layers of notification suppression:
1. **IMAP APPEND** (not SMTP) → no delivery pipeline → no push notifications
2. **`\Seen` flag + label folder** (not INBOX) → no unread badge, not in inbox

**Source:** [Mozilla Bugzilla #885220](https://bugzilla.mozilla.org/show_bug.cgi?id=885220), [Gmail Community: Notifications for filtered messages](https://support.google.com/mail/thread/205631614)

---

## 12. Undocumented Quirks and Gotchas {#12-quirks}

### Quirk 1: `X-Entity-Ref-ID` (Undocumented Threading Signal)

Gmail uses the `X-Entity-Ref-ID` header as an **undocumented** secondary threading/de-threading signal:
- Setting it to a **unique value per email** (e.g., UUID) prevents threading
- Setting it to the **same value** across emails encourages threading
- Used by transactional email senders to prevent password reset emails from stacking

**For our project:** We do NOT set this header. We rely on standard headers (`References`, `In-Reply-To`, `Subject`). Not setting it means Gmail ignores it — which is what we want.

### Quirk 2: Sender/Receiver Threading Asymmetry

If you send the same message twice with subject `"test"` (no `Re:` prefix):
- **Receiver** sees them threaded
- **Sender** sees them as separate

If subject has `"Re: test"`:
- Both sender and receiver see them threaded

**For our project:** Since we're APPENDing to our own mailbox (we're both "sender" and "receiver"), and all messages after the first use `"Re: "` prefix, this quirk doesn't affect us.

### Quirk 3: Gmail Never Joins Threads

Once two messages are in separate threads, **no subsequent message can merge those threads.** This means:
- If our first sync has a bug that creates two threads for one chat, they can never be merged
- Getting the first message right (the thread anchor) is critical
- A message that accidentally starts a new thread is permanent

**For our project:** Extra validation on the first message per chat. Verify it doesn't match an existing thread with the same subject before creating a new one.

### Quirk 4: Labels Attach to Messages, Not Threads

If you label a thread, only the current messages get the label. New messages added later don't inherit it. Since we APPEND each message individually to the label folder, each message gets the label independently. No issue.

### Quirk 5: IMAP SEARCH by Message-ID is Literal

When searching `HEADER Message-ID <imsg-42-12847@imessage-gmail-sync>`, the angle brackets are part of the search string. If your Message-ID has special characters (like `%`), Gmail's IMAP SEARCH may behave unexpectedly.

**For our project:** Our Message-IDs use only alphanumeric characters, hyphens, and `@`. No special characters. Safe for IMAP SEARCH.

### Quirk 6: X-GM-THRID Timestamp Encoding

You can extract a timestamp from the Thread ID: drop the last 5 hex digits of the hex representation, treat the remainder as epoch milliseconds. This can be useful for debugging.

### Quirk 7: Gmail IMAP Bandwidth Limits

Gmail IMAP has undocumented bandwidth limits for downloads. `Account has exceeded the Gmail bandwidth limit for downloads via IMAP` can occur with heavy usage. This applies to FETCH operations, not APPEND — but worth knowing for our IMAP SEARCH dedup queries.

### Quirk 8: Duplicate Message-IDs on APPEND

If you IMAP APPEND a message with the same `Message-ID` as an existing message, Gmail will store a **second copy** (unlike SMTP delivery, which may deduplicate). This means our idempotent sync needs to check before appending, not rely on Gmail to deduplicate.

---

## 13. Implications for imessage-gmail-sync {#13-implications}

### What Our Threading Strategy Must Do

1. **Set `Message-ID` on every message** — deterministic, RFC-compliant, angle-bracketed
2. **Set `References` on every non-first message** — containing at minimum the root Message-ID and direct parent
3. **Set `In-Reply-To` on every non-first message** — pointing to the direct parent
4. **Keep Subject constant** within a chat — `"iMessage: {name}"` first, `"Re: iMessage: {name}"` after
5. **Never change Subject mid-chat** — any change breaks the thread permanently
6. **Accept the 100-message limit** — let Gmail split naturally; don't try to work around it
7. **Deduplicate before APPEND** — Gmail won't deduplicate for us on IMAP APPEND

### APPEND Configuration

```python
# Optimal flags for our use case
flags = ("\\Seen",)  # Mark as read — no unread badge

# Destination folder
folder = "iMessage"   # Custom label — NOT inbox

# Together: messages appear read, labeled, archived, silent
```

### Dedup Strategy

```
Primary:  Local SQLite — track last synced ROWID per chat (O(1))
Secondary: IMAP SEARCH HEADER Message-ID — check our deterministic ID (O(n) per query)
```

The secondary check is critical because IMAP APPEND creates duplicates if the same Message-ID is appended twice (unlike SMTP).

### Message-ID Format

```
<imsg-{chat_id}-{message_rowid}@imessage-gmail-sync>
```

- Angle brackets: mandatory (Gmail enforces RFC 5322)
- Alphanumeric + hyphens only: safe for IMAP SEARCH
- Deterministic: same input → same ID → idempotent
- Unique: `(chat_id, rowid)` is a primary key in chat.db

### References Truncation

```python
MAX_REFERENCES = 25
KEEP_RECENT = 20

def truncate_references(message_ids: list[str]) -> list[str]:
    if len(message_ids) <= MAX_REFERENCES:
        return message_ids
    # Keep first (root) + last KEEP_RECENT (recent ancestors)
    return [message_ids[0]] + message_ids[-KEEP_RECENT:]
```

This is well within Gmail's limits. At 50 bytes per Message-ID × 25 entries = 1.25KB. Gmail's header limit is 500KB total.

### From Header Pattern

```
# Message from another person
From: "Mom via iMessage" <user@gmail.com>

# Message from the user themselves
From: "Me via iMessage" <user@gmail.com>

# Always the same email address (the authenticated user)
To: user@gmail.com
```

IMAP APPEND preserves these exactly. Gmail's conversation view shows the display name per message.

### Post-Sync Validation (Optional)

After syncing a chat, optionally verify threading:

```python
# Fetch X-GM-THRID for all messages in the label with matching subject
# All messages for one iMessage chat should share the same X-GM-THRID
# (up to the 100-message limit, after which a new thread starts)
```

---

## 14. Sources {#14-sources}

### Google Official Documentation
- [Managing Threads — Gmail API](https://developers.google.com/workspace/gmail/api/guides/threads)
- [Gmail IMAP Extensions](https://developers.google.com/workspace/gmail/imap/imap-extensions)
- [REST Resource: users.messages](https://developers.google.com/workspace/gmail/api/reference/rest/v1/users.messages)
- [Group emails into conversations — Gmail Help](https://support.google.com/mail/answer/5900)
- [Gmail message header limits — Workspace Admin](https://support.google.com/a/answer/14016360)
- [Threading changes in Gmail conversation view (March 2019)](https://workspaceupdates.googleblog.com/2019/03/threading-changes-in-gmail-conversation-view.html)

### Google Engineer Statements
- [Brandon Long on X-GM-THRID immutability (IETF imapext)](https://mailarchive.ietf.org/arch/msg/imapext/RPT8q63tPY1dWhzbuo-BoEyUh74/)
- [Original IETF thread on Gmail extensions](https://mailarchive.ietf.org/arch/msg/imapext/WZhTko_X0nzyn57CJT1GEBQeWxA/)

### RFC Standards
- [RFC 5322 — Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [RFC 5256 — IMAP SORT and THREAD Extensions](https://datatracker.ietf.org/doc/html/rfc5256)
- [RFC 2822 — Internet Message Format (superseded by 5322)](https://datatracker.ietf.org/doc/html/rfc2822)

### Third-Party Analysis
- [cloudHQ: How does Gmail decide to group emails into conversations?](https://support.cloudhq.net/how-does-gmail-decide-to-group-emails-into-conversations/)
- [9to5Google: Gmail Conversation View requires 'definite relationship'](https://9to5google.com/2019/03/29/gmail-conversation-view-thread/)
- [Metaspike: Gmail Message ID and Thread ID Timestamps](https://www.metaspike.com/dates-gmail-message-id-thread-id-timestamps/)
- [Metaspike: Forensic Examination of Manipulated Email in Gmail](https://www.metaspike.com/forensic-examination-manipulated-email-gmail/)
- [Lee Phillips: Gmail Tampers with Outgoing Email](https://lee-phillips.org/gmailRewriting/)
- [Waypoint: Prevent threading with X-Entity-Ref-ID](https://www.usewaypoint.com/blog/prevent-threading-on-gmail-with-the-x-entity-ref-id-email-header)
- [Nylas: How to Make Email Threading Less Chaotic](https://www.nylas.com/blog/how-to-make-email-threading-less-chaotic/)
- [Tom Scott: Gmail labels attach to messages, not threads](https://www.tomscott.com/fix-gmail-labels-threads/)
- [Juliana Fernandez Rueda: Threading Emails — Lessons From a Spike](https://medium.com/@juliana.fernandez.rueda/threading-emails-lessons-from-a-spike-54b50a250322)

### Threading Algorithms and Standards
- [D.J. Bernstein: Threading — Message-ID, References, In-Reply-To](https://cr.yp.to/immhf/thread.html)
- [Jamie Zawinski: Message Threading (JWZ algorithm)](https://www.jwz.org/doc/threading.html)

### Gmail Quirks and Bug Reports
- [Spam Resource: Gmail says yes to angle brackets in Message-ID](https://www.spamresource.com/2025/06/gmail-says-yes-to-angle-brackets-in.html)
- [Gmail strips square brackets from Message-Ids — Google Issue Tracker](https://issuetracker.google.com/issues/183687621)
- [Mozilla Bugzilla: No new mail notification on Gmail IMAP](https://bugzilla.mozilla.org/show_bug.cgi?id=885220)
- [U-M ITS: Threading changes in Gmail](https://its.umich.edu/communication/collaboration/google/update/threading-changes-gmail-conversation-view)
