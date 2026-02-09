# 08 — Complete File Tree, Dependencies & Build Order

## Complete File Tree

```
imessage-gmail-sync/
│
├── install.sh                          # One-liner bootstrap script
│                                       # brew install imsg + venv + deps + CLI wrapper
│
├── requirements.txt                    # Pip dependencies (2 packages)
│
├── config.example.toml                 # Example config file with all options documented
│
├── bin/
│   └── imessage-gmail-sync             # Shell wrapper (installed to ~/.local/bin/)
│                                       # Activates venv, runs `python -m src.main`
│
├── src/
│   ├── __init__.py                     # Package init (version string)
│   │
│   ├── main.py                         # CLI entrypoint
│   │                                   # argparse: setup | sync | status | chats
│   │                                   # Dispatches to appropriate module
│   │                                   # ~80 lines
│   │
│   ├── config.py                       # TOML config loader + validator
│   │                                   # load_config(), validate_config()
│   │                                   # Merges file values with defaults
│   │                                   # ~120 lines
│   │
│   ├── imsg_client.py                  # Subprocess wrapper for `imsg` CLI
│   │                                   # list_chats(), get_chat_history()
│   │                                   # verify_installation()
│   │                                   # JSONL parsing, error handling
│   │                                   # ~150 lines
│   │
│   ├── gmail_imap.py                   # IMAP engine for Gmail
│   │                                   # gmail_connection() context manager
│   │                                   # append_message(), ensure_label_exists()
│   │                                   # find_message_by_message_id()
│   │                                   # get_last_synced_timestamp()
│   │                                   # test_gmail_connection()
│   │                                   # ~200 lines
│   │
│   ├── threading_engine.py             # Email threading header generation
│   │                                   # generate_message_id()
│   │                                   # build_thread_headers()
│   │                                   # resolve_chat_display_name()
│   │                                   # References truncation logic
│   │                                   # ~100 lines
│   │
│   ├── formatter.py                    # iMessage → RFC 2822 email formatting
│   │                                   # build_email_message()
│   │                                   # format_html_body() — iMessage bubble style
│   │                                   # format_plaintext_body() — clean fallback
│   │                                   # Attachment metadata, reactions
│   │                                   # ~250 lines
│   │
│   ├── sync_engine.py                  # Sync orchestrator
│   │                                   # run_sync() — main entrypoint
│   │                                   # sync_single_chat() — per-chat logic
│   │                                   # Two-layer dedup (SQLite + IMAP)
│   │                                   # Rate limiting, retry, progress logging
│   │                                   # ~200 lines
│   │
│   ├── state.py                        # Local SQLite state database
│   │                                   # initialize_state_database()
│   │                                   # get_last_synced_rowid()
│   │                                   # update_sync_state()
│   │                                   # record_sync_run_start/end()
│   │                                   # get_sync_summary()
│   │                                   # ~120 lines
│   │
│   ├── setup_wizard.py                 # Interactive first-run guide
│   │                                   # run_setup() — full wizard flow
│   │                                   # prompt_gmail_credentials()
│   │                                   # check_prerequisites()
│   │                                   # check_macos_permissions()
│   │                                   # create_config_file()
│   │                                   # ~180 lines
│   │
│   └── logging_config.py              # Colorized logging setup
│                                       # ColorizedFormatter
│                                       # configure_logging()
│                                       # ~40 lines
│
├── plan/                               # Architecture plan (this directory)
│   ├── 01-overview.md
│   ├── 02-imsg-integration.md
│   ├── 03-gmail-imap-engine.md
│   ├── 04-threading-and-formatting.md
│   ├── 05-sync-engine.md
│   ├── 06-config-and-cli.md
│   ├── 07-installer.md
│   └── 08-file-tree-and-build-order.md
│
├── LICENSE
└── README.md
```

**Estimated total source**: ~1,440 lines of Python across 10 files. Small, readable, no bloat.

---

## `requirements.txt`

```
imapclient>=5.0,<6.0
tomli-w>=1.0,<2.0
tomli>=2.0,<3.0; python_version < "3.11"
```

That is the entire dependency list. Three entries, one conditional.

| Package | Version | Size | Transitive Deps | Purpose |
|---------|---------|------|-----------------|---------|
| `imapclient` | 5.x | ~500 KB | 0 (stdlib only) | IMAP operations, Gmail labels |
| `tomli-w` | 1.x | ~15 KB | 0 | Write TOML (setup wizard config generation) |
| `tomli` | 2.x | ~30 KB | 0 | Read TOML (backport for Python <3.11 only) |

Total pip install footprint: **<1 MB**. Zero C extensions. Zero transitive dependencies.

---

## Build Order (Implementation Sequence)

Implementation should proceed in this order, where each phase produces a testable artifact:

### Phase 1: Foundation (Files 1-4)

