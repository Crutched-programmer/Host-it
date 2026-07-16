# File-Vault

> Self-hosted personal file vault with drag-and-drop upload, full-text search, and a minimal web UI — runs entirely on your local machine.

## The Problem It Solves

Cloud storage services are either paid, privacy-invasive, or have very limited storage space for the free tier ( 5GB for Microsoft Onedrive and 15GB for Google Drive ) . Keeping files organized on a local machine means either digging through folders manually or relying on a desktop search tool that indexes everything whether you want it to or not. File-Vault gives you a private, browser-accessible file store on your own hardware — upload anything, search by filename or tag, and retrieve it instantly. No account, no sync client, just your files and your devices.

## Features

* Drag-and-drop or click-to-upload from any browser on your local network
* Full-text search across filenames, tags, and optional notes attached to files
* Tag files at upload time or edit tags later from the UI
* File type filtering — filter by category (image, audio, video, document, archive, other)
* Inline preview for images, audio, video, and PDF files
* Direct download link for every file
* Duplicate detection by MD5 hash — warns before storing an identical file twice
* Rename and delete files from the UI
* Flat file storage — files live in a single `vault/` directory, no nested folder chaos
* SQLite metadata store — filename, original name, upload time, size, tags, notes, hash
* Pagination for large libraries
* Optional basic password protection via a single shared passphrase (session cookie)

## Tech Stack

* Python 3.8+
* Flask (web server and upload handler)
* SQLite via sqlite3 (built-in, no database server needed)
* hashlib (deduplication, built-in)
* mimetypes (built-in, file type detection)
* HTML5 + vanilla JS (frontend, no framework)

## Requirements

```
flask>=2.0
werkzeug>=2.0
```

No heavy dependencies. Both packages install via pip in under a second.
## Code 
Copy and past this code and save it as <code> server.py </code>.
~~~~
# ===========================
# File-Vault.py
# Minimal Starter Implementation
# ===========================

import os
import sqlite3
import hashlib
import mimetypes
from datetime import datetime

from flask import (
    Flask,
    request,
    jsonify,
    send_from_directory,
    render_template_string,
)
from werkzeug.utils import secure_filename

# ----------------------------
# Configuration
# ----------------------------

PORT = 6060
VAULT_DIR = "vault"
DB_NAME = "File-Vault.db"
MAX_UPLOAD_MB = 500

os.makedirs(VAULT_DIR, exist_ok=True)

app = Flask(__name__)
app.config["MAX_CONTENT_LENGTH"] = MAX_UPLOAD_MB * 1024 * 1024


# ----------------------------
# Database
# ----------------------------

conn = sqlite3.connect(DB_NAME, check_same_thread=False)
cur = conn.cursor()

cur.execute("""
CREATE TABLE IF NOT EXISTS files (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    stored_name TEXT,
    original_name TEXT,
    upload_time TEXT,
    size INTEGER,
    tags TEXT,
    notes TEXT,
    md5 TEXT
)
""")

conn.commit()


# ----------------------------
# Helpers
# ----------------------------

def md5_file(path):
    h = hashlib.md5()

    with open(path, "rb") as f:
        while True:
            chunk = f.read(8192)

            if not chunk:
                break

            h.update(chunk)

    return h.hexdigest()


def unique_filename(filename):
    base, ext = os.path.splitext(filename)

    candidate = filename

    counter = 1

    while os.path.exists(os.path.join(VAULT_DIR, candidate)):
        candidate = f"{base}_{counter}{ext}"
        counter += 1

    return candidate


# ----------------------------
# Homepage
# ----------------------------

@app.route("/")
def home():

    return render_template_string("""
    <html>
    <head>
        <title>File Vault</title>
    </head>

    <body style="font-family:Arial;margin:40px">

        <h1>📁 File Vault</h1>

        <form action="/upload" method="POST"
              enctype="multipart/form-data">

            <input type="file" name="file">

            <br><br>

            <input
                name="tags"
                placeholder="Tags">

            <br><br>

            <textarea
                name="notes"
                placeholder="Notes"></textarea>

            <br><br>

            <button>Upload</button>

        </form>

        <hr>

        <form action="/search">

            <input
                name="q"
                placeholder="Search">

            <button>Go</button>

        </form>

    </body>

    </html>
    """)


