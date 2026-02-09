# 02 — `imsg` CLI Integration Layer

## Purpose

`src/imsg_client.py` wraps the `steipete/imsg` CLI binary (installed via `brew install steipete/tap/imsg`) as a subprocess. It parses JSON output into Python dicts and provides a clean function-based API for the sync engine.

## `imsg` CLI Reference (What We Consume)

### `imsg chats --json`

Returns one JSON object **per line** (JSONL) for each chat:

```json
{
  "id": 42,
  "name": "Family Group",
  "identifier": "chat395427184639285248",
  "service": "iMessage",
  "last_message_at": "2026-02-08T19:30:00Z"
}
```

Key fields:
- `id` — integer, stable chat ID from `chat.db` ROWID. This is our primary key for threading.
- `name` — display name. For 1:1 chats this is the contact name or phone number. For group chats this is the group name (or null if unnamed).
- `identifier` — internal identifier string
- `service` — `"iMessage"` or `"SMS"`
- `last_message_at` — ISO 8601 timestamp of most recent message

### `imsg history --chat-id <id> --json`

Returns one JSON object **per line** for each message in the chat:

```json
{
  "id": 12847,
  "chat_id": 42,
  "guid": "p:0/E3F2A1B4-...",
  "reply_to_guid": null,
  "sender": "+15551234567",
  "is_from_me": false,
  "text": "Hey are we still on for dinner?",
  "created_at": "2026-02-08T18:45:00Z",
  "attachments": [
    {
      "filename": "IMG_1234.heic",
      "transfer_name": "IMG_1234.heic",
      "uti": "public.heic",
      "mime_type": "image/heic",
      "total_bytes": 2456789,
      "is_sticker": false,
      "original_path": "~/Library/Messages/Attachments/...",
      "missing": false
    }
  ],
  "reactions": []
}
```

Key fields:
- `id` — integer ROWID from `message` table. **This is our sync cursor.** Always monotonically increasing.
- `chat_id` — foreign key back to the chat
- `sender` — E.164 phone number or email (for iMessage email handles)
- `is_from_me` — boolean, true if the logged-in user sent this message
- `text` — message body (can be null for attachment-only messages)
- `created_at` — ISO 8601 timestamp
- `attachments` — array (empty if none). We log attachment metadata but **do not sync attachment files** in v1.
- `reactions` — array of tapback reactions (logged, not synced in v1)

### Useful Flags

| Flag | Purpose | Our Usage |
|------|---------|-----------|
| `--limit N` | Cap number of results | Used for initial sync backfill cap |
| `--start ISO8601` | Filter messages after this time | Used for time-bounded lookback |
| `--end ISO8601` | Filter messages before this time | Not used in v1 |
| `--attachments` | Include attachment metadata | Always on |
| `--json` | JSONL output | Always on |
| `--region XX` | Phone number normalization region | Configurable, default `US` |

## `imsg_client.py` — Function API

```python
# src/imsg_client.py
"""Subprocess wrapper around the `imsg` CLI binary."""

import json
import subprocess
import logging
from typing import Optional

logger = logging.getLogger(__name__)

IMSG_BINARY = "imsg"  # assumes on PATH via brew


def get_imsg_path() -> str:
    """Verify imsg is installed and return its absolute path."""
    # subprocess: `which imsg` or `brew --prefix steipete/tap/imsg`
    # Raises RuntimeError if not found with install instructions
    ...


def list_chats(limit: int = 100) -> list[dict]:
    """
    Run `imsg chats --limit {limit} --json` and return parsed list.

    Returns list of dicts with keys: id, name, identifier, service, last_message_at
    Sorted by last_message_at descending (most recent first).
    """
    ...


def get_chat_history(
    chat_id: int,
    limit: Optional[int] = None,
    since_timestamp: Optional[str] = None,
    region: str = "US",
) -> list[dict]:
    """
    Run `imsg history --chat-id {chat_id} --attachments --json` with optional filters.

    Args:
        chat_id: The chat ROWID from list_chats()
        limit: Max messages to fetch (None = all)
        since_timestamp: ISO 8601 lower bound for --start flag
        region: Phone number normalization region

    Returns list of message dicts sorted by id ascending (oldest first).
    Messages are sorted ascending so we can process them in chronological order
    for correct threading (each message references the one before it).
    """
    ...


def verify_installation() -> dict:
    """
    Check that imsg is installed, runnable, and has DB access.

    Returns dict with:
        installed: bool
        version: str or None
        db_accessible: bool (tries `imsg chats --limit 1`)
        error: str or None
    """
    ...
```

