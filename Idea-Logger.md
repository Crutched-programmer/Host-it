# Idea-Logger

> A local plaintext idea inbox with a CLI and LAN web UI that timestamps every thought and never lets you lose one.

## The Problem It Solves

Ideas arrive at random — mid-terminal session, on another device across the room, or at 2AM. Most die because the friction of opening a notes app is just high enough to kill the impulse. Idea-Logger removes that friction: one command or one browser tab, and the idea is logged, timestamped, and safe.

## Features

* Instant CLI capture — one command to dump an idea from anywhere in the terminal
* LAN web UI — accessible from any device on your local network (phone, tablet, another PC)
* Plaintext storage — all ideas saved to a single `.txt` file, human-readable forever
* Auto-timestamping — every entry tagged with date and time, no manual effort
* Tag support — prefix ideas with `#tag` for loose categorization
* Search — filter ideas by keyword or tag from the CLI
* No database, no login, no cloud — pure local, zero dependencies beyond Python

## Tech Stack

* Python 3 (stdlib only — `http.server`, `argparse`, `datetime`, `pathlib`)
* HTML + vanilla JS (web UI, no frameworks)
* Plaintext flat file storage

## Requirements

* Python 3.6+
* A Linux system (or WSL)
* A browser (for web UI)

No pip installs required.

## Install & Run

```bash
# Clone or copy the project
git clone https://github.com/Crutched_programmer/idea-logger
cd idea-logger

# Make the CLI script executable
chmod +x idea.py

# Optional: add to PATH for global access
echo 'alias idea="python3 /path/to/idea-logger/idea.py"' >> ~/.bashrc
source ~/.bashrc

# Start the web server (default port 6060)
python3 server.py
```

Access the web UI at `http://localhost:6060` or `http://<your-lan-ip>:6060` from any device on the network.

## Usage

### CLI

```bash
# Log an idea instantly
python3 idea.py "Build a rack-mount enclosure for the old PC"

# Log with a tag
python3 idea.py "#hardware Build a rack-mount enclosure for the old PC"

# List all ideas (newest first)
python3 idea.py --list

# Search by keyword
python3 idea.py --search "rack"

# Search by tag
python3 idea.py --search "#hardware"

# Show total idea count
python3 idea.py --count
```

### Web UI

- Open `http://<lan-ip>:6060` on any device
- Type an idea into the text box and hit **Log It**
- All ideas are displayed in reverse-chronological order
- Use the search bar to filter in real time

## How It Works

```
idea.py (CLI)
    |
    v
ideas.txt  <------>  server.py (HTTP server)
                          |
                          v
                     Web UI (browser)
```

- `idea.py` appends a new line to `ideas.txt` in the format:
  `[YYYY-MM-DD HH:MM:SS] #tag idea text here`
- `server.py` serves a single-page HTML UI and exposes two endpoints:
  - `GET /ideas` — returns all lines from `ideas.txt` as JSON
  - `POST /log` — appends a new idea to `ideas.txt`
- The web UI polls `/ideas` on load and on every submission, no page refresh needed

### idea.py

```python
#!/usr/bin/env python3
# idea.py — CLI client for Idea-Logger
# Usage: python3 idea.py "your idea here" | --list | --search <term> | --count

import argparse
import sys
from datetime import datetime
from pathlib import Path

# Path to the flat-file idea store — change if you want to store elsewhere
IDEAS_FILE = Path(__file__).parent / "ideas.txt"


def log_idea(text):
    """Append a timestamped idea to the ideas file."""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    entry = f"[{timestamp}] {text}\n"
    with open(IDEAS_FILE, "a", encoding="utf-8") as f:
        f.write(entry)
    print(f"Logged: {entry.strip()}")


def list_ideas():
    """Print all ideas in reverse-chronological order (newest first)."""
    if not IDEAS_FILE.exists():
        print("No ideas logged yet.")
        return
    lines = IDEAS_FILE.read_text(encoding="utf-8").strip().splitlines()
    for line in reversed(lines):
        print(line)


def search_ideas(query):
    """Search ideas by keyword or tag (case-insensitive)."""
    if not IDEAS_FILE.exists():
        print("No ideas logged yet.")
        return
    lines = IDEAS_FILE.read_text(encoding="utf-8").strip().splitlines()
    results = [l for l in lines if query.lower() in l.lower()]
    if results:
        for line in reversed(results):
            print(line)
    else:
        print(f"No results for '{query}'.")


def count_ideas():
    """Print total number of logged ideas."""
    if not IDEAS_FILE.exists():
        print("0 ideas logged.")
        return
    count = len(IDEAS_FILE.read_text(encoding="utf-8").strip().splitlines())
    print(f"{count} idea(s) logged.")


def main():
    parser = argparse.ArgumentParser(description="Idea-Logger CLI")
    parser.add_argument("idea", nargs="?", help="Idea text to log")
    parser.add_argument("--list", action="store_true", help="List all ideas")
    parser.add_argument("--search", metavar="QUERY", help="Search ideas by keyword or tag")
    parser.add_argument("--count", action="store_true", help="Show total idea count")
    args = parser.parse_args()

    if args.idea:
        log_idea(args.idea)
    elif args.list:
        list_ideas()
    elif args.search:
        search_ideas(args.search)
    elif args.count:
        count_ideas()
    else:
        parser.print_help()


if __name__ == "__main__":
    main()
```

### server.py