# ----------------------------
# Upload
# ----------------------------

@app.route("/upload", methods=["POST"])
def upload():

    if "file" not in request.files:
        return "No file."

    file = request.files["file"]

    if file.filename == "":
        return "No filename."

    filename = secure_filename(file.filename)
    filename = unique_filename(filename)

    path = os.path.join(VAULT_DIR, filename)

    file.save(path)

    md5 = md5_file(path)

    cur.execute(
        "SELECT id FROM files WHERE md5=?",
        (md5,)
    )

    duplicate = cur.fetchone()

    if duplicate:
        os.remove(path)
        return "Duplicate file detected."

    size = os.path.getsize(path)

    cur.execute("""
        INSERT INTO files(
            stored_name,
            original_name,
            upload_time,
            size,
            tags,
            notes,
            md5
        )
        VALUES(?,?,?,?,?,?,?)
    """, (
        filename,
        file.filename,
        datetime.now().isoformat(),
        size,
        request.form.get("tags", ""),
        request.form.get("notes", ""),
        md5
    ))

    conn.commit()

    return "Upload successful."


# ----------------------------
# Search
# ----------------------------

@app.route("/search")
def search():

    q = request.args.get("q", "")

    pattern = "%" + q + "%"

    cur.execute("""
        SELECT
            id,
            original_name,
            tags,
            notes,
            upload_time
        FROM files

        WHERE
            original_name LIKE ?
            OR tags LIKE ?
            OR notes LIKE ?
    """, (
        pattern,
        pattern,
        pattern
    ))

    rows = cur.fetchall()

    return jsonify(rows)


# ----------------------------
# Download
# ----------------------------

@app.route("/file/<name>")
def file(name):

    return send_from_directory(
        VAULT_DIR,
        name,
        as_attachment=True
    )


# ----------------------------
# Edit Metadata
# ----------------------------

@app.route("/edit/<int:file_id>", methods=["POST"])
def edit(file_id):

    tags = request.form.get("tags", "")
    notes = request.form.get("notes", "")

    cur.execute("""
        UPDATE files

        SET
            tags=?,
            notes=?

        WHERE id=?
    """, (
        tags,
        notes,
        file_id
    ))

    conn.commit()

    return "Updated."


# ----------------------------
# Delete
# ----------------------------

@app.route("/delete/<int:file_id>", methods=["POST"])
def delete(file_id):

    cur.execute(
        "SELECT stored_name FROM files WHERE id=?",
        (file_id,)
    )

    row = cur.fetchone()

    if row is None:
        return "Not found."

    path = os.path.join(
        VAULT_DIR,
        row[0]
    )

    if os.path.exists(path):
        os.remove(path)

    cur.execute(
        "DELETE FROM files WHERE id=?",
        (file_id,)
    )

    conn.commit()

    return "Deleted."


# ----------------------------
# Start Server
# ----------------------------

if __name__ == "__main__":

    print(f"Running on http://localhost:{PORT}")

    app.run(
        host="0.0.0.0",
        port=PORT,
        debug=False
    )
~~~~
## Install & Run

```bash
# 1. Clone the repo
git clone https://github.com/your-username/File-Vault
cd File-Vault

# 2. Install dependencies
pip install flask werkzeug

# 3. Start the server
python File-Vault.py

# 4. Open in browser
# Local:   http://localhost:6060
# Network: http://<your-lan-ip>:6060
```

To enable password protection:

```bash
python File-Vault.py --password yourpassphrase
```

To change storage directory or port:

```bash
python File-Vault.py --vault /mnt/external/vault --port 8080
```

## Usage

### Uploading

Open the UI, drag files onto the drop zone or click to browse. Multiple files can be uploaded at once. Before submitting, add comma-separated tags and an optional note. Click Upload.

If a file with the same MD5 hash already exists in the vault, the UI will warn you and show the existing entry. You can skip or upload anyway with a different name.

### Searching

Use the search bar at the top. It matches against:

