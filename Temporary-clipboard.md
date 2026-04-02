<h1>Temporary-clipboard</h1>

> Instantly share a file or text snippet to any device on your local network via a one-time link that self-destructs after download.

## The Problem It Solves

You need to move a file from your PC to your phone, or paste a snippet of text to another machine. Emailing yourself is slow, USB is a cable hunt, and cloud drives need accounts. Temporary-clipboard spins up a temporary link on your local network that serves the file or text once, then deletes it. Open the link on any device, grab the thing, done.

## Features

* Upload a file or paste text through a browser
* Generates a short random link (e.g. `http://192.168.1.x:7070/abc12`)
* Link works exactly once — file is deleted after first download
* Optional expiry timer (default 10 minutes) if nobody downloads it
* QR code displayed on the upload page for easy phone scanning
* No accounts, no internet, no leftover files

## Tech Stack

* Python 3
* Flask — web server
* `qrcode` — generates the QR code as a PNG
* `secrets` — for generating the random token
* Plain HTML — no frontend framework

## Requirements

* Python 3.8+
* Flask
* qrcode
* Pillow (required by qrcode for image output)

## Install & Run

```bash
mkdir Temporary-clipboard && cd Temporary-clipboard

pip install flask qrcode Pillow

# Save Temporary-clipboard.py here, then run:
python3 Temporary-clipboard.py
```

Open `http://<your-local-IP>:7070` from any device on your network.

```bash
# Find your local IP
hostname -I | awk '{print $1}'
```

## Usage

* Open the page on your PC
* Either choose a file to upload, or type/paste text into the text box
* Hit Drop — a one-time link and QR code appear instantly
* Scan the QR code with your phone, or open the link on any device
* The file or text is served once, then wiped
* If nobody downloads within 10 minutes, it expires automatically

## How It Works

