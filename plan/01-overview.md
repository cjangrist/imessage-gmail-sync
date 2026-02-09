# 01 — Project Overview & Architecture

## What This Is

`imessage-gmail-sync` is a macOS CLI tool that reads iMessage conversations via the open-source [`steipete/imsg`](https://github.com/steipete/imsg) CLI and injects them into a Gmail account as properly-threaded, human-readable email conversations using **IMAP APPEND** (not SMTP — no phone notifications).

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Single one-liner install** | `curl -fsSL <raw-gh-url>/install.sh \| bash` installs brew deps + Python venv + config scaffold |
| **Minimal footprint** | Self-contained Python venv at `~/.local/share/imessage-gmail-sync/venv`; only 2 pip packages (`imapclient`, `tomli-w`) plus stdlib |
| **No SMTP** | All Gmail writes go through IMAP APPEND → no delivery pipeline → no push notifications on phone |
| **Conversation fidelity** | iMessage chats (1:1 and group) map 1:1 to Gmail conversation threads via `Message-ID` / `In-Reply-To` / `References` headers |
| **Idempotent sync** | Tracks last-synced message ROWID per chat; IMAP search as secondary dedup gate |
| **Config-driven** | Single TOML file at `~/.config/imessage-gmail-sync/config.toml` |

## High-Level Data Flow

```
┌──────────────┐      JSON       ┌──────────────────┐     IMAP APPEND    ┌────────────┐
│  imsg CLI    │ ──────────────► │  Python Sync     │ ──────────────────► │  Gmail     │
│  (brew)      │  chat list +    │  Engine          │  RFC 2822 emails   │  (IMAP)    │
│              │  message history │                  │  with threading    │            │
│ reads        │                 │ - format body    │  headers           │ label:     │
│ chat.db      │                 │ - build threads  │  + \Seen flag      │ "iMessage" │
│ (read-only)  │                 │ - dedup via      │  + archive         │            │
│              │                 │   rowid + IMAP   │                    │            │
└──────────────┘                 └──────────────────┘                    └────────────┘
```

## Component Map

```
imessage-gmail-sync/
├── install.sh                  # one-liner bootstrap (brew + venv + config scaffold)
├── bin/
│   └── imessage-gmail-sync     # thin shell wrapper that activates venv + runs main
├── src/
│   ├── __init__.py
│   ├── main.py                 # CLI entrypoint (argparse)
│   ├── config.py               # TOML loader + validation
│   ├── imsg_client.py          # subprocess wrapper around `imsg` CLI
│   ├── gmail_imap.py           # IMAP connection, APPEND, label, archive, mark-read
│   ├── threading_engine.py     # Message-ID generation, In-Reply-To/References chain
│   ├── formatter.py            # iMessage → email body (HTML + plaintext multipart)
│   ├── sync_engine.py          # orchestrator: fetch messages → format → inject → update state
│   ├── state.py                # SQLite state DB for last-synced rowid per chat
│   ├── setup_wizard.py         # interactive first-run CLI guide
│   └── logging_config.py       # colorized logging setup
├── requirements.txt            # imapclient, tomli-w (and tomli for <3.11)
├── config.example.toml         # example config
└── plan/                       # this architecture plan
```

## Runtime Dependencies

| Dependency | Source | Purpose |
|-----------|--------|---------|
| `imsg` | `brew install steipete/tap/imsg` | Read iMessage chats + history from `chat.db` |
| Python 3.11+ | System or `brew install python@3.12` | Runtime (stdlib `tomllib`, `imaplib`, `email`, `sqlite3`, `json`, `subprocess`) |
| `imapclient` | pip (venv) | Higher-level IMAP: APPEND, Gmail labels (`X-GM-LABELS`), search, folder ops |
| `tomli-w` | pip (venv) | Write TOML for config generation (stdlib `tomllib` handles reads) |

**That's it.** Two pip packages. Everything else is Python stdlib.

## Key Technical Decisions

### Why IMAP APPEND Instead of SMTP

SMTP delivery triggers Gmail's full mail processing pipeline: spam filters, inbox placement, and — critically — **push notifications on every synced device**. Syncing 500 iMessages via SMTP would blast the user's phone with 500 notification sounds.

IMAP APPEND bypasses all of that. It places a fully-formed RFC 2822 message directly into the mailbox folder. Gmail treats it as if it was always there. Combined with the `\Seen` flag, the messages appear as read, archived, and labeled — silently.

### Why `imsg` CLI via Subprocess (Not Direct DB Access)

1. `imsg` already handles the gnarly SQLite schema of `chat.db` (joins across `message`, `chat`, `chat_message_join`, `handle`, `chat_handle_join`, `attachment`, `message_attachment_join`)
2. It normalizes phone numbers to E.164
3. It provides stable JSON output with `--json`
4. It handles macOS permissions (Full Disk Access) and gives clear errors
5. We avoid duplicating 2000+ lines of Swift parsing logic in Python
6. Future `imsg watch` integration for real-time mode is trivial

### Why Local SQLite State (Not Just IMAP Search)

IMAP search for dedup is O(n) over the label and subject-match sensitive. A local SQLite DB mapping `(chat_id, last_synced_rowid)` makes re-sync O(1) per chat. IMAP search is the **secondary** dedup gate for crash recovery / state loss.

### Why `tomllib` + `tomli-w` (Not `pyyaml` or `configparser`)

- `tomllib` is stdlib since 3.11 — zero install for reads
- TOML is the modern Python config standard (`pyproject.toml`)
- `tomli-w` is tiny (single file) and only needed during setup wizard to write initial config
- Avoids pulling in `pyyaml` (C extension, larger surface area)

## Gmail Conversation Threading Strategy

Gmail threads messages when **all** of these are true:
1. Subject line matches (with `Re:` prefix allowed)
2. `In-Reply-To` header references a `Message-ID` already in the thread
3. `References` header contains `Message-ID`s from the thread chain
4. Messages are within ~1 week of each other (for auto-threading; IMAP APPEND with explicit headers bypasses this)

Our approach:
- Generate a deterministic `Message-ID` per iMessage: `<imsg-{chat_id}-{rowid}@imessage-gmail-sync>`
- First message in a chat thread gets no `In-Reply-To` (starts a new Gmail conversation)
- Subsequent messages set `In-Reply-To` to the previous message's `Message-ID` and `References` accumulates the full chain
- Subject is `"iMessage: {chat_display_name}"` for the first message, `"Re: iMessage: {chat_display_name}"` for all subsequent

This guarantees every iMessage chat (1:1 or group) becomes exactly one Gmail conversation thread.

## Security Model

- Gmail app password stored in TOML config file (user-only permissions `600`)
- `to` address is **always** the authenticated IMAP login address — config validation enforces this
- No outbound SMTP — messages never leave the user's own mailbox
- `imsg` reads `chat.db` in read-only mode — zero risk to Messages.app
- Config directory: `~/.config/imessage-gmail-sync/` with `700` permissions

## Future Work (Noted, Not Planned)

- **Real-time sync**: `imsg watch --json` piped into sync engine (single CLI command)
- **Brew service**: `brew services start imessage-gmail-sync` for background daemon
- **Attachment sync**: embed images inline or as email attachments (currently text-only)
- **Two-way sync**: reply from Gmail → send via iMessage (requires `imsg send`)
