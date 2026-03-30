# MoodCast
> A self-hosted micro weather station for your emotions — log your mood, get a personal "forecast" for the week.

## The Problem It Solves
You never remember how you actually felt last Tuesday, or why this week feels harder than last. MoodCast lets you log a daily mood score + note in seconds via any browser on your LAN. It then generates a personal "forecast" — a simple trend line showing whether your emotional weather is improving, stable, or stormy — no therapist login required.

## Features
- One-click daily mood logging (1–5 scale + optional note)
- Personal weekly "forecast" — trend + average displayed as ASCII weather symbols
- Full mood history with color-coded calendar heatmap
- Completely anonymous — no accounts, no names, just your machine
- Lightweight web UI accessible from any device on your LAN

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
   git clone https://github.com/yourusername/moodcast.git
   cd moodcast
   ```
2. Run the server:
   ```bash
   python3 server.py
   ```
3. Open http://<your-server-ip>:8080 on any device on your LAN

Optional — run on boot:
   ```bash
   crontab -e
   # add this line:
   @reboot python3 /path/to/moodcast/server.py &
   ```

## Folder Structure
```
moodcast/
├── README.md
├── server.py
└── moodcast.db       (auto-created on first run)
```

## Notes / Assumptions
- One mood entry per day is enforced — submitting again overwrites today's entry
- Forecast is calculated as a 7-day rolling average with a simple trend arrow
- Change PORT constant at the top of server.py if 8080 is taken

===FILE: server.py===
import sqlite3
import os
import time
import datetime
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs

PORT = 8080
DB_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)), "moodcast.db")

MOODS = {
    1: ("⛈", "#e05050", "stormy"),
    2: ("🌧", "#e09050", "cloudy"),
    3: ("⛅", "#e0d050", "mixed"),
    4: ("🌤", "#90d050", "clearing"),
    5: ("☀", "#50d090", "sunny"),
}

def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS entries (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    date TEXT UNIQUE NOT NULL,
                    score INTEGER NOT NULL,
                    note TEXT DEFAULT ""
                )''')
    conn.commit()
    conn.close()

def log_mood(score, note):
    today = datetime.date.today().isoformat()
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    # Overwrite if already logged today
    c.execute("INSERT INTO entries (date, score, note) VALUES (?, ?, ?) ON CONFLICT(date) DO UPDATE SET score=excluded.score, note=excluded.note",
              (today, score, note[:300]))
    conn.commit()
    conn.close()

def get_last_30():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT date, score, note FROM entries ORDER BY date DESC LIMIT 30")
    rows = c.fetchall()
    conn.close()
    return rows

def get_forecast():
    # Returns (avg_score, trend_symbol, label)
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT score FROM entries ORDER BY date DESC LIMIT 14")
    rows = [r[0] for r in c.fetchall()]
    conn.close()
    if len(rows) < 2:
        return None
    recent = sum(rows[:7]) / len(rows[:7])
    older = sum(rows[7:]) / len(rows[7:]) if len(rows) > 7 else recent
    trend = "↑" if recent > older + 0.3 else ("↓" if recent < older - 0.3 else "→")
    icon, color, label = MOODS.get(round(recent), MOODS[3])
    return round(recent, 1), trend, label, color, icon

def get_today_entry():
    today = datetime.date.today().isoformat()
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT score, note FROM entries WHERE date = ?", (today,))
    row = c.fetchone()
    conn.close()
    return row

def render_heatmap(entries):
    # Build a simple 30-day color grid
    entry_map = {e[0]: e[1] for e in entries}
    today = datetime.date.today()
    cells = ""
    for i in range(29, -1, -1):
        d = (today - datetime.timedelta(days=i)).isoformat()
        score = entry_map.get(d)
        if score:
            _, color, label = MOODS.get(score, MOODS[3])
            cells += f'<div class="cell" style="background:{color}" title="{d}: {label}"></div>'
        else:
            cells += f'<div class="cell empty" title="{d}: no entry"></div>'
    return cells

def render_page():
    entries = get_last_30()
    forecast = get_forecast()
    today_entry = get_today_entry()
    heatmap = render_heatmap(entries)

    forecast_html = ""
    if forecast:
        avg, trend, label, color, icon = forecast
        forecast_html = f'<div class="forecast" style="border-color:{color}"><span class="ficon">{icon}</span><span class="ftext">7-day forecast: <b style="color:{color}">{label}</b> &nbsp;{trend}&nbsp; avg {avg}/5</span></div>'
    else:
        forecast_html = '<div class="forecast">Not enough data yet — log a few days first.</div>'

    mood_buttons = ""
    for score, (icon, color, label) in MOODS.items():
        selected = " selected" if today_entry and today_entry[0] == score else ""
        mood_buttons += f'<button class="moodbtn{selected}" style="--c:{color}" onclick="setMood({score})" title="{label}">{icon}<span>{label}</span></button>'

    today_note = today_entry[1] if today_entry else ""
    logged_banner = '<p class="logged">✓ Mood logged for today — you can update it below.</p>' if today_entry else ""

    history_rows = ""
    for (date, score, note) in entries:
        icon, color, label = MOODS.get(score, MOODS[3])
        history_rows += f'<tr><td>{date}</td><td style="color:{color}">{icon} {label}</td><td class="note">{note}</td></tr>'

    return f'''<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>MoodCast</title>
<style>
  * {{ box-sizing: border-box; }}
  body {{ font-family: monospace; background: #0f0f0f; color: #ddd; max-width: 720px; margin: 40px auto; padding: 0 16px; }}
  h1 {{ color: #90d0ff; letter-spacing: 3px; margin-bottom: 4px; }}
  .sub {{ color: #444; font-size: 0.8em; margin-bottom: 24px; }}
  .forecast {{ border: 1px solid #333; border-left: 4px solid #555; padding: 12px 16px; margin-bottom: 24px; display: flex; align-items: center; gap: 12px; background: #1a1a1a; }}
  .ficon {{ font-size: 2em; }}
  .ftext {{ font-size: 0.9em; }}
  .heatmap {{ display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 24px; }}
  .cell {{ width: 20px; height: 20px; border-radius: 3px; background: #222; }}
  .cell.empty {{ background: #1a1a1a; border: 1px solid #2a2a2a; }}
  .section {{ color: #555; font-size: 0.75em; letter-spacing: 2px; text-transform: uppercase; margin-bottom: 10px; }}
  .moodrow {{ display: flex; gap: 8px; margin-bottom: 16px; flex-wrap: wrap; }}
  .moodbtn {{ background: #1a1a1a; border: 2px solid #333; color: #ddd; padding: 10px 14px; cursor: pointer; font-family: monospace; font-size: 0.85em; display: flex; flex-direction: column; align-items: center; gap: 4px; border-radius: 4px; transition: border-color 0.15s; }}
  .moodbtn:hover, .moodbtn.selected {{ border-color: var(--c); color: var(--c); }}
  textarea {{ width: 100%; background: #1a1a1a; color: #ddd; border: 1px solid #333; padding: 8px; font-family: monospace; font-size: 0.85em; resize: vertical; margin-bottom: 8px; }}
  .submitbtn {{ background: #90d0ff; color: #111; border: none; padding: 8px 20px; font-family: monospace; font-weight: bold; cursor: pointer; }}
  .submitbtn:hover {{ background: #b0e0ff; }}
  .logged {{ color: #50d090; font-size: 0.8em; }}
  table {{ width: 100%; border-collapse: collapse; font-size: 0.85em; margin-top: 16px; }}
  th {{ text-align: left; color: #555; border-bottom: 1px solid #222; padding: 6px 8px; }}
  td {{ padding: 6px 8px; border-bottom: 1px solid #1a1a1a; }}
  .note {{ color: #666; }}
  #moodinput {{ display: none; }}
</style>
</head>
<body>
<h1>⛅ MOODCAST</h1>
<p class="sub">your personal emotional weather station</p>

{forecast_html}

<p class="section">last 30 days</p>
<div class="heatmap">{heatmap}</div>

<p class="section">log today</p>
{logged_banner}
<form method="POST" action="/log">
  <input type="hidden" name="score" id="moodinput" value="">
  <div class="moodrow">{mood_buttons}</div>
  <textarea name="note" rows="2" maxlength="300" placeholder="optional note...">{today_note}</textarea>
  <button type="submit" class="submitbtn">LOG</button>
</form>

<p class="section">history</p>
<table>
  <tr><th>Date</th><th>Mood</th><th>Note</th></tr>
  {history_rows}
</table>

<script>
  function setMood(score) {{
    document.getElementById('moodinput').value = score;
    document.querySelectorAll('.moodbtn').forEach(b => b.classList.remove('selected'));
    document.querySelectorAll('.moodbtn')[score-1].classList.add('selected');
  }}
</script>
</body>
</html>'''

class Handler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        pass

    def do_GET(self):
        html = render_page()
        self.send_response(200)
        self.send_header("Content-Type", "text/html; charset=utf-8")
        self.end_headers()
        self.wfile.write(html.encode("utf-8"))

    def do_POST(self):
        if self.path == "/log":
            length = int(self.headers.get("Content-Length", 0))
            raw = self.rfile.read(length).decode("utf-8")
            params = parse_qs(raw)
            try:
                score = int(params.get("score", ["3"])[0])
                score = max(1, min(5, score))
            except ValueError:
                score = 3
            note = params.get("note", [""])[0]
            log_mood(score, note)
        self.send_response(303)
        self.send_header("Location", "/")
        self.end_headers()

if __name__ == "__main__":
    init_db()
    print(f"MoodCast running on http://0.0.0.0:{PORT}")
    print("Ctrl+C to stop.")
    HTTPServer(("0.0.0.0", PORT), Handler).serve_forever()