```python
#!/usr/bin/env python3
# server.py — LAN web server for Idea-Logger
# Run this to expose the web UI on your local network
# Access at http://<your-ip>:6060

import json
from datetime import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer
from pathlib import Path
from urllib.parse import parse_qs, urlparse

# Port the server listens on — change if 6060 is taken
PORT = 6060

# Path to the flat-file idea store — must match idea.py
IDEAS_FILE = Path(__file__).parent / "ideas.txt"

# Inline HTML for the web UI — kept in one file to avoid serving static assets
HTML = """<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Idea-Logger</title>
  <style>
    /* Minimal dark UI — easy on the eyes at 2AM */
    body { font-family: monospace; background: #111; color: #eee; max-width: 700px; margin: 40px auto; padding: 0 20px; }
    h1 { color: #7ec8e3; }
    textarea { width: 100%; background: #1e1e1e; color: #eee; border: 1px solid #444; padding: 10px; font-size: 1rem; resize: vertical; border-radius: 4px; }
    button { margin-top: 8px; padding: 8px 20px; background: #7ec8e3; color: #111; border: none; cursor: pointer; font-weight: bold; border-radius: 4px; }
    button:hover { background: #5ab0cc; }
    input[type=text] { width: 100%; background: #1e1e1e; color: #eee; border: 1px solid #444; padding: 8px; font-size: 0.95rem; margin: 10px 0; border-radius: 4px; }
    #ideas { margin-top: 20px; }
    .entry { border-bottom: 1px solid #2a2a2a; padding: 8px 0; font-size: 0.9rem; line-height: 1.5; }
    .ts { color: #888; font-size: 0.8rem; }
    .tag { color: #f4a261; }
    .count { color: #888; font-size: 0.85rem; margin-top: 8px; }
  </style>
</head>
<body>
  <h1>Idea-Logger</h1>
  <textarea id="input" rows="3" placeholder="What's the idea?"></textarea><br>
  <button onclick="logIdea()">Log It</button>
  <input type="text" id="search" placeholder="Search ideas..." oninput="renderIdeas()">
  <div class="count" id="count"></div>
  <div id="ideas"></div>

  <script>
    let allIdeas = [];

    // Fetch all ideas from server on load
    async function fetchIdeas() {
      const res = await fetch('/ideas');
      allIdeas = await res.json();
      renderIdeas();
    }

    // Render ideas filtered by search query
    function renderIdeas() {
      const query = document.getElementById('search').value.toLowerCase();
      const filtered = allIdeas.filter(l => l.toLowerCase().includes(query));
      document.getElementById('count').textContent = filtered.length + ' idea(s)';
      document.getElementById('ideas').innerHTML = [...filtered].reverse().map(line => {
        // Highlight tags in orange
        const highlighted = line.replace(/(#\\w+)/g, '<span class="tag">$1</span>');
        return `<div class="entry">${highlighted}</div>`;
      }).join('');
    }

    // POST a new idea to server, then refresh
    async function logIdea() {
      const text = document.getElementById('input').value.trim();
      if (!text) return;
      await fetch('/log', { method: 'POST', body: text });
      document.getElementById('input').value = '';
      await fetchIdeas();
    }

    // Allow Ctrl+Enter to submit
    document.addEventListener('keydown', e => {
      if (e.ctrlKey && e.key === 'Enter') logIdea();
    });

    fetchIdeas();
  </script>
</body>
</html>"""


class IdeaHandler(BaseHTTPRequestHandler):

    def log_message(self, format, *args):
        # Suppress default access logs to keep terminal clean
        pass

    def do_GET(self):
        if self.path == "/":
            # Serve the web UI
            self.send_response(200)
            self.send_header("Content-Type", "text/html")
            self.end_headers()
            self.wfile.write(HTML.encode("utf-8"))

        elif self.path == "/ideas":
            # Return all ideas as a JSON array of lines
            if IDEAS_FILE.exists():
                lines = IDEAS_FILE.read_text(encoding="utf-8").strip().splitlines()
            else:
                lines = []
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps(lines).encode("utf-8"))

        else:
            self.send_response(404)
            self.end_headers()

    def do_POST(self):
        if self.path == "/log":
            # Read idea text from POST body and append to file
            length = int(self.headers.get("Content-Length", 0))
            text = self.rfile.read(length).decode("utf-8").strip()
            if text:
                timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                entry = f"[{timestamp}] {text}\n"
                with open(IDEAS_FILE, "a", encoding="utf-8") as f:
                    f.write(entry)
            self.send_response(200)
            self.end_headers()
        else:
            self.send_response(404)
            self.end_headers()


if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", PORT), IdeaHandler)
    print(f"Idea-Logger running at http://0.0.0.0:{PORT}")
    print(f"LAN access: http://<your-ip>:{PORT}")
    print(f"Storing ideas in: {IDEAS_FILE.resolve()}")
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\nServer stopped.")
```

## Customization

* **Change port** — edit `PORT = 6060` in `server.py`
* **Change storage location** — edit `IDEAS_FILE` in both `idea.py` and `server.py`
* **Auto-start on boot** — add a cron job or systemd service pointing to `server.py`
* **Add categories** — extend the tag system by pre-defining tag colors in the HTML CSS block
* **Export** — `ideas.txt` is plain text; pipe it anywhere (`cat ideas.txt | grep "#hardware"`)

## Limitations

* Single flat file — not suitable for thousands of entries without pagination
* No edit or delete via web UI (intentional — ideas are append-only by design)
* No authentication — LAN-only, do not expose to the internet
* Search is client-side — fast for small files, slower if ideas.txt grows very large

## Future Improvements

* Delete/archive ideas via web UI
* Export to JSON or Markdown
* Daily digest — cron job that emails or notifies a summary of the day's ideas
* Priority flags (`!high`, `!low`) with visual sorting
* Sync `ideas.txt` to another machine via SyncDrop

## Notes

* `ideas.txt` is the single source of truth — back it up with your usual rsync/SyncDrop flow
* The web UI works well on mobile; use it as a homescreen bookmark for instant access
* Pair with the `idea` shell alias for fastest possible capture from any terminal session
