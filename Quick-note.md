# Quick-note

> A minimal local sticky-note server — jot something down from any device on your network, find it there when you come back.

## The Problem It Solves

You want to quickly write something down — a thought, a command, a number — and have it visible from your PC or phone without opening a notes app, syncing to a cloud, or messaging yourself. Quick-note is a single persistent text box served over your local network. Whatever you type is saved instantly. It is always there when you come back.

## Features

* One big text area, always in sync across devices
* Auto-saves as you type (no save button needed)
* Persists across server restarts via a plain text file
* Accessible from any device on your local network
* Tiny — one Python file, one data file

## Tech Stack

* Python 3
* Flask — web server
* Plain HTML + JavaScript — auto-save via fetch on input event

## Requirements

* Python 3.8+
* Flask

## Install & Run

```bash
mkdir Quick-note && cd Quick-note

pip install flask

# Save Quick-note.py here, then run:
python3 Quick-note.py
```

Open `http://<your-local-IP>:5500` from any device on your network.

```bash
# Find your local IP
hostname -I | awk '{print $1}'
```

## Usage

* Open the page from your PC or phone
* Start typing — content saves automatically after each keystroke
* Come back later, refresh the page — your note is still there
* To clear it, just select all and delete

## How It Works

```python
#!/usr/bin/env python3
"""
Quick-note.py — A persistent single-note server for your local network.

Serves a text area that auto-saves to a local file on every keystroke.
No database, no login, no friction.
"""

import os
from flask import Flask, request, jsonify, render_template_string

app = Flask(__name__)

# Path to the file where the note is stored on disk
NOTE_FILE = os.path.join(os.path.dirname(__file__), "note.txt")

# ──────────────────────────────────────────────
# HTML TEMPLATE
# ──────────────────────────────────────────────

TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Quick-note</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { background: #111; display: flex; flex-direction: column; height: 100vh; padding: 20px; font-family: monospace; }
    #header { color: #555; font-size: 0.8rem; margin-bottom: 10px; display: flex; justify-content: space-between; }
    #status { color: #555; font-size: 0.8rem; }
    #note { flex: 1; width: 100%; background: #181818; color: #eee; border: 1px solid #333; padding: 16px; font-size: 1rem; font-family: monospace; resize: none; outline: none; line-height: 1.6; }
    #note:focus { border-color: #555; }
  </style>
</head>
<body>
  <div id="header">
    <span>Quick-note</span>
    <span id="status">saved</span>
  </div>
  <textarea id="note" placeholder="Start typing...">{{ content }}</textarea>

  <script>
    const note = document.getElementById('note');
    const status = document.getElementById('status');
    let saveTimer = null;

    // Auto-save 500ms after the user stops typing
    note.addEventListener('input', () => {
      status.textContent = 'saving...';
      clearTimeout(saveTimer);
      saveTimer = setTimeout(save, 500);
    });

    function save() {
      // POST the current content to the server
      fetch('/save', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ content: note.value })
      })
      .then(res => res.json())
      .then(data => {
        // Show saved confirmation, then fade back to subtle
        status.textContent = data.ok ? 'saved' : 'error';
      })
      .catch(() => { status.textContent = 'error'; });
    }
  </script>
</body>
</html>
"""

# ──────────────────────────────────────────────
# HELPERS
# ──────────────────────────────────────────────

def read_note() -> str:
    """Read the current note content from disk. Returns empty string if file doesn't exist."""
    try:
        with open(NOTE_FILE, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        return ""


def write_note(content: str) -> None:
    """Write note content to disk, overwriting previous content."""
    with open(NOTE_FILE, "w", encoding="utf-8") as f:
        f.write(content)


# ──────────────────────────────────────────────
# ROUTES
# ──────────────────────────────────────────────

@app.route("/")
def index():
    """Serve the note page with the current saved content loaded in."""
    content = read_note()
    return render_template_string(TEMPLATE, content=content)


@app.route("/save", methods=["POST"])
def save():
    """Receive note content from the browser and write it to disk."""
    data = request.get_json()
    if data and "content" in data:
        write_note(data["content"])
        return jsonify({"ok": True})
    return jsonify({"ok": False}), 400


# ──────────────────────────────────────────────
# ENTRY POINT
# ──────────────────────────────────────────────

if __name__ == "__main__":
    # 0.0.0.0 makes it accessible from other devices on the local network
    app.run(host="0.0.0.0", port=5500, debug=False)
```

The page loads the current `note.txt` content into the textarea. Every keystroke triggers a 500ms debounced `fetch` POST to `/save`, which writes the content to disk. On restart, the note loads right back from the file.

## Customization

* Change `NOTE_FILE` path to store the note somewhere else (e.g. a shared folder)
* Lower the debounce delay from `500` to `200` ms for faster saves
* Add multiple named notes by extending the route to `/save/<name>` and storing separate files
* Swap the dark theme colors in the `<style>` block to taste

## Limitations

* Single shared note — last write wins if two devices type at the same time
* No history or undo beyond the browser's own undo stack
* No authentication — anyone on your local network can read and overwrite it

## Future Improvements

* Multiple named note slots selectable from a sidebar
* Markdown preview toggle
* Timestamp of last save shown in the header

## Notes

* The note is stored as plain `note.txt` in the same folder — easy to read, back up, or grep
* Bookmark `http://<local-IP>:5500` on your phone for instant access
* Run in background: `nohup python3 Quick-note.py &`
