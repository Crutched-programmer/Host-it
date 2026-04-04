# File-Vault

> Self-hosted personal file vault with drag-and-drop upload, full-text search, and a minimal web UI — runs entirely on your local machine.

## The Problem It Solves

Cloud storage services are either paid, privacy-invasive, or have very limited storage space for the free tier ( 5GB for Microsoft Onedrive and 15GB for Google Drive ) . Keeping files organized on a local machine means either digging through folders manually or relying on a desktop search tool that indexes everything whether you want it to or not. File-Vault gives you a private, browser-accessible file store on your own hardware — upload anything, search by filename or tag, and retrieve it instantly. No account, no sync client, no internet required.

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
