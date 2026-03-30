===FILE: README.md===
# LocalPodcastIndexer
> A lightweight tool to index and search local podcasts.

## The Problem It Solves
Local podcast files are often scattered across various directories, making it difficult to find and listen to them. LocalPodcastIndexer allows you to quickly index your podcast library and search for episodes by title or keyword.

## Features
- Indexes all MP3 and M4A podcast files in specified directories.
- Provides a simple text-based menu for searching and playing podcasts.
- Uses SQLite for efficient storage and retrieval of podcast metadata.
- Minimal resource usage, designed to run on old PCs with limited resources.

## Tech Stack
- Python 3.6+
- SQLite

## Requirements
- Linux with Python 3.6 or later installed.
- `mpg123` for audio playback (install via package manager).

## Install & Run
1. Clone the repository:
   ```bash
   git clone https://github.com/yourusername/localpodcastindexer.git
   cd localpodcastindexer
   ```
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Configure directories to scan in `config.json`.
4. Run the indexer:
   ```bash
   python index.py
   ```
5. Start the search and play interface:
   ```bash
   python search.py
   ```

## Folder Structure
```
localpodcastindexer/
├── README.md
├── config.json
├── db.sqlite3
├── index.py
├── search.py
├── requirements.txt
└── utils.py
```

## Notes / Assumptions
- The script assumes that podcast files are in MP3 or M4A format.
- Audio playback uses `mpg123`, which must be installed on the system.

===FILE: config.json===
```json
{
  "directories": [
    "/path/to/podcasts/dir1",
    "/path/to/podcasts/dir2"
  ]
}
```

===FILE: index.py===
```python
import os
import json
import sqlite3
from mutagen.easyid3 import EasyID3
from mutagen.mp4 import MP4, MP4Cover

# Load configuration
with open('config.json', 'r') as config_file:
    config = json.load(config_file)

# Initialize database
conn = sqlite3.connect('db.sqlite3')
c = conn.cursor()
c.execute('''CREATE TABLE IF NOT EXISTS podcasts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT,
                artist TEXT,
                album TEXT,
                filepath TEXT
            )''')

def index_podcast(file_path):
    try:
        if file_path.endswith('.mp3'):
            audio = EasyID3(file_path)
        elif file_path.endswith('.m4a'):
            audio = MP4(file_path)
        else:
            return
        
        title = audio.get('title', ['Unknown'])[0]
        artist = audio.get('artist', ['Unknown'])[0]
        album = audio.get('album', ['Unknown'])[0]
        
        c.execute("INSERT INTO podcasts (title, artist, album, filepath) VALUES (?, ?, ?, ?)",
                  (title, artist, album, file_path))
    except Exception as e:
        print(f"Error processing {file_path}: {e}")

# Clear existing data
c.execute('DELETE FROM podcasts')

# Index all podcast files in directories
for directory in config['directories']:
    for root, _, files in os.walk(directory):
        for file in files:
            if file.lower().endswith(('.mp3', '.m4a')):
                index_podcast(os.path.join(root, file))

conn.commit()
conn.close()

print("Indexing complete.")
```

===FILE: search.py===
```python
import sqlite3

# Initialize database
conn = sqlite3.connect('db.sqlite3')
c = conn.cursor()

def search_podcasts(query):
    c.execute("SELECT id, title, artist, album FROM podcasts WHERE title LIKE ? OR artist LIKE ?", ('%' + query + '%', '%' + query + '%'))
    return c.fetchall()

def play_podcast(file_path):
    os.system(f"mpg123 {file_path}")

while True:
    print("\nPodcast Search and Play")
    print("------------------------")
    query = input("Enter search term (or 'exit' to quit): ").strip()
    
    if query.lower() == 'exit':
        break
    
    results = search_podcasts(query)
    
    if not results:
        print("No podcasts found.")
        continue
    
    for idx, (id, title, artist, album) in enumerate(results):
        print(f"{idx + 1}. {title} by {artist} ({album})")
    
    choice = input("Choose a podcast to play (or 'back' to search again): ").strip()
    
    if choice.lower() == 'back':
        continue
    
    try:
        choice_idx = int(choice) - 1
        selected_podcast = results[choice_idx]
        play_podcast(selected_podcast[3])
    except (ValueError, IndexError):
        print("Invalid choice. Please try again.")

conn.close()
```

===FILE: requirements.txt===
```
mutagen==1.45.1
sqlite3
```

This solution provides a lightweight tool to index and search local podcasts, making it easy to find and play episodes on old PCs with limited resources.
