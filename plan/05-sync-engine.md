# 05 — Sync Engine & State Management

## Purpose

`src/sync_engine.py` is the orchestrator. It coordinates: fetch chats from `imsg` → determine what's new → format emails → inject into Gmail → update local state. `src/state.py` manages a local SQLite database that tracks sync progress per chat.

---

## Part A: State Management (`state.py`)

### Why Local State

Without local state, every sync would need to either:
1. Re-sync everything (wasteful, duplicates)
2. Search Gmail via IMAP for the latest message per chat (slow, O(n) per chat)

A local SQLite DB gives us O(1) "where did I leave off?" per chat.

### State DB Location

```
~/.local/share/imessage-gmail-sync/sync_state.db
```

Same base directory as the Python venv. Created automatically on first run.

### Schema

```sql
CREATE TABLE IF NOT EXISTS sync_state (
    chat_id            INTEGER PRIMARY KEY,
    chat_display_name  TEXT NOT NULL,
    last_synced_rowid  INTEGER NOT NULL DEFAULT 0,
    last_synced_at     TEXT NOT NULL,  -- ISO 8601
    total_messages_synced INTEGER NOT NULL DEFAULT 0,
    first_synced_at    TEXT NOT NULL   -- ISO 8601
);

CREATE TABLE IF NOT EXISTS sync_runs (
    run_id         INTEGER PRIMARY KEY AUTOINCREMENT,
    started_at     TEXT NOT NULL,
    completed_at   TEXT,
    status         TEXT NOT NULL DEFAULT 'running',  -- running, completed, failed
    chats_synced   INTEGER DEFAULT 0,
    messages_synced INTEGER DEFAULT 0,
    error_message  TEXT
);
```

### State Functions

```python
# src/state.py
"""Local SQLite state tracking for sync progress."""

import sqlite3
import logging
from datetime import datetime, timezone

logger = logging.getLogger(__name__)

STATE_DB_PATH = "~/.local/share/imessage-gmail-sync/sync_state.db"


def initialize_state_database(database_path: str) -> None:
    """Create state DB and tables if they don't exist."""
    ...


def get_last_synced_rowid(database_path: str, chat_id: int) -> int:
    """
    Get the ROWID of the last message we synced for this chat.
    Returns 0 if this chat has never been synced.
    """
    ...


def update_sync_state(
    database_path: str,
    chat_id: int,
    chat_display_name: str,
    last_synced_rowid: int,
    messages_synced_count: int,
) -> None:
    """
    Update (upsert) the sync state for a chat after successful sync.
    Uses INSERT OR REPLACE with updated timestamp and cumulative count.
    """
    ...


def record_sync_run_start(database_path: str) -> int:
    """Record the start of a sync run. Returns the run_id."""
    ...


def record_sync_run_end(
    database_path: str,
    run_id: int,
    status: str,
    chats_synced: int,
    messages_synced: int,
    error_message: str | None = None,
) -> None:
    """Record the completion (or failure) of a sync run."""
    ...


def get_sync_summary(database_path: str) -> dict:
    """
    Return a summary of sync state for display:
    - Total chats synced
    - Total messages synced
    - Last sync timestamp
    - Per-chat breakdown (name, count, last synced)
    """
    ...
```

---

## Part B: Sync Engine (`sync_engine.py`)

### Sync Modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Default (incremental)** | `imessage-gmail-sync sync` | Sync only messages newer than last sync per chat |
| **Full backfill** | `imessage-gmail-sync sync --full` | Re-sync all messages (respects `max_messages_per_chat`) |
| **Limited** | `imessage-gmail-sync sync --limit 50` | Sync at most N messages per chat |
| **Time-bounded** | `imessage-gmail-sync sync --since 2026-02-01` | Only messages after this date |
| **Dry run** | `imessage-gmail-sync sync --dry-run` | Do everything except IMAP APPEND (print what would happen) |
| **Single chat** | `imessage-gmail-sync sync --chat-id 42` | Sync only one specific chat |

