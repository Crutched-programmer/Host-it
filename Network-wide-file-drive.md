# Network-wide-file-drive

> Lightweight bidirectional file sync daemon for syncing folders between machines on a local network without cloud services.

## The Problem It Solves

Syncing files between your old repurposed PC and a laptop requires either cloud storage or manual rsync commands. Network-wide-file-drive watches a folder, detects changes, and automatically pushes or pulls updates to/from remote machines via SSH, keeping your working directories in sync with minimal overhead.

## Features

* Real-time folder monitoring with inotify
* Bidirectional sync (push changes to remote, pull remote changes)
* SSH-based transport (no central server needed)
* Conflict resolution (newer file wins, with backup of older)
* Selective sync with exclude patterns (ignore node_modules, .git, etc.)
* Dry-run mode to preview changes before sync
* Simple TOML config for multiple sync pairs
* Runs as a background daemon with systemd support
* Network error recovery with exponential backoff

## Tech Stack

* Python 3.7+
* inotify (file monitoring via python-inotify)
* SSH (rsync under the hood)
* TOML (configuration)
* systemd (optional, for daemon management)

## Requirements

* Linux or WSL
* Python 3.7+
* SSH access to remote machine (key-based auth recommended)
* rsync installed on both local and remote machines
* python-inotify: `pip install inotify-simple`

## Install & Run

```bash
# Clone or download the project
git clone https://github.com/yourusername/Network-wide-file-drive
cd Network-wide-file-drive

# Install dependencies
pip install inotify-simple pyyaml

# Create config file
cp config.example.toml config.toml

# Edit config with your sync pairs
nano config.toml

# Start the daemon
python Network-wide-file-drive.py start

# Check status
python Network-wide-file-drive.py status

# View sync logs
python Network-wide-file-drive.py logs --lines 20

# Stop the daemon
python Network-wide-file-drive.py stop
```

## Usage

**Create config file (config.toml):**

```toml
[sync_pair_1]
local_path = "/home/user/projects"
remote_host = "192.168.1.100"
remote_user = "user"
remote_path = "/home/user/projects"
direction = "bidirectional"  # or "push" or "pull"
exclude_patterns = [".git", "node_modules", "__pycache__", "*.tmp"]
conflict_resolution = "newer"  # or "local" or "remote"

[sync_pair_2]
local_path = "/home/user/documents"
remote_host = "laptop.local"
remote_user = "user"
remote_path = "/home/user/docs"
direction = "push"
exclude_patterns = [".cache"]
```

**Start monitoring:**

```bash
python Network-wide-file-drive.py start
```

**Dry-run (preview changes without syncing):**

```bash
python Network-wide-file-drive.py sync --dry-run
```

**Force full sync:**

```bash
python Network-wide-file-drive.py sync --full
```

**Check daemon status:**

```bash
python Network-wide-file-drive.py status
```

**View recent sync activity:**

```bash
python Network-wide-file-drive.py logs --lines 50
```

## How It Works

### Monitoring & Detection

Network-wide-file-drive uses inotify to watch for file system events (create, modify, delete) on the local folder. When changes are detected, they're queued and batched every 5 seconds to avoid syncing on every keystroke.

### Sync Process

1. **Event Detection:** inotify catches file changes in watched folder
2. **Batching:** Changes queue up, processed every 5 seconds
3. **Exclusion Check:** Files matching exclude patterns (*.log, node_modules) are skipped
4. **Dry-run:** Generate list of changes without transferring
5. **Conflict Check:** Compare timestamps and sizes between local and remote
6. **Sync:** Use rsync with SSH to transfer only changed blocks
7. **Logging:** Record all syncs with timestamp and file count
8. **Retry:** Failed syncs retry with exponential backoff (2s → 4s → 8s → 30s)

### SSH & Rsync Transport

```bash
# Behind the scenes, Network-wide-file-drive runs:
rsync -avz --delete \
  --exclude=.git \
  --exclude=node_modules \
  /local/path/ \
  user@remote:/remote/path/
```

Uses SSH keys for passwordless auth. Config points to key location.

### Conflict Resolution Strategies

* **Newer:** File with latest modification time wins (default)
* **Local:** Always keep local version, overwrite remote
* **Remote:** Always keep remote version, overwrite local
* **Backup:** Keep both, rename older to `.backup.TIMESTAMP`

## Customization

**Change Monitoring Interval:** Edit `BATCH_DELAY` in `Network-wide-file-drive.py` (default: 5 seconds).

**Add Custom Sync Rules:** Extend the config file with additional `[sync_pair_N]` sections, each with unique paths and settings.

**Use Different SSH Key:** Set `ssh_key_path` in config:
```toml
ssh_key_path = "/home/user/.ssh/id_macropad"
```

**Disable Sync for Specific Extensions:**
```toml
exclude_patterns = [".git", "*.log", "*.tmp", ".cache", "node_modules"]
```

**Change Conflict Behavior:** Set `conflict_resolution` per sync pair.

**Bandwidth Limiting:** Add rsync bandwidth limit:
```toml
rsync_args = "--bwlimit=1000"  # 1000 KB/s
```

## Limitations

* One-way sync pairs not bidirectional (but can define separate push/pull configs)
* Deletes propagate immediately (no trash/recycle bin)
* Network latency can cause race conditions in rapid bidirectional changes
* Requires SSH access (no SMB/NFS support in base version)
* inotify only works on Linux (not macOS native; requires polling on WSL)
* Large files can block the daemon during transfer

## Future Improvements

* Web UI dashboard showing sync status and history
* Bandwidth throttling with QoS controls
* Delta sync (only transfer changed blocks, not entire files)
* File versioning with automatic retention policies
* Encryption for sync data in transit and at rest
* Mobile app for push notifications on sync events
* SMB/NFS backend support for non-SSH setups
* Incremental backups with deduplication
* Conflict resolution UI for user-facing decisions

## Notes

* Always test new configs with `--dry-run` first
* Keep SSH keys secure and use key-based auth (no passwords in config)
* Backups of conflicted files stored as `.backup.TIMESTAMP`
* Sync logs rotate daily, kept for 30 days
* First sync can take time on large folders; subsequent syncs are incremental
* Works best on LANs; WAN syncing requires VPN for security
* Exclude `.git` folders unless you specifically want to sync git repos
