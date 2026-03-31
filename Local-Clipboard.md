<h1>Local-Clipboard</h1>
<h2>A local clipboard history manager. Every time you copy something, it saves it. Pull anything back up from a simple terminal UI. That's it.</h2>

---

## The problem it solves

You copied something, then copied something else over it. It's gone. clipstack means it's never gone.

---

## What it does

- Watches your clipboard in the background
- Saves every new thing you copy to a local file
- Let's you browse + re-copy old clips from a simple TUI menu
- Runs on any Linux machine with Python

---

## Setup

```bash
# 1. Install the one dependency
pip install pyperclip

# 2. Clone it
git clone https://github.com/YOUR_USERNAME/clipstack.git
cd clipstack

# 3. Start the background watcher (keep this running)
python watcher.py

# 4. Open the picker in any other terminal whenever you need it
python picker.py
```

---

## How to use it

```
# In Terminal 1 — always running in background
python watcher.py

# In Terminal 2 — whenever you want an old clip
python picker.py

# Arrow keys to browse, Enter to copy the selected item back to clipboard
```

---

## Project structure

```
clipstack/
├── watcher.py     # Runs in background, saves new clips
├── picker.py      # TUI to browse and re-copy clips
└── clips.txt      # Your clipboard history (plain text, one clip per block)
```

---

## Code

### `watcher.py`
```python
"""
watcher.py — Clipboard watcher.
Runs in the background and saves every new copied item to clips.txt.
Keep this running in a terminal or set it up as a systemd service.
"""

import time
import pyperclip

CLIPS_FILE = "clips.txt"   # where history is saved
POLL_INTERVAL = 1.0        # check clipboard every 1 second

def load_last_clip():
    """Read the most recently saved clip so we don't duplicate it on startup."""
    try:
        with open(CLIPS_FILE, "r") as f:
            content = f.read()
            blocks = content.strip().split("\n---\n")
            return blocks[-1].strip() if blocks else ""
    except FileNotFoundError:
        return ""  # no history file yet, that's fine

def save_clip(text: str):
    """Append a new clip to the history file, separated by a divider."""
    with open(CLIPS_FILE, "a") as f:
        f.write(text.strip() + "\n---\n")

def run():
    print("[clipstack] Watcher running. Ctrl+C to stop.")
    last = load_last_clip()  # seed with last saved clip to avoid re-saving on startup

    while True:
        try:
            current = pyperclip.paste()  # read current clipboard content

            # Only save if something new was copied
            if current and current.strip() != last:
                save_clip(current)
                print(f"[clipstack] Saved: {current[:60]}...")  # preview first 60 chars
                last = current.strip()

        except Exception as e:
            # clipboard can fail on empty or when no display is available
            print(f"[clipstack] Warning: {e}")

        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    run()
```

---

### `picker.py`
```python
"""
picker.py — Clipboard history picker.
Shows a simple arrow-key TUI to browse saved clips.
Press Enter to copy the selected clip back to your clipboard.
"""

import curses
import pyperclip

CLIPS_FILE = "clips.txt"   # same file the watcher writes to

def load_clips():
    """Load all saved clips from the history file, newest first."""
    try:
        with open(CLIPS_FILE, "r") as f:
            content = f.read()
        # Split on the divider, strip whitespace, drop empty blocks
        blocks = [b.strip() for b in content.strip().split("\n---\n") if b.strip()]
        return list(reversed(blocks))  # newest first
    except FileNotFoundError:
        return []  # no history yet

def run(stdscr):
    """Main TUI loop using curses."""
    curses.curs_set(0)          # hide the cursor
    stdscr.clear()

    clips = load_clips()

    if not clips:
        # Nothing saved yet — show message and exit
        stdscr.addstr(0, 0, "No clips saved yet. Copy something first!")
        stdscr.addstr(1, 0, "Press any key to exit.")
        stdscr.getch()
        return

    selected = 0               # index of currently highlighted clip
    max_rows, max_cols = stdscr.getmaxyx()

    while True:
        stdscr.clear()

        # Header
        stdscr.addstr(0, 0, "📋 clipstack — arrow keys to browse, Enter to copy, q to quit")
        stdscr.addstr(1, 0, "─" * (max_cols - 1))

        # Show clips — as many as fit in the terminal
        visible_rows = max_rows - 3  # leave room for header + footer
        for i, clip in enumerate(clips[:visible_rows]):
            # Truncate long clips to fit in one line
            preview = clip.replace("\n", " ")[:max_cols - 6]
            prefix = "▶ " if i == selected else "  "

            if i == selected:
                # Highlight the selected row
                stdscr.attron(curses.A_REVERSE)
                stdscr.addstr(i + 2, 0, f"{prefix}{preview}")
                stdscr.attroff(curses.A_REVERSE)
            else:
                stdscr.addstr(i + 2, 0, f"{prefix}{preview}")

        # Footer
        stdscr.addstr(max_rows - 1, 0, f"  {len(clips)} clips total")

        key = stdscr.getch()  # wait for keypress

        if key == curses.KEY_UP and selected > 0:
            selected -= 1                       # move selection up
        elif key == curses.KEY_DOWN and selected < len(clips) - 1:
            selected += 1                       # move selection down
        elif key == ord("\n"):                  # Enter key — copy to clipboard
            pyperclip.copy(clips[selected])
            stdscr.addstr(max_rows - 1, 0, "  ✅ Copied! Press any key to exit.")
            stdscr.getch()
            break
        elif key == ord("q"):                   # q to quit without copying
            break

if __name__ == "__main__":
    curses.wrapper(run)  # curses.wrapper handles setup/teardown cleanly
```

---

## Auto-start watcher on boot (optional)

Create `/etc/systemd/user/clipstack.service`:

```ini
[Unit]
Description=clipstack clipboard watcher

[Service]
WorkingDirectory=/path/to/clipstack
ExecStart=/usr/bin/python3 watcher.py
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable clipstack
systemctl --user start clipstack
```

---

## Requirements

- Python 3.10+
- `pip install pyperclip`
- Linux with a display (X11 or Wayland)
- `xclip` or `xsel` installed for pyperclip to work:
  ```bash
  sudo apt install xclip
  ```

---

## .gitignore

```
clips.txt
__pycache__/
*.pyc
```

---

## Status

> ✅ Complete and working
