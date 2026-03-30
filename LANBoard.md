===FILE: README.md===
# LANBoard
> A self-hosted LAN message board for your local network — no accounts, no internet, no cloud.

## The Problem It Solves
Sometimes you need a frictionless shared scratchpad for a household, small office, or LAN party with no cloud, no logins, and no surveillance. LANBoard runs entirely on your local network. Open the IP in any browser, post a message, and it vanishes after 24 hours automatically.

## Features
- Post anonymous messages visible to anyone on the same LAN
- Messages auto-expire after 24 hours (configurable)
- Tag messages by category: idea / question / rant / info
- Minimal dark web UI — works on any browser including ancient ones
- Zero pip dependencies — pure Python stdlib only

## Tech Stack
- Python 3.6+ (stdlib only — http.server + sqlite3)
- SQLite (auto-created on first run)
- Plain HTML/CSS served inline (no frameworks)

## Requirements
- Linux with Python 3.6 or later installed
- Nothing else — zero pip installs needed

## Install & Run
1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/lanboard.git
   cd lanboard
   ```
2. Run the server:
   ```bash
   python3 server.py
   ```
3. Open http://<your-server-ip>:8080 on any device on your LAN
4. Start dropping messages

Optional — run on boot:
   ```bash
   crontab -e
   # add this line:
   @reboot python3 /path/to/lanboard/server.py &
   ```

## Folder Structure
```
lanboard/
├── README.md
├── server.py
└── lanboard.db       (auto-created on first run)
```

## Notes / Assumptions
- Messages stored in lanboard.db (SQLite, auto-created next to server.py)
- Expiry cleanup runs on every page load — no background thread needed
- No authentication — designed for trusted LAN use only
- Change PORT or EXPIRY_HOURS constants at the top of server.py

===FILE: server.py===
import sqlite3
import os
import time
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs

PORT = 8080
EXPIRY_HOURS = 24
DB_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "lanboard.db")
TAGS = ["idea", "question", "rant", "info"]
TAG_COLORS = {"idea": "#f0c040", "question": "#60b8f0", "rant": "#f07060", "info": "#80d080"}

def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS messages (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    body TEXT NOT NULL,
                    tag TEXT DEFAULT "info",
                    created_at INTEGER NOT NULL
                )''')
    conn.commit()
    conn.close()

def purge_expired():
    cutoff = int(time.time()) - (EXPIRY_HOURS * 3600)
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("DELETE FROM messages WHERE created_at < ?", (cutoff,))
    conn.commit()
    conn.close()

def get_messages():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT id, body, tag, created_at FROM messages ORDER BY created_at DESC")
    rows = c.fetchall()
    conn.close()
    return rows

def post_message(body, tag):
    if not body.strip():
        return
    if tag not in TAGS:
        tag = "info"
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("INSERT INTO messages (body, tag, created_at) VALUES (?, ?, ?)",
              (body.strip()[:500], tag, int(time.time())))
    conn.commit()
    conn.close()

def render_page(messages):
    rows_html = ""
    for (mid, body, tag, created_at) in messages:
        color = TAG_COLORS.get(tag, "#aaa")
        age_min = max(0, int((time.time() - created_at) / 60))
        age_str = f"{age_min}m ago" if age_min < 60 else f"{age_min // 60}h ago"
        rows_html += f'''
        <div class="msg">
            <span class="tag" style="background:{color}">{tag}</span>
            <span class="body">{body}</span>
            <span class="age">{age_str}</span>
        </div>'''

    if not rows_html:
        rows_html = '<p class="empty">No messages yet. Drop one below.</p>'

    tag_options = "".join(f'<option value="{t}">{t}</option>' for t in TAGS)

    return f'''<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>LANBoard</title>
<style>
  body {{ font-family: monospace; background: #111; color: #eee; max-width: 700px; margin: 40px auto; padding: 0 16px; }}
  h1 {{ color: #f0c040; letter-spacing: 2px; }}
  .msg {{ border-left: 3px solid #333; padding: 8px 12px; margin: 10px 0; background: #1a1a1a; }}
  .tag {{ padding: 2px 8px; border-radius: 3px; font-size: 0.75em; color: #111; font-weight: bold; margin-right: 8px; }}
  .body {{ font-size: 0.95em; }}
  .age {{ float: right; color: #555; font-size: 0.8em; }}
  .empty {{ color: #555; }}
  form {{ margin-top: 30px; border-top: 1px solid #333; padding-top: 20px; }}
  textarea {{ width: 100%; background: #1a1a1a; color: #eee; border: 1px solid #444; padding: 8px; font-family: monospace; font-size: 0.9em; resize: vertical; box-sizing: border-box; }}
  select {{ background: #1a1a1a; color: #eee; border: 1px solid #444; padding: 6px; font-family: monospace; }}
  button {{ background: #f0c040; color: #111; border: none; padding: 8px 20px; font-family: monospace; font-weight: bold; cursor: pointer; margin-top: 8px; }}
  button:hover {{ background: #ffd060; }}
  .hint {{ color: #555; font-size: 0.75em; margin-top: 6px; }}
</style>
</head>
<body>
<h1>// LANBOARD</h1>
<p class="hint">Messages expire after {EXPIRY_HOURS}h &mdash; LAN only &mdash; anonymous</p>
<div id="board">{rows_html}</div>
<form method="POST" action="/post">
  <textarea name="body" rows="3" maxlength="500" placeholder="post a message..."></textarea><br>
  <select name="tag">{tag_options}</select>
  <button type="submit">POST</button>
  <p class="hint">max 500 chars</p>
</form>
</body>
</html>'''

class Handler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass  # suppress request logs

    def do_GET(self):
        purge_expired()
        messages = get_messages()
        html = render_page(messages)
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.end_headers()
        self.wfile.write(html.encode("utf-8"))

    def do_POST(self):
        if self.path == "/post":
            length = int(self.headers.get("Content-Length", 0))
            raw = self.rfile.read(length).decode("utf-8")
            params = parse_qs(raw)
            body = params.get("body", [""])[0]
            tag = params.get("tag", ["info"])[0]
            post_message(body, tag)
        self.send_response(303)
        self.send_header("Location", "/")
        self.end_headers()

if __name__ == "__main__":
    init_db()
    print(f"LANBoard running on http://0.0.0.0:{PORT}")
    print(f"Messages expire after {EXPIRY_HOURS} hours. Ctrl+C to stop.")
    HTTPServer(("0.0.0.0", PORT), Handler).serve_forever()