### Core Sync Algorithm

```
SYNC(config, cli_args):
    1. Connect to Gmail via IMAP (verify auth)
    2. Ensure iMessage label exists
    3. Fetch chat list from imsg
    4. Apply chat filters (--chat-id, skip chats with no new messages)
    5. Record sync run start

    FOR each chat:
        a. Look up last_synced_rowid from local state DB
        b. If first sync and no local state: check IMAP for last synced timestamp (fallback)
        c. Fetch message history from imsg (with --start filter if resuming)
        d. Filter to only messages with ROWID > last_synced_rowid
        e. If no new messages: skip, log
        f. Resolve chat display name
        g. Build thread: for each message (chronological order):
            i.   Generate threading headers (Message-ID, In-Reply-To, References)
            ii.  Format email body (HTML + plaintext)
            iii. Build complete RFC 2822 message
            iv.  If not --dry-run: IMAP APPEND to label folder
            v.   Log progress
        h. Update local state with new last_synced_rowid
        i. Log chat sync summary

    6. Record sync run completion
    7. Print final summary
```

### Implementation Skeleton

```python
# src/sync_engine.py
"""Orchestrates the iMessage → Gmail sync process."""

import logging
import time
from typing import Optional

logger = logging.getLogger(__name__)


def run_sync(
    config: dict,
    full_sync: bool = False,
    message_limit: Optional[int] = None,
    since_date: Optional[str] = None,
    target_chat_id: Optional[int] = None,
    dry_run: bool = False,
) -> dict:
    """
    Main sync entrypoint.

    Args:
        config: Parsed TOML config dict
        full_sync: If True, ignore local state and re-sync all
        message_limit: Cap messages per chat
        since_date: ISO 8601 lower time bound
        target_chat_id: Sync only this chat (None = all)
        dry_run: If True, do everything except IMAP APPEND

    Returns:
        Summary dict with chats_synced, messages_synced, errors
    """
    ...


def sync_single_chat(
    chat: dict,
    messages: list[dict],
    last_synced_rowid: int,
    gmail_client,
    config: dict,
    dry_run: bool,
) -> dict:
    """
    Sync one chat's new messages to Gmail.

    Returns dict with:
        chat_id: int
        chat_name: str
        messages_synced: int
        new_last_synced_rowid: int
        skipped: int (already synced / dedup)
        errors: list[str]
    """
    ...
```

### Dedup Strategy (Two-Layer)

```
Layer 1: Local SQLite State (Primary, Fast)
──────────────────────────────────────────
    For each chat, we store last_synced_rowid.
    New messages have ROWID > last_synced_rowid.
    O(1) lookup per chat. No network.

Layer 2: IMAP Message-ID Search (Secondary, Slow)
──────────────────────────────────────────────────
    Before APPEND, optionally check if our deterministic
    Message-ID already exists in the label folder.
    SEARCH HEADER Message-ID "<imsg-42-12847@imessage-gmail-sync>"
    Only used when:
      - State DB is missing/corrupt (first run after state loss)
      - --full sync mode (re-checking everything)
      - Paranoid mode (config flag, default OFF for performance)
    O(n) over label folder per query.
```

Why two layers:
- Layer 1 handles 99% of cases instantly
- Layer 2 prevents duplicates if state DB is deleted, machine is reimaged, or sync runs from a new device

### Rate Limiting and Throttling

```python
APPEND_DELAY_SECONDS = 0.1          # 100ms between appends
APPEND_BATCH_SIZE = 50              # log progress every 50 messages
GMAIL_HOURLY_APPEND_LIMIT = 450     # conservative (actual ~500)
RETRY_ATTEMPTS = 3
RETRY_BACKOFF_BASE_SECONDS = 2.0    # 2s, 4s, 8s

def throttled_append(client, label, message_bytes, timestamp, flags):
    """
    APPEND with rate limiting and retry logic.

    - Sleeps APPEND_DELAY_SECONDS between calls
    - On transient failure (connection drop, rate limit):
      retries with exponential backoff
    - On permanent failure (bad message, quota):
      logs error, returns None, continues
    """
    ...
```