## Subprocess Execution Pattern

```python
def _run_imsg(arguments: list[str], timeout_seconds: int = 30) -> list[dict]:
    """
    Execute imsg with given arguments and parse JSONL output.

    - Runs with subprocess.run(capture_output=True, text=True, timeout=...)
    - Checks returncode; logs stderr on failure
    - Parses stdout as JSONL (one JSON object per line)
    - Returns list of parsed dicts
    - Raises ImsgError on non-zero exit or parse failure
    """
    command = [IMSG_BINARY] + arguments
    logger.debug("Running: %s", " ".join(command))

    result = subprocess.run(
        command,
        capture_output=True,
        text=True,
        timeout=timeout_seconds,
    )

    if result.returncode != 0:
        logger.error("imsg failed (exit %d): %s", result.returncode, result.stderr)
        raise ImsgError(f"imsg exited {result.returncode}: {result.stderr.strip()}")

    messages = []
    for line in result.stdout.strip().splitlines():
        line = line.strip()
        if line:
            messages.append(json.loads(line))

    logger.debug("Parsed %d records from imsg", len(messages))
    return messages
```

## Error Handling

| Scenario | Detection | Response |
|----------|-----------|----------|
| `imsg` not installed | `which imsg` fails | Print install instructions: `brew install steipete/tap/imsg` |
| Full Disk Access denied | `imsg chats` returns error about DB access | Print macOS permission instructions |
| Empty chat list | Zero results from `imsg chats` | Log warning; not an error (new Mac or no messages) |
| Malformed JSON | `json.loads()` raises | Log the raw line, skip it, continue |
| Timeout | `subprocess.TimeoutExpired` | Raise with helpful message (large chat history) |
| `imsg` crash/segfault | Non-zero exit + no JSON | Raise `ImsgError` with stderr |

## Custom Exception

```python
class ImsgError(Exception):
    """Raised when the imsg CLI fails or returns unexpected output."""
    pass
```

## Message ID Stability

The `id` field (ROWID) from `imsg history` is a stable, monotonically increasing integer from the SQLite `message` table. This is critical because:

1. We use it as our **sync cursor** — "last synced ROWID for this chat"
2. We use it in our **deterministic Message-ID** generation for email threading: `<imsg-{chat_id}-{message_rowid}@imessage-gmail-sync>`
3. It survives across `imsg` invocations (it's the DB primary key)

## Group Chat Handling

Group chats in `imsg` output:
- `name` field contains the group name (set by the group creator) or `null` for unnamed groups
- `identifier` contains the internal chat identifier
- Messages have `sender` field showing who sent each message (critical for formatting "who said what")
- `is_from_me` flag distinguishes the user's own messages

For unnamed group chats, we generate a display name from participant phone numbers/emails: `"Group: +1555123, +1555456, +1555789"` (truncated if >3 participants).

## Performance Considerations

- `imsg chats` is fast (<1s for hundreds of chats) — reads from SQLite index
- `imsg history` can be slow for chats with 10k+ messages — always use `--limit` or `--start` to bound
- We use `--start` with the timestamp of the last-synced message to avoid re-fetching old messages
- For initial backfill, respect the user's `max_messages_per_chat` config (default 500)
- Subprocess spawning overhead is negligible (~5ms per invocation on macOS)

## Testing During Development

```bash
# Verify imsg works
imsg chats --limit 3 --json

# Get history for a specific chat
imsg history --chat-id 1 --limit 5 --attachments --json

# Test with time filter
imsg history --chat-id 1 --start "2026-02-01T00:00:00Z" --json
```
