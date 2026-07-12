# LAN-Time-Capsule

> A self-hosted digital time capsule that lets you write messages,
> attach files, and lock them until a future date---all stored locally
> on your own server.

## The Problem It Solves

People often want to leave reminders, goals, predictions, letters, or
memories for their future selves. Existing services are usually
cloud-based, require accounts, or may disappear over time.
LAN-Time-Capsule keeps everything local, private, and entirely under
your control.

## Features

-   Create text-based time capsules with a custom unlock date
-   Attach photos, PDFs, or other files to each capsule
-   Automatic unlocking once the selected date arrives
-   Countdown timer for every locked capsule
-   Browse all previously opened capsules
-   Optional password protection
-   Simple web interface accessible from any device on your LAN
-   SQLite database---no external services required
-   Fully self-hosted

## Tech Stack

-   Python 3
-   Flask
-   SQLite
-   HTML + CSS + Vanilla JavaScript

## Folder Structure

``` text
LAN-Time-Capsule/
├── server.py
├── database.db
├── uploads/
├── templates/
├── static/
└── README.md
```

## server.py

``` python
from flask import Flask
import sqlite3
from datetime import datetime

app = Flask(__name__)
DB = "database.db"

def init():
    conn = sqlite3.connect(DB)
    conn.execute('''CREATE TABLE IF NOT EXISTS capsules(
    id INTEGER PRIMARY KEY,
    title TEXT,
    message TEXT,
    unlock_date TEXT)''')
    conn.commit()
    conn.close()

@app.route("/")
def home():
    conn = sqlite3.connect(DB)
    rows = conn.execute("SELECT title,message,unlock_date FROM capsules").fetchall()
    conn.close()
    now = datetime.now()
    html = "<h1>LAN Time Capsule</h1>"
    for title, message, unlock in rows:
        if now >= datetime.fromisoformat(unlock):
            html += f"<h2>{title}</h2><p>{message}</p><hr>"
        else:
            html += f"<h2>{title}</h2><p>🔒 Locked until {unlock}</p><hr>"
    return html

if __name__ == "__main__":
    init()
    app.run(host="0.0.0.0", port=5050)
```

## Future Improvements

-   End-to-end encryption
-   Shared capsules
-   Email notifications
-   Voice notes
-   Dark mode

## License

MIT License