### Progress Reporting

During sync, log output looks like:

```
[INFO] Starting sync...
[INFO] Connected to Gmail (user@gmail.com)
[INFO] Found 23 iMessage chats
[INFO] ─── Syncing: Mom (chat 7) ───
[INFO]   Last synced: ROWID 12840 (Feb 7, 2026)
[INFO]   New messages: 12
[INFO]   [1/12] Mom: "Hey are we still on for dinner?" → APPENDED
[INFO]   [2/12] Me: "Yes! 7pm works" → APPENDED
[INFO]   ...
[INFO]   [12/12] Mom: "See you then! 😊" → APPENDED
[INFO]   ✓ Chat 7 complete: 12 messages synced
[INFO] ─── Syncing: Family Group (chat 42) ───
[INFO]   Last synced: never (first sync)
[INFO]   New messages: 156 (capped at 500)
[INFO]   [1/156] Dad: "Happy new year everyone!" → APPENDED
[INFO]   ...
[INFO] ═══ Sync Complete ═══
[INFO]   Chats synced: 23
[INFO]   Messages synced: 847
[INFO]   Duration: 2m 34s
[INFO]   Errors: 0
```

### Dry Run Output

```
[INFO] Starting sync (DRY RUN — no messages will be injected)...
[INFO] ─── Would sync: Mom (chat 7) ───
[INFO]   Would sync 12 new messages
[INFO]   [1/12] Mom: "Hey are we still on..." → Subject: "Re: iMessage: Mom"
[INFO]   [2/12] Me: "Yes! 7pm works" → Subject: "Re: iMessage: Mom"
[INFO] ─── Would sync: Family Group (chat 42) ───
[INFO]   Would sync 156 new messages (first sync)
[INFO] ═══ Dry Run Complete ═══
[INFO]   Would sync 23 chats, 847 messages
```

### Error Recovery

| Failure Point | Recovery |
|---------------|----------|
| IMAP connection drops mid-batch | State was updated per-chat, not per-batch. Re-run resumes from last completed chat. |
| imsg crashes | Sync run recorded as "failed" in state DB. Re-run picks up where it left off. |
| Duplicate detection failure | Deterministic Message-IDs mean Gmail deduplicates naturally (same Message-ID = same message). IMAP APPEND of a duplicate Message-ID may create a duplicate in Gmail, but our Layer 1 dedup prevents this in practice. |
| Partial chat sync (e.g., 50 of 100 messages injected, then crash) | Local state not updated until ALL messages for a chat are injected. On re-run, the entire chat batch is re-attempted. Layer 2 (IMAP search) catches already-injected messages. |
| Config file deleted | Sync fails at startup with clear error. Config must be restored or re-created via setup wizard. |
| State DB corrupted | Fallback to IMAP timestamp detection for last sync point. Layer 2 dedup prevents duplicates. |

### Atomic State Updates

State is updated **per-chat, after all messages for that chat are successfully appended**. This means:

- If sync crashes after completing chat A but before starting chat B, re-run skips A and starts at B
- If sync crashes mid-chat-B (50 of 100 messages appended), re-run redoes all of chat B. The first 50 messages will be checked by Layer 2 dedup (if enabled) or may create duplicates (acceptable — user can clean up; better than losing messages).

For v1, this is the right tradeoff. Per-message state updates would require a transaction log and add significant complexity for minimal gain.

### Chat Filtering

```python
def should_sync_chat(
    chat: dict,
    last_synced_rowid: int,
    config: dict,
    target_chat_id: int | None,
) -> bool:
    """
    Determine if a chat should be included in this sync run.

    Filters:
    - If target_chat_id is set, only sync that chat
    - Skip chats with no messages (last_message_at is None)
    - Skip chats in config.exclude_chats list (by chat_id)
    - Skip SMS chats if config.imessage_only is True
    """
    ...
```