```
1. src/__init__.py              → package declaration
2. src/logging_config.py        → colorized logging (used by everything)
3. src/config.py                → TOML loader (needed for all operations)
4. config.example.toml          → example config for development
```

**Test**: `python -c "from src.config import load_config; print(load_config('/path/to/test.toml'))"`

### Phase 2: External Integrations (Files 5-6)

```
5. src/imsg_client.py           → imsg subprocess wrapper
6. src/gmail_imap.py            → IMAP connection + APPEND
```

**Test imsg**: `python -c "from src.imsg_client import list_chats; print(list_chats(limit=3))"`
**Test IMAP**: `python -c "from src.gmail_imap import test_gmail_connection; print(test_gmail_connection('email', 'pass'))"`

### Phase 3: Core Logic (Files 7-9)

```
7. src/threading_engine.py      → Message-ID generation + header chains
8. src/formatter.py             → Email body formatting (HTML + plaintext)
9. src/state.py                 → SQLite state database
```

**Test threading**: Generate headers for a mock chat, verify Message-ID format and References chain.
**Test formatting**: Build an email from mock data, write to `.eml` file, open in Mail.app to verify rendering.
**Test state**: Create DB, insert/query sync state, verify idempotent updates.

### Phase 4: Orchestration (File 10)

```
10. src/sync_engine.py          → Sync orchestrator (ties everything together)
```

**Test**: `imessage-gmail-sync sync --dry-run --limit 3` — should list what would sync without touching Gmail.

### Phase 5: User Interface (Files 11-13)

```
11. src/main.py                 → CLI entrypoint with argparse
12. src/setup_wizard.py         → Interactive setup
13. bin/imessage-gmail-sync     → Shell wrapper
```

**Test**: `imessage-gmail-sync setup` → full wizard flow
**Test**: `imessage-gmail-sync sync --dry-run` → end-to-end dry run

### Phase 6: Installer (File 14)

```
14. install.sh                  → One-liner installer
```

**Test**: Fresh macOS machine (or VM), run the one-liner, verify everything installs and `imessage-gmail-sync setup` works.

---

## Runtime Paths Summary

| Path | Purpose | Created By |
|------|---------|------------|
| `~/.local/share/imessage-gmail-sync/` | Installation root | install.sh |
| `~/.local/share/imessage-gmail-sync/venv/` | Python virtual environment | install.sh |
| `~/.local/share/imessage-gmail-sync/src/` | Python source code | install.sh |
| `~/.local/share/imessage-gmail-sync/sync_state.db` | SQLite sync state | state.py (first run) |
| `~/.config/imessage-gmail-sync/config.toml` | User configuration | setup_wizard.py |
| `~/.local/bin/imessage-gmail-sync` | CLI wrapper script | install.sh |
| `/opt/homebrew/bin/imsg` | imsg binary | brew install |

---

## Stdlib Modules Used (No Install Required)

| Module | Purpose |
|--------|---------|
| `argparse` | CLI argument parsing |
| `email.message` | RFC 2822 email construction |
| `email.utils` | Email header formatting (dates, addresses) |
| `imaplib` | Low-level IMAP (used by imapclient internally) |
| `json` | Parse imsg JSONL output |
| `subprocess` | Run imsg CLI |
| `sqlite3` | Local state database |
| `tomllib` | Read TOML config (Python 3.11+) |
| `logging` | Structured logging |
| `pathlib` | Path manipulation |
| `datetime` | Timestamp handling |
| `getpass` | Hidden password input in setup wizard |
| `os` | File permissions, environment |
| `sys` | Version checks, exit codes |
| `time` | Rate limiting delays |
| `contextlib` | Context manager for IMAP connection |
| `textwrap` | Help text formatting |
| `html` | HTML escaping for message bodies |

---

## Future Work — Not Planned, Just Noted

These are explicitly **not** part of this implementation. They are recorded here as acknowledged future directions only.

| Feature | Notes |
|---------|-------|
| **Real-time sync** | `imsg watch --json` piped into sync engine. Single CLI command: `imessage-gmail-sync watch`. Would use the existing sync_engine with streaming input. |
| **Brew service** | `brew services start imessage-gmail-sync` for launchd-managed background daemon. Would need a plist and service management. |
| **Attachment embedding** | Embed images inline as `cid:` references or as MIME attachments. Would increase email size and require reading from `~/Library/Messages/Attachments/`. |
| **Two-way sync** | Reply from Gmail → send via iMessage. Would need IMAP IDLE + `imsg send`. Significant complexity. |
| **Contact name resolution** | Map phone numbers to contact names via macOS Contacts framework. Would require additional macOS permissions. |
| **Multiple Gmail accounts** | Support syncing to different Gmail accounts. Config schema would need `[[gmail]]` array. |
