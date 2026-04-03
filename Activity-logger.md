#  🌊
> A self-hosted passive activity logger. Answers "what did I actually do today?" — without sending your data anywhere.

---

## What it does

- 🕐 Passively logs active window titles every N seconds
- 📝 Drop manual notes via CLI
- 📊 Generates a dark-mode HTML timeline per day
- 🔍 Full-text search across all activity + notes

---

## Project Structure

```
Activity-Logger/
├── Activity-Logger.py    # Main CLI — notes, reports, search
├── tracker.py     # Background daemon — window watcher
├── db.py          # SQLite read/write helpers
├── report.py      # HTML timeline generator
└── Activity-Logger.db    # Auto-created on first run (gitignore this)
```

---

## Setup Guide

### 1. Prerequisites

```bash
# Python 3.10+
python3 --version

# xdotool — for reading active window on Linux
sudo apt install xdotool libnotify-bin
```

### 2. Clone & enter project

```bash
git clone https://github.com/YOUR_USERNAME/Activity-Logger.git
cd Activity-Logger
# No pip installs needed — stdlib only (sqlite3, subprocess, argparse)
```

### 3. Run any command — DB auto-creates on first use

```bash
python Activity-Logger.py
```

### 4. Start the background tracker (separate terminal)

```bash
python tracker.py --interval 30
```

### 5. Drop notes anytime

```bash
python Activity-Logger.py note "fixed the navbar bug"
python Activity-Logger.py note "took a break, back at 3pm"
```

### 6. Generate today's report

```bash
python Activity-Logger.py report today
# Opens HTML timeline in browser automatically

python Activity-Logger.py report 2025-03-31
```

### 7. Search history

```bash
python Activity-Logger.py search "flask"
python Activity-Logger.py search "youtube"
```

---

## Auto-start tracker on boot (Linux systemd)

Create `/etc/systemd/user/Activity-Logger.service`:

```ini
[Unit]
Description=Activity-Logger window tracker

[Service]
WorkingDirectory=/path/to/Activity-Logger
ExecStart=/usr/bin/python3 tracker.py --interval 30
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable Activity-Logger
systemctl --user start Activity-Logger
```

---

## .gitignore

```
Activity-Logger.db
report_*.html
__pycache__/
*.pyc
venv/
```

---

## Roadmap

- [ ] Weekly summary report
- [ ] Bash/zsh history ingestion
- [ ] macOS support via AppKit
- [ ] Tag system for notes
- [ ] Export to CSV

---

---

# Code

---

## `db.py`

```python
"""
db.py — SQLite read/write helpers for Activity-Logger.
All activity events and notes are stored in a single local DB file.
"""

import sqlite3
import os
from datetime import datetime

# Default DB path — lives next to this script
DB_PATH = os.path.join(os.path.dirname(__file__), "Activity-Logger.db")


def get_conn():
    """Return a connection to the SQLite database."""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row  # rows behave like dicts
    return conn


def init_db():
    """Create tables if they don't exist yet. Safe to call on every startup."""
    conn = get_conn()
    cur = conn.cursor()

    # activity: passive window tracking events
    cur.execute("""
        CREATE TABLE IF NOT EXISTS activity (
            id        INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            app       TEXT,
            title     TEXT
        )
    """)

    # notes: manual notes dropped via CLI
    cur.execute("""
        CREATE TABLE IF NOT EXISTS notes (
            id        INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT NOT NULL,
            content   TEXT NOT NULL
        )
    """)

    conn.commit()
    conn.close()


def insert_activity(app: str, title: str):
    """Log a window activity event with current timestamp."""
    conn = get_conn()
    conn.execute(
        "INSERT INTO activity (timestamp, app, title) VALUES (?, ?, ?)",
        (datetime.now().isoformat(), app, title)
    )
    conn.commit()
    conn.close()


def insert_note(content: str):
    """Save a manual note with current timestamp."""
    conn = get_conn()
    conn.execute(
        "INSERT INTO notes (timestamp, content) VALUES (?, ?)",
        (datetime.now().isoformat(), content)
    )
    conn.commit()
    conn.close()


def get_events_for_day(date_str: str):
    """
    Fetch all activity + notes for a given date.
    date_str format: 'YYYY-MM-DD'
    Returns a merged list sorted by timestamp.
    """
    conn = get_conn()
    cur = conn.cursor()

    cur.execute("""
        SELECT timestamp, 'activity' as type, app, title, NULL as content
        FROM activity WHERE date(timestamp) = ?
    """, (date_str,))
    activity_rows = [dict(r) for r in cur.fetchall()]

    cur.execute("""
        SELECT timestamp, 'note' as type, NULL as app, NULL as title, content
        FROM notes WHERE date(timestamp) = ?
    """, (date_str,))
    note_rows = [dict(r) for r in cur.fetchall()]

    conn.close()

    # Merge and sort chronologically
    all_events = activity_rows + note_rows
    all_events.sort(key=lambda x: x["timestamp"])
    return all_events


def search_events(query: str):
    """Full-text search across window titles and notes. Returns newest first."""
    conn = get_conn()
    cur = conn.cursor()
    like = f"%{query}%"

    cur.execute("""
        SELECT timestamp, 'activity' as type, app, title, NULL as content
        FROM activity WHERE title LIKE ? OR app LIKE ?
        ORDER BY timestamp DESC
    """, (like, like))
    activity_rows = [dict(r) for r in cur.fetchall()]

    cur.execute("""
        SELECT timestamp, 'note' as type, NULL as app, NULL as title, content
        FROM notes WHERE content LIKE ?
        ORDER BY timestamp DESC
    """, (like,))
    note_rows = [dict(r) for r in cur.fetchall()]

    conn.close()
    return activity_rows + note_rows
```

