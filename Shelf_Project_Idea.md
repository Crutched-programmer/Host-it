# Shelf

**Your personal knowledge warehouse. Self-hosted. Fast. Offline-first.**

## MVP (v1.0)

-   Index one or more folders
-   Instant search by filename
-   PDF preview
-   Image preview
-   Video thumbnail
-   ZIP file information
-   Tags
-   Favorites
-   Recently added
-   File statistics
-   Dark mode

## v1.5

-   OCR for scanned PDFs
-   Search inside text and Markdown files
-   Duplicate file detection
-   Broken shortcut detection
-   Large-file finder
-   Automatic rescans

## v2.0

-   Optional local AI semantic search
-   Related files
-   Natural language search
-   PDF summaries
-   Collections

## Tech Stack

-   Backend: Python + FastAPI
-   Frontend: NiceGUI
-   Database: SQLite
-   Search: Whoosh
-   Scheduler: APScheduler

## Project Structure

``` text
Shelf/
├── app/
├── database/
├── indexer/
├── previews/
├── search/
├── static/
├── templates/
├── uploads/
├── config/
├── logs/
├── docs/
├── requirements.txt
└── main.py
```

## Future Plugins

-   GitHub repository browser
-   KiCad project viewer
-   Arduino sketch viewer
-   STL viewer
-   STEP viewer
-   Music player
-   Ebook reader

## Why Shelf?

A lightweight, privacy-first self-hosted document library that helps
users organize and instantly find datasheets, notes, PDFs, CAD files,
firmware, and downloads without relying on cloud services.