```python
#!/usr/bin/env python3
"""
Temporary-clipboard.py — One-time local file and text sharing server.

Generates a random token link for each upload. The file or text
is held in memory (or as a temp file) and deleted after one download
or after the expiry timeout, whichever comes first.
"""

import os
import io
import time
import secrets
import threading
import tempfile
import qrcode
import base64
from flask import Flask, request, redirect, render_template_string, send_file, abort

app = Flask(__name__)

# ──────────────────────────────────────────────
# CONFIGURATION
# ──────────────────────────────────────────────

PORT = 7070

# How long (in seconds) a drop lives before auto-expiry if not downloaded
EXPIRY_SECONDS = 600  # 10 minutes

# In-memory store for active drops
# Format: { token: { "type": "file"|"text", "data": ..., "filename": str, "expires": float } }
DROPS = {}

# Lock for thread-safe access to DROPS dict
LOCK = threading.Lock()

# ──────────────────────────────────────────────
# HTML TEMPLATE
# ──────────────────────────────────────────────

TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Temporary-clipboard</title>
  <style>
    body { font-family: monospace; max-width: 600px; margin: 40px auto; padding: 0 16px; background: #111; color: #eee; }
    h1 { font-size: 1.4rem; }
    input[type=file], textarea { width: 100%; background: #222; color: #eee; border: 1px solid #444; padding: 10px; box-sizing: border-box; font-family: monospace; }
    textarea { height: 120px; resize: vertical; margin-top: 12px; }
    button { padding: 10px 24px; margin-top: 10px; background: #444; color: #eee; border: none; cursor: pointer; font-size: 1rem; }
    button:hover { background: #666; }
    .drop-box { margin-top: 32px; padding: 20px; border: 1px solid #444; text-align: center; }
    .drop-box a { color: #7af; font-size: 1.1rem; word-break: break-all; }
    .drop-box img { margin-top: 16px; display: block; margin-left: auto; margin-right: auto; }
    .expiry { color: #777; font-size: 0.8rem; margin-top: 8px; }
    hr { border-color: #333; margin: 24px 0; }
    label { color: #aaa; font-size: 0.85rem; }
  </style>
</head>
<body>
  <h1>Temporary-clipboard</h1>
  <p>One-time local file or text share. Link dies after one download.</p>

  <form method="POST" action="/drop" enctype="multipart/form-data">
    <label>File</label><br>
    <input type="file" name="file"><br>
    <label>Or paste text</label>
    <textarea name="text" placeholder="Paste anything here..."></textarea><br>
    <button type="submit">Drop</button>
  </form>

  {% if link %}
  <div class="drop-box">
    <p>One-time link:</p>
    <a href="{{ link }}">{{ link }}</a>
    <p class="expiry">Expires in 10 minutes or after one download.</p>
    {% if qr %}
    <img src="data:image/png;base64,{{ qr }}" width="180" height="180" alt="QR Code">
    {% endif %}
  </div>
  {% endif %}

</body>
</html>
"""


# ──────────────────────────────────────────────
# HELPERS
# ──────────────────────────────────────────────

def make_qr(url: str) -> str:
    """
    Generate a QR code for the given URL and return it as a base64 PNG string.
    This gets embedded directly in the HTML page — no file needed.
    """
    img = qrcode.make(url)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    return base64.b64encode(buf.getvalue()).decode("utf-8")


def schedule_expiry(token: str) -> None:
    """
    Start a background thread that deletes the drop after EXPIRY_SECONDS
    if it hasn't been downloaded yet.
    """
    def expire():
        time.sleep(EXPIRY_SECONDS)
        with LOCK:
            drop = DROPS.pop(token, None)
            # If it was a temp file, clean it up from disk
            if drop and drop["type"] == "file" and os.path.exists(drop["data"]):
                os.remove(drop["data"])

    t = threading.Thread(target=expire, daemon=True)
    t.start()


def get_local_ip() -> str:
    """
    Get the local network IP of this machine so we can build the full link URL.
    Falls back to 127.0.0.1 if detection fails.
    """
    import socket
    try:
        # Connect to an external address just to determine outbound interface IP
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return ip
    except Exception:
        return "127.0.0.1"


# ──────────────────────────────────────────────
# ROUTES
# ──────────────────────────────────────────────

@app.route("/")
def index():
    """Serve the upload page with no active drop shown."""
    return render_template_string(TEMPLATE, link=None, qr=None)


@app.route("/drop", methods=["POST"])
def drop():
    """
    Accept a file upload or text paste.
    Generate a random token, store the content, return the one-time link + QR.
    """
    token = secrets.token_urlsafe(6)  # short random token, e.g. "xK9mPq"
    expires = time.time() + EXPIRY_SECONDS

    file = request.files.get("file")
    text = request.form.get("text", "").strip()

    if file and file.filename:
        # Save uploaded file to a system temp path on disk
        suffix = os.path.splitext(file.filename)[1]
        tmp = tempfile.NamedTemporaryFile(delete=False, suffix=suffix)
        file.save(tmp.name)
        with LOCK:
            DROPS[token] = {
                "type": "file",
                "data": tmp.name,       # path to temp file on disk
                "filename": file.filename,
                "expires": expires
            }

    elif text:
        # Store text directly in memory as bytes
        with LOCK:
            DROPS[token] = {
                "type": "text",
                "data": text.encode("utf-8"),
                "filename": "drop.txt",
                "expires": expires
            }
    else:
        # Nothing was submitted — go back to the upload page
        return redirect("/")

    schedule_expiry(token)

    # Build the full URL using the machine's detected local IP
    ip = get_local_ip()
    link = f"http://{ip}:{PORT}/get/{token}"
    qr = make_qr(link)

    return render_template_string(TEMPLATE, link=link, qr=qr)


@app.route("/get/<token>")
def get_drop(token: str):
    """
    Serve the file or text for the given token.
    Deletes the drop immediately after serving — one-time only.
    """
    with LOCK:
        drop = DROPS.pop(token, None)  # remove from store on first access

    if not drop:
        # Already downloaded or expired
        abort(404)

    if drop["type"] == "file":
        path = drop["data"]
        filename = drop["filename"]

        def cleanup():
            """Delete the temp file a couple seconds after the response is sent."""
            try:
                time.sleep(2)
                os.remove(path)
            except FileNotFoundError:
                pass

        response = send_file(path, download_name=filename, as_attachment=True)
        threading.Thread(target=cleanup, daemon=True).start()
        return response

    elif drop["type"] == "text":
        # Serve text as a downloadable .txt file straight from memory
        buf = io.BytesIO(drop["data"])
        return send_file(buf, download_name="drop.txt", as_attachment=True, mimetype="text/plain")


# ──────────────────────────────────────────────
# ENTRY POINT
# ──────────────────────────────────────────────

if __name__ == "__main__":
    print(f"Temporary-clipboard running at http://{get_local_ip()}:{PORT}")
    # 0.0.0.0 exposes the server to all devices on the local network
    app.run(host="0.0.0.0", port=PORT, debug=False)
```

On each drop, a random 6-character token is generated via `secrets.token_urlsafe`. Files go to a system temp path on disk, text stays in memory as bytes. A background thread deletes the entry after `EXPIRY_SECONDS`. The `/get/<token>` route pops the entry out of the dict on first hit — a second request finds nothing and gets a 404.

## Customization

* Increase `EXPIRY_SECONDS` for more time to scan the QR code
* Remove `as_attachment=True` on text drops to display in browser instead of downloading
* Add a max file size check: `if file.content_length > 50 * 1024 * 1024: abort(413)`
* Swap the in-memory dict for a SQLite file for persistence across restarts

## Limitations

* Files are written to `/tmp` — large files will eat disk space temporarily
* In-memory text drops are lost on server restart
* No authentication — anyone on your local network can use the drop page
* QR code requires phone and PC to be on the same wifi network

## Future Improvements

* Multi-file zip bundling — select several files, get one download link
* Configurable download count limit per drop (allow N downloads, not just 1)
* Drag-and-drop file area on the upload page

## Notes

* The QR code is base64-embedded directly in the page — no extra file or route needed
* Pairs naturally with linkdrop if you want persistent links vs one-time drops
* Run as a background process: `nohup python3 Temporary-clipboard.py &`