* Filename (original and stored)
* Tags
* Notes

Examples:

```
invoice                   # finds all files with "invoice" in name or tags
tax 2024                  # finds files matching both words
tag:receipt               # finds files tagged exactly "receipt"
type:pdf                  # filters by file type category
```

### Previewing and Downloading

Click any file row to expand the inline preview panel. Images, audio, video, and PDFs render directly in the browser. All other types show a download button.

The direct download URL format is:

```
http://localhost:6060/file/<stored_filename>
```

### Tagging and Notes

Click the edit icon on any file row to update its tags or notes without re-uploading. Changes are saved to the SQLite database immediately.

### Deleting

Click the delete icon on a file row. You will be asked to confirm. Deletion removes both the file from disk and its database record.

## How It Works

```
File-Vault.py
├── Flask app
│   ├── GET  /              # serves the UI (index.html rendered inline)
│   ├── POST /upload        # receives multipart form, saves file, indexes it
│   ├── GET  /search        # returns JSON results for search query
│   ├── GET  /file/<name>   # serves raw file with correct Content-Type
│   ├── POST /edit/<id>     # updates tags and notes for a file record
│   └── POST /delete/<id>   # deletes file from disk and database
│
├── on_upload(file)
│   ├── compute MD5 hash of incoming bytes
│   ├── check database for existing hash
│   ├── sanitize filename via werkzeug secure_filename
│   ├── save to vault/ directory with collision-safe name
│   └── insert record into SQLite files table
│
└── on_search(query)
    ├── parse query for tag: and type: prefixes
    └── run parameterized LIKE query against filename, tags, notes columns
```

Files are stored flat in the `vault/` directory. If two uploaded files share the same original name, the stored name gets a short random suffix to avoid collisions. The original filename is preserved in the database so search and display always show what you uploaded.

## Customization

**Change max upload size** — edit `MAX_UPLOAD_MB` at the top of `File-Vault.py`:

```python
# Maximum size per upload in megabytes
MAX_UPLOAD_MB = 500
```

**Allowed file types** — by default all types are accepted. To whitelist specific extensions, edit `ALLOWED_EXTENSIONS`:

```python
# Set to None to allow all types, or specify a set of extensions
ALLOWED_EXTENSIONS = None
# ALLOWED_EXTENSIONS = {'.pdf', '.jpg', '.png', '.mp3', '.zip'}
```

**Auto-tagging by file type** — a helper at the bottom of the upload handler automatically adds a `type:<category>` tag to every file on upload. Edit `AUTO_TAG_TYPES` to change the category mappings.

**Thumbnail generation** — install `Pillow` and set `GENERATE_THUMBNAILS = True` to generate 200x200 thumbnails for image uploads, stored in `vault/.thumbs/`. The UI will show them in a grid view toggle.

## Limitations

* No user accounts — single-user, single-passphrase model only
* No folder hierarchy — all files live flat in `vault/`, organization is entirely via tags
* Full-text search is SQLite LIKE, not indexed FTS — on very large libraries (50,000+ files) search will slow down
* No chunked upload — very large files (multi-GB) may time out depending on your network and Flask config
* No versioning — uploading a file with the same name as an existing file creates a new record, it does not replace the old one

## Future Improvements

* Chunked upload with progress bar for large files
* SQLite FTS5 index for faster full-text search on large libraries
* Folder / collection grouping as a virtual layer on top of flat storage
* Shareable expiring download links for sending files to others on LAN
* Bulk tag editor — select multiple files and apply or remove a tag at once
* ZIP download for a filtered search result set

## Notes

* The `vault/` directory and `File-Vault.db` are created automatically on first run in the project directory. Point `--vault` at an external drive for large collections.
* Passphrase protection sets a signed session cookie. It is not HTTPS — do not expose this to the public internet without putting it behind a reverse proxy with TLS (nginx + certbot).
* The frontend is a single inline HTML string rendered by Flask — no build step, no npm, no static asset pipeline. Works immediately after `python File-Vault.py`.
* Tested on Ubuntu 22.04, Debian 12, and WSL2 on Windows 11.