---

## `tracker.py`

```python
"""
tracker.py — Background daemon that logs active window title every N seconds.
Run this in background: python tracker.py --interval 30
Works on Linux via xdotool.
"""

import time
import argparse
import subprocess
import sys
from db import init_db, insert_activity


def get_active_window_linux():
    """
    Use xdotool to get the focused window title + app name.
    Returns (app, title) tuple, or (None, None) on failure.
    Requires: sudo apt install xdotool
    """
    try:
        # Get active window ID
        win_id = subprocess.check_output(
            ["xdotool", "getactivewindow"], stderr=subprocess.DEVNULL
        ).decode().strip()

        # Get window title
        title = subprocess.check_output(
            ["xdotool", "getwindowname", win_id], stderr=subprocess.DEVNULL
        ).decode().strip()

        # Get process name via PID
        pid = subprocess.check_output(
            ["xdotool", "getwindowpid", win_id], stderr=subprocess.DEVNULL
        ).decode().strip()

        app = subprocess.check_output(
            ["ps", "-p", pid, "-o", "comm="], stderr=subprocess.DEVNULL
        ).decode().strip()

        return app, title

    except Exception:
        # Window may have closed between calls — skip this tick
        return None, None


def get_active_window():
    """Cross-platform dispatcher. Extend for Windows/macOS as needed."""
    if sys.platform.startswith("linux"):
        return get_active_window_linux()
    else:
        print("[tracker] Only Linux supported via xdotool for now.")
        return None, None


def run(interval: int):
    """
    Main tracking loop.
    Polls active window every `interval` seconds.
    Skips duplicate consecutive entries to avoid log spam.
    """
    init_db()  # ensure tables exist before writing
    print(f"[tracker] Started. Polling every {interval}s. Ctrl+C to stop.")

    last_title = None  # track previous title to deduplicate

    while True:
        app, title = get_active_window()

        # Only log if we got data and it changed
        if title and title != last_title:
            insert_activity(app or "unknown", title)
            print(f"[tracker] {app} — {title}")
            last_title = title

        time.sleep(interval)


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Activity-Logger background window tracker")
    parser.add_argument("--interval", type=int, default=30,
                        help="Polling interval in seconds (default: 30)")
    args = parser.parse_args()
    run(args.interval)
```

---

## `report.py`

