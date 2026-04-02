# Link-Clipboard

> Paste a URL from your phone or browser, find it instantly on any device via your local network.

## The Problem It Solves

You find something on your phone and want it on your PC, or vice versa. AirDrop does not exist, clipboard sync needs cloud accounts, and messaging yourself feels stupid. Link-clipboard is a tiny local web server that holds your last 20 URLs. Open it from any device on your home network and paste or grab links instantly.

## Features

* Paste a URL from any device, see it on any other device
* Stores last 20 links with timestamps
* One-click copy button on each link
* No login, no account, no internet required
* Auto-detects page title of pasted URLs
* Runs on your home server, accessible via local IP

## Tech Stack

* Python 3
* Flask — tiny web server
* `requests` + `beautifulsoup4` — for fetching page titles
* Plain HTML/CSS — no frontend framework

## Requirements

* Python 3.8+
* Flask
* requests
* beautifulsoup4

## Install & Run

```bash
# Create project folder
mkdir Link-clipboard && cd Link-clipboard

# Install dependencies
pip install flask requests beautifulsoup4

# Save Link-clipboard.py into this folder, then run:
python3 Link-clipboard.py
```

Open `http://<your-PC-local-IP>:6070` from any device on your network.

Find your local IP with:
```bash
hostname -I | awk '{print $1}'
```

## Usage

* Open the page from your phone or PC
* Paste a URL into the box and hit Drop
* The link appears at the top of the list with its page title
* Hit Copy on any entry to grab it to clipboard
* Links auto-expire after the 20-link limit (oldest dropped first)

## How It Works

```python
#!/usr/bin/env python3
"""
Link-clipboard.py — A minimal local link-sharing server.
Stores the last 20 dropped URLs in memory and serves them
as a simple webpage accessible on your home network.
"""

from flask import Flask, request, redirect, render_template_string
from datetime import datetime
import requests
from bs4 import BeautifulSoup

app = Flask(__name__)

# In-memory store — holds last 20 links as list of dicts
# Each entry: { "url": str, "title": str, "time": str }
LINKS = []
MAX_LINKS = 20

# ──────────────────────────────────────────────
# HTML TEMPLATE — served for every page load
# ──────────────────────────────────────────────

TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Link-clipboard</title>
  <style>
    body { font-family: monospace; max-width: 640px; margin: 40px auto; padding: 0 16px; background: #111; color: #eee; }
    h1 { font-size: 1.4rem; margin-bottom: 24px; }
    input[type=text] { width: 100%; padding: 10px; font-size: 1rem; background: #222; color: #eee; border: 1px solid #444; box-sizing: border-box; }
    button { padding: 10px 20px; margin-top: 8px; background: #444; color: #eee; border: none; cursor: pointer; font-size: 1rem; }
    button:hover { background: #666; }
    .link-item { border-bottom: 1px solid #333; padding: 12px 0; }
    .link-title { font-weight: bold; margin-bottom: 4px; }
    .link-url { color: #aaa; word-break: break-all; font-size: 0.85rem; }
    .link-time { color: #555; font-size: 0.75rem; margin-top: 4px; }
    .copy-btn { padding: 4px 10px; font-size: 0.8rem; float: right; }
  </style>
</head>
<body>
  <h1>Link-clipboard</h1>

  <!-- URL input form -->
  <form method="POST" action="/drop">
    <input type="text" name="url" placeholder="Paste URL here..." autofocus>
    <button type="submit">Drop</button>
  </form>

  <!-- Link list -->
  <div style="margin-top: 32px;">
    {% for link in links %}
    <div class="link-item">
      <div class="link-title">
        <!-- Copy button triggers JS clipboard copy -->
        <button class="copy-btn" onclick="navigator.clipboard.writeText('{{ link.url }}')">Copy</button>
        {{ link.title }}
      </div>
      <div class="link-url"><a href="{{ link.url }}" target="_blank" style="color:#7af">{{ link.url }}</a></div>
      <div class="link-time">{{ link.time }}</div>
    </div>
    {% endfor %}
    {% if not links %}
    <p style="color:#555">No links yet. Drop one above.</p>
    {% endif %}
  </div>
</body>
</html>
"""


def fetch_title(url: str) -> str:
    """
    Try to fetch the page title of a URL.
    Returns the title string, or a truncated URL as fallback.
    Times out after 3 seconds to avoid hanging.
    """
    try:
        res = requests.get(url, timeout=3, headers={"User-Agent": "Link-clipboard/1.0"})
        soup = BeautifulSoup(res.text, "html.parser")
        title = soup.title.string.strip() if soup.title else url[:60]
        return title[:80]  # cap title length
    except Exception:
        # Network error, bad URL, timeout — just use the URL itself
        return url[:60]


@app.route("/")
def index():
    """Serve the main page with the current link list."""
    # Show newest links first
    return render_template_string(TEMPLATE, links=reversed(LINKS))


@app.route("/drop", methods=["POST"])
def drop():
    """Accept a posted URL, fetch its title, store it, redirect back."""
    url = request.form.get("url", "").strip()

    # Basic validation — must start with http
    if not url.startswith("http"):
        return redirect("/")

    title = fetch_title(url)
    timestamp = datetime.now().strftime("%d %b %Y, %H:%M")

    # Add to list
    LINKS.append({"url": url, "title": title, "time": timestamp})

    # Keep only the last MAX_LINKS entries
    if len(LINKS) > MAX_LINKS:
        LINKS.pop(0)

    return redirect("/")


# ──────────────────────────────────────────────
# ENTRY POINT
# ──────────────────────────────────────────────

if __name__ == "__main__":
    # 0.0.0.0 makes it reachable from other devices on the local network
    app.run(host="0.0.0.0", port=6070, debug=False)
```

Flask serves one page. When you POST a URL, it fetches the page title in the background and stores the entry in a list. The template renders everything inline — no database, no static files.

## Customization

* Change `MAX_LINKS` to store more or fewer entries
* Add a password field at the top of the template for basic protection
* Persist links across restarts by writing `LINKS` to a JSON file on each drop
* Add a Delete button per entry with a `/delete/<index>` route

## Limitations

* Links are lost on restart (in-memory only by default)
* No authentication — anyone on your local network can see and add links
* Title fetch adds ~1–3 seconds delay per drop if the site is slow

## Future Improvements

* JSON file persistence so links survive restarts
* QR code display for each link (scannable from phone)
* Browser extension button that drops the current tab instantly

## Notes

* Bookmark `http://<local-IP>:6070` on your phone for one-tap access
* Runs comfortably under 30MB RAM — no problem on your Dell i3 setup
* Pair with a startup script or systemd service to auto-launch on boot
