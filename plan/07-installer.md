# 07 — One-Liner Installer

## Purpose

A single `curl | bash` command installs everything: brew dependencies, Python venv, the CLI wrapper, and prints setup instructions.

## The One-Liner

```bash
curl -fsSL https://raw.githubusercontent.com/cjangrist/imessage-gmail-sync/main/install.sh | bash
```

Or with explicit shell:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/cjangrist/imessage-gmail-sync/main/install.sh)"
```

## What the Installer Does

```
install.sh
├── 1. Preflight checks
│   ├── macOS only (uname check)
│   ├── brew installed? if not → install brew
│   └── python3 available? (brew or system)
│
├── 2. Install imsg via brew
│   ├── brew tap steipete/tap
│   └── brew install steipete/tap/imsg
│
├── 3. Create project directory
│   └── ~/.local/share/imessage-gmail-sync/
│
├── 4. Clone or download source
│   ├── git clone (if git available) OR
│   └── curl tarball + extract to project dir
│
├── 5. Create Python venv (self-contained)
│   ├── python3 -m venv ~/.local/share/imessage-gmail-sync/venv
│   ├── venv/bin/pip install --upgrade pip (quiet)
│   └── venv/bin/pip install -r requirements.txt (quiet)
│       (only: imapclient, tomli-w)
│
├── 6. Install CLI wrapper to PATH
│   ├── Create bin/imessage-gmail-sync shell script
│   └── Symlink to ~/.local/bin/imessage-gmail-sync
│   └── Ensure ~/.local/bin is on PATH
│
├── 7. Create config directory
│   ├── mkdir -p ~/.config/imessage-gmail-sync (mode 700)
│   └── Copy config.example.toml if no config exists
│
└── 8. Print next steps
    └── "Run: imessage-gmail-sync setup"
```

## Full Installer Script

```bash
#!/usr/bin/env bash
# install.sh — One-liner installer for imessage-gmail-sync
# Usage: curl -fsSL <url>/install.sh | bash
set -euo pipefail

# ─── Constants ────────────────────────────────────────────────────────
REPO_URL="https://github.com/cjangrist/imessage-gmail-sync"
INSTALL_DIR="$HOME/.local/share/imessage-gmail-sync"
VENV_DIR="$INSTALL_DIR/venv"
CONFIG_DIR="$HOME/.config/imessage-gmail-sync"
BIN_DIR="$HOME/.local/bin"
BIN_NAME="imessage-gmail-sync"

GREEN='\033[0;32m'
YELLOW='\033[0;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

info()  { echo -e "${GREEN}[✓]${NC} $1"; }
warn()  { echo -e "${YELLOW}[!]${NC} $1"; }
error() { echo -e "${RED}[✗]${NC} $1"; exit 1; }

# ─── Preflight ────────────────────────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════════╗"
echo "║   iMessage → Gmail Sync — Installer          ║"
echo "╚══════════════════════════════════════════════╝"
echo ""

# macOS only
[[ "$(uname -s)" == "Darwin" ]] || error "This tool only runs on macOS."

# Homebrew
if ! command -v brew &>/dev/null; then
    warn "Homebrew not found. Installing..."
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    # Add brew to PATH for this session
    eval "$(/opt/homebrew/bin/brew shellenv 2>/dev/null || /usr/local/bin/brew shellenv 2>/dev/null)"
fi
info "Homebrew found: $(brew --version | head -1)"

# Python 3
if ! command -v python3 &>/dev/null; then
    warn "Python 3 not found. Installing via brew..."
    brew install python@3.12
fi
PYTHON="$(command -v python3)"
PY_VERSION="$($PYTHON --version 2>&1)"
info "Python found: $PY_VERSION"

# Check Python version >= 3.10
PY_MINOR=$($PYTHON -c "import sys; print(sys.version_info.minor)")
PY_MAJOR=$($PYTHON -c "import sys; print(sys.version_info.major)")
if [[ "$PY_MAJOR" -lt 3 ]] || [[ "$PY_MINOR" -lt 10 ]]; then
    error "Python 3.10+ required (found $PY_VERSION). Run: brew install python@3.12"
fi

# ─── Install imsg ─────────────────────────────────────────────────────
if ! command -v imsg &>/dev/null; then
    info "Installing imsg..."
    brew tap steipete/tap 2>/dev/null || true
    brew install steipete/tap/imsg
else
    info "imsg already installed: $(imsg --version 2>&1 || echo 'unknown version')"
fi

# ─── Create project directory ─────────────────────────────────────────
mkdir -p "$INSTALL_DIR"
info "Project directory: $INSTALL_DIR"

# ─── Download source ──────────────────────────────────────────────────
if command -v git &>/dev/null; then
    if [[ -d "$INSTALL_DIR/src" ]]; then
        info "Source already exists, pulling latest..."
        git -C "$INSTALL_DIR" pull --ff-only 2>/dev/null || true
    else
        info "Cloning repository..."
        git clone --depth 1 "$REPO_URL.git" "$INSTALL_DIR/repo"
        cp -r "$INSTALL_DIR/repo/src" "$INSTALL_DIR/src"
        cp "$INSTALL_DIR/repo/requirements.txt" "$INSTALL_DIR/requirements.txt"
        cp "$INSTALL_DIR/repo/config.example.toml" "$INSTALL_DIR/config.example.toml"
    fi
else
    info "Downloading source..."
    curl -fsSL "$REPO_URL/archive/refs/heads/main.tar.gz" | tar -xz -C "$INSTALL_DIR" --strip-components=1
fi

# ─── Create Python venv ──────────────────────────────────────────────
if [[ ! -d "$VENV_DIR" ]]; then
    info "Creating Python virtual environment..."
    $PYTHON -m venv "$VENV_DIR"