```python
"""
report.py — Generates a dark-mode HTML daily timeline from the database.
Usage:
  python report.py today
  python report.py 2025-03-31
Opens the report in your default browser automatically.
"""

import os
import sys
import webbrowser
from datetime import date, datetime
from db import get_events_for_day


def format_time(iso_str: str) -> str:
    """Convert ISO timestamp to HH:MM format."""
    return datetime.fromisoformat(iso_str).strftime("%H:%M")


def generate_html(events: list, date_str: str) -> str:
    """Build a self-contained HTML timeline page. Works fully offline."""

    rows_html = ""
    if not events:
        rows_html = "<p class='empty'>No activity recorded for this day.</p>"
    else:
        for e in events:
            time_label = format_time(e["timestamp"])
            if e["type"] == "note":
                # Notes get a green pill badge
                rows_html += f"""
                <div class="event note">
                    <span class="time">{time_label}</span>
                    <span class="badge note-badge">NOTE</span>
                    <span class="content">📝 {e['content']}</span>
                </div>"""
            else:
                # Activity events show app name + window title
                rows_html += f"""
                <div class="event activity">
                    <span class="time">{time_label}</span>
                    <span class="badge app-badge">{e['app']}</span>
                    <span class="content">{e['title']}</span>
                </div>"""

    return f"""<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Activity-Logger — {date_str}</title>
  <style>
    * {{ box-sizing: border-box; margin: 0; padding: 0; }}
    body {{ background:#0d0f14; color:#c9d1d9; font-family:'Courier New',monospace; padding:2rem; max-width:860px; margin:0 auto; }}
    h1 {{ font-size:1.6rem; color:#58a6ff; margin-bottom:.25rem; letter-spacing:.05em; }}
    .subtitle {{ color:#8b949e; font-size:.85rem; margin-bottom:2rem; }}
    .timeline {{ border-left:2px solid #21262d; padding-left:1.5rem; display:flex; flex-direction:column; gap:.75rem; }}
    .event {{ display:flex; align-items:baseline; gap:.75rem; padding:.5rem .75rem; border-radius:6px; background:#161b22; border:1px solid #21262d; flex-wrap:wrap; }}
    .event.note {{ border-color:#388bfd44; background:#0d1117; }}
    .time {{ font-size:.8rem; color:#8b949e; min-width:3.5rem; flex-shrink:0; }}
    .badge {{ font-size:.65rem; padding:.15rem .5rem; border-radius:999px; font-weight:bold; flex-shrink:0; text-transform:uppercase; letter-spacing:.05em; }}
    .app-badge  {{ background:#21262d; color:#58a6ff; }}
    .note-badge {{ background:#1f3a1f; color:#3fb950; }}
    .content {{ font-size:.88rem; color:#e6edf3; flex:1; word-break:break-word; }}
    .empty {{ color:#8b949e; font-style:italic; padding:1rem 0; }}
  </style>
</head>
<body>
  <h1>🌊 Activity-Logger</h1>
  <p class="subtitle">Daily timeline — {date_str} &nbsp;·&nbsp; {len(events)} events</p>
  <div class="timeline">{rows_html}</div>
</body>
</html>"""


def make_report(date_str: str):
    """Fetch events, render HTML, write to file, open in browser."""
    if date_str.lower() == "today":
        date_str = date.today().isoformat()

    events = get_events_for_day(date_str)
    html = generate_html(events, date_str)

    out_path = os.path.join(os.path.dirname(__file__), f"report_{date_str}.html")
    with open(out_path, "w", encoding="utf-8") as f:
        f.write(html)

    print(f"[report] Generated: {out_path}")
    webbrowser.open(f"file://{os.path.abspath(out_path)}")


if __name__ == "__main__":
    target = sys.argv[1] if len(sys.argv) > 1 else "today"
    make_report(target)
```

---

## `Activity-Logger.py`

```python
"""
Activity-Logger.py — Main CLI entrypoint.

Commands:
  python Activity-Logger.py note "finished the auth module"
  python Activity-Logger.py report today
  python Activity-Logger.py report 2025-03-31
  python Activity-Logger.py search "navbar"
"""

import sys
from db import init_db, insert_note, search_events
from report import make_report
from datetime import datetime


def cmd_note(args):
    """Drop a manual timestamped note into the database."""
    if not args:
        print("Usage: python Activity-Logger.py note \"your note here\"")
        sys.exit(1)
    content = " ".join(args)
    insert_note(content)
    print(f"[Activity-Logger] Note saved: \"{content}\"")


def cmd_report(args):
    """Generate an HTML timeline report for a given date."""
    date_str = args[0] if args else "today"
    make_report(date_str)


def cmd_search(args):
    """Search all activity + notes for a keyword."""
    if not args:
        print("Usage: python Activity-Logger.py search \"keyword\"")
        sys.exit(1)

    query = " ".join(args)
    results = search_events(query)

    if not results:
        print(f"[Activity-Logger] No results for: \"{query}\"")
        return

    print(f"[Activity-Logger] {len(results)} result(s) for \"{query}\":\n")
    for r in results:
        ts = datetime.fromisoformat(r["timestamp"]).strftime("%Y-%m-%d %H:%M")
        if r["type"] == "note":
            print(f"  {ts}  📝 NOTE     {r['content']}")
        else:
            print(f"  {ts}  🖥  {r['app']:<20} {r['title']}")


def print_help():
    print("""
🌊 Activity-Logger — passive activity logger

Commands:
  note <text>         Save a manual note
  report [date]       Generate HTML timeline ('today' or 'YYYY-MM-DD')
  search <keyword>    Search activity + notes
""")


if __name__ == "__main__":
    init_db()  # ensure DB is ready before any command

    if len(sys.argv) < 2:
        print_help()
        sys.exit(0)

    command = sys.argv[1].lower()
    rest = sys.argv[2:]

    if command == "note":
        cmd_note(rest)
    elif command == "report":
        cmd_report(rest)
    elif command == "search":
        cmd_search(rest)
    else:
        print(f"[Activity-Logger] Unknown command: '{command}'")
        print_help()
        sys.exit(1)
```