fi

info "Installing Python dependencies..."
"$VENV_DIR/bin/pip" install --upgrade pip --quiet 2>/dev/null
"$VENV_DIR/bin/pip" install -r "$INSTALL_DIR/requirements.txt" --quiet

INSTALLED_PACKAGES=$("$VENV_DIR/bin/pip" list --format=freeze 2>/dev/null | wc -l | tr -d ' ')
info "Python venv ready ($INSTALLED_PACKAGES packages in $VENV_DIR)"

# ─── Install CLI wrapper ─────────────────────────────────────────────
mkdir -p "$BIN_DIR"

cat > "$BIN_DIR/$BIN_NAME" << 'WRAPPER'
#!/usr/bin/env bash
# imessage-gmail-sync — CLI wrapper that activates the venv and runs main
set -euo pipefail
INSTALL_DIR="$HOME/.local/share/imessage-gmail-sync"
VENV_PYTHON="$INSTALL_DIR/venv/bin/python"

if [[ ! -x "$VENV_PYTHON" ]]; then
    echo "Error: venv not found at $INSTALL_DIR/venv"
    echo "Re-run the installer: curl -fsSL <url>/install.sh | bash"
    exit 1
fi

exec "$VENV_PYTHON" -m src.main "$@"
WRAPPER

chmod +x "$BIN_DIR/$BIN_NAME"
info "CLI installed: $BIN_DIR/$BIN_NAME"

# ─── Ensure ~/.local/bin is on PATH ──────────────────────────────────
if [[ ":$PATH:" != *":$BIN_DIR:"* ]]; then
    # Detect shell config file
    SHELL_NAME="$(basename "$SHELL")"
    if [[ "$SHELL_NAME" == "zsh" ]]; then
        SHELL_RC="$HOME/.zshrc"
    elif [[ "$SHELL_NAME" == "bash" ]]; then
        SHELL_RC="$HOME/.bash_profile"
    else
        SHELL_RC="$HOME/.profile"
    fi

    echo "" >> "$SHELL_RC"
    echo '# Added by imessage-gmail-sync installer' >> "$SHELL_RC"
    echo "export PATH=\"\$HOME/.local/bin:\$PATH\"" >> "$SHELL_RC"
    warn "Added $BIN_DIR to PATH in $SHELL_RC"
    warn "Run: source $SHELL_RC  (or restart your terminal)"
    export PATH="$BIN_DIR:$PATH"
fi

# ─── Create config directory ─────────────────────────────────────────
mkdir -p "$CONFIG_DIR"
chmod 700 "$CONFIG_DIR"

if [[ ! -f "$CONFIG_DIR/config.toml" ]]; then
    if [[ -f "$INSTALL_DIR/config.example.toml" ]]; then
        cp "$INSTALL_DIR/config.example.toml" "$CONFIG_DIR/config.toml.example"
        info "Example config copied to $CONFIG_DIR/config.toml.example"
    fi
fi

# ─── Done ─────────────────────────────────────────────────────────────
echo ""
echo "╔══════════════════════════════════════════════╗"
echo "║   Installation complete!                      ║"
echo "╚══════════════════════════════════════════════╝"
echo ""
echo "  Next step — run the setup wizard:"
echo ""
echo "    imessage-gmail-sync setup"
echo ""
echo "  This will:"
echo "    • Check macOS permissions (Full Disk Access)"
echo "    • Configure your Gmail credentials"
echo "    • Test the connection"
echo "    • Show a preview of what will sync"
echo ""
```

## CLI Wrapper Script Details

The wrapper script at `~/.local/bin/imessage-gmail-sync`:
1. Is a thin bash script (not Python)
2. Activates the venv by calling the venv's Python directly
3. Runs `python -m src.main` with all passed arguments
4. Does not modify `$PATH` or any env vars beyond what the venv Python provides

This means:
- No `source venv/bin/activate` needed
- No conda/pyenv conflicts
- The venv is truly self-contained
- System Python is never touched

## Uninstall

No formal uninstaller needed. Manual cleanup:

```bash
# Remove installation
mv ~/.local/share/imessage-gmail-sync ~/trash/imessage-gmail-sync-$(date +%Y%m%d)

# Remove config (optional)
mv ~/.config/imessage-gmail-sync ~/trash/imessage-gmail-sync-config-$(date +%Y%m%d)

# Remove CLI wrapper
mv ~/.local/bin/imessage-gmail-sync ~/trash/

# Remove brew packages (optional)
brew uninstall steipete/tap/imsg
```

## Idempotent Re-Installs

The installer is safe to re-run:
- `brew install` is a no-op if already installed
- `git pull` updates source in-place
- `pip install -r requirements.txt` is a no-op if deps unchanged
- Config directory creation is `mkdir -p` (safe if exists)
- PATH addition checks before appending
- Existing config files are never overwritten

## Dependency Audit

What the installer puts on disk:

| Component | Location | Size (approx) |
|-----------|----------|---------------|
| `imsg` binary | `/opt/homebrew/bin/imsg` | ~5 MB (Swift binary) |
| Python venv | `~/.local/share/imessage-gmail-sync/venv/` | ~25 MB |
| `imapclient` | (in venv) | ~500 KB |
| `tomli-w` | (in venv) | ~15 KB |
| Source code | `~/.local/share/imessage-gmail-sync/src/` | ~50 KB |
| CLI wrapper | `~/.local/bin/imessage-gmail-sync` | ~300 B |
| Config | `~/.config/imessage-gmail-sync/` | ~1 KB |
| State DB | `~/.local/share/imessage-gmail-sync/sync_state.db` | ~50 KB (grows) |
| **Total** | | **~30 MB** |

That's it. No system-wide pip installs. No conda environments. No Docker. No node_modules.
