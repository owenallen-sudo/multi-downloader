# aria2 Download Manager Wrapper

A powerful bash wrapper around [aria2c](https://aria2.github.io/) that simplifies downloading files and entire directory structures while preserving folder hierarchies. Features persistent configuration, custom flag shortcuts, automatic retry logic, and desktop notifications via ntfy.sh.

## Overview

This script provides a user-friendly interface to aria2c with advanced features:
- **Automatic URL detection**: Distinguishes between single files and folder URLs
- **Persistent settings**: Saves your preferences to `~/.config/aria2dl/config`
- **Custom flag system**: Create shortcuts for frequently-used flag combinations
- **Intelligent retry logic**: Automatically retries failed downloads (up to 3 times by default)
- **Desktop notifications**: Real-time status updates via ntfy.sh
- **Session recovery**: Resume interrupted downloads seamlessly
- **Bandwidth management**: Fine-grained control over download/upload speeds
- **Folder hierarchy preservation**: Download entire directories while maintaining structure

---

## Requirements

The script requires three external utilities to function. **You must have them installed** before using this wrapper:

### Required Dependencies

| Utility | Purpose | Version |
|---------|---------|---------|
| **aria2c** | Core download manager; handles multi-threaded downloads with resume support | Any recent version |
| **curl** | URL inspection, HTTP head requests, and ntfy.sh notification delivery | 7.0+ |
| **wget** | Website crawling with `--spider` mode to discover all files in a directory | 1.12+ |

The script will fail immediately if any of these tools are missing. Verify installation:

```bash
which aria2c curl wget
```

All three should output valid paths. If any are missing, you'll see a command not found error when running the script.

---

## Configuration & Defaults

The script creates a configuration directory at `~/.config/aria2dl/` containing:

- **config** – Persistent settings (automatically created after first run)
- **flags** – Custom flag definitions

All default values are applied on first run and can be overridden via command-line flags or by editing the config file directly.

### Default Settings

```bash
FILEPATH="$HOME"                    # Download to home directory by default
CHECK_CERT=true                     # Verify SSL/TLS certificates
DOWNLOAD_LIMIT="750K"              # Max download speed: 750 Kilobytes/sec
UPLOAD_LIMIT="1M"                  # Max upload speed: 1 Megabyte/sec
STREAMS=12                          # Number of parallel connections
MAX_SCRIPT_TRIES=3                  # Retry failed downloads up to 3 times
CONTINUE=false                      # Don't resume by default
URI_SELECTOR=feedback               # Intelligent connection selection
FILE_ALLOCATION=falloc              # Fast file pre-allocation
SAVING_INTERVAL=5                   # Save session every 5 seconds
ALLOW_OVERWRITE=true                # Overwrite existing files
TIMEOUT=600                         # 10-minute connection timeout
RETRY_WAIT=8                        # 8-second wait between aria2c retries
MAX_TRIES_INSIDE_ARIA2C=600         # aria2c internal retry limit
NTFY_ENABLED=true                   # Send notifications by default
NTFY_TOPIC="download_on_pc"         # Default ntfy.sh topic
COMPRESSION=true                    # Accept gzip compression
```

---

## Command-Line Flags Reference

All flags can be combined and are saved for future use. The script persists these settings, so you only need to specify them once.

### Download Location

**`--filepath=PATH`**
- **Type**: Directory or file path
- **Default**: `$HOME`
- **Examples**:
  - `--filepath=/mnt/external/downloads` – Download to external drive
  - `--filepath=/tmp/video.mp4` – Specify exact output filename
  - `--filepath=/media/usb/` – Download to USB (trailing slash = directory)
- **Behavior**: If PATH is a directory, the script preserves the URL filename. If PATH is a file, it uses that exact name.

### Performance Tuning

**`--streams=N`**
- **Type**: Integer (number of parallel connections)
- **Default**: `12`
- **Range**: 1-32 (higher = more bandwidth usage but faster)
- **Examples**:
  - `--streams=1` – Single connection (slow but stable on weak networks)
  - `--streams=8` – Balanced (good for most home connections)
  - `--streams=32` – Maximum (for datacenter/CDN downloads)
- **Impact**: Each stream opens a separate TCP connection to the server. More streams = faster but higher CPU/memory usage.

**`--download-limit=RATE`**
- **Type**: Speed in bytes/sec with unit suffix
- **Default**: `750K`
- **Format**: `[number][K|M|G]` where K=1024 bytes, M=1MB, G=1GB
- **Examples**:
  - `--download-limit=100K` – Limit to 100 KB/sec (mobile-friendly)
  - `--download-limit=5M` – Limit to 5 MB/sec (gigabit network)
  - `--download-limit=10M` – Datacenter speed
- **Use case**: Prevent network saturation while downloading, leaving bandwidth for other tasks.

**`--upload-limit=RATE`**
- **Type**: Speed in bytes/sec with unit suffix
- **Default**: `1M`
- **Format**: Same as `--download-limit`
- **Examples**:
  - `--upload-limit=512K` – Restrict upload when using BitTorrent/p2p
  - `--upload-limit=5M` – Full gigabit upload
- **Use case**: Relevant only when downloading from sources that upload (BitTorrent, cloud sync).

### Retry & Recovery

**`--max-script-attempts=N`**
- **Type**: Integer (number of retry attempts)
- **Default**: `3`
- **Range**: 1-10
- **Examples**:
  - `--max-script-attempts=1` – No retries; fail immediately
  - `--max-script-attempts=5` – Retry 5 times (good for unstable networks)
  - `--max-script-attempts=10` – Maximum resilience
- **Behavior**: If aria2c exits with an error, the wrapper script waits 10 seconds and tries again from the last checkpoint (if using `-c` flag).
- **Log**: Each attempt is timestamped in `aria2.log`.

**`--continue` or `-c`**
- **Type**: Boolean flag
- **Default**: `false` (start fresh)
- **Behavior**: 
  - When enabled: Resume downloads from the last checkpoint using `aria2.session`
  - When disabled: Delete the session file before each download (start from 0%)
- **Use case**: Enable this when retrying failed downloads to avoid re-downloading.
- **Example**: `aria2 -c https://example.com/largefile.iso`

**`--retry-inside-aria2-wait=SECS`**
- **Type**: Integer (seconds)
- **Default**: `8`
- **Range**: 1-60
- **Examples**:
  - `--retry-inside-aria2-wait=2` – Quick retries (aggressive)
  - `--retry-inside-aria2-wait=15` – Slow retries (server-friendly)
- **Behavior**: When aria2c encounters a connection failure, it waits this many seconds before retrying internally.
- **Use case**: Increase on servers that rate-limit or temporarily block connections.

**`--max-tries-inside-aria2=N`**
- **Type**: Integer (retry count)
- **Default**: `600`
- **Range**: 1-1000
- **Examples**:
  - `--max-tries-inside-aria2=10` – Limited retries (fail fast)
  - `--max-tries-inside-aria2=600` – Maximum retries (very patient)
- **Behavior**: Maximum number of times aria2c will retry a single failed connection before giving up.
- **Interaction**: Works in conjunction with `--retry-inside-aria2-wait`. Total wait time ≈ retries × wait_seconds.

### Certificate & Security

**`--check-certificate=BOOL`**
- **Type**: `true` or `false`
- **Default**: `true`
- **Examples**:
  - `--check-certificate=true` – Verify SSL (secure, default)
  - `--check-certificate=false` – Skip SSL verification (use only for trusted, self-signed sources)
- **Warning**: Setting to `false` disables HTTPS security checks. Only use on trusted networks.

**`--http-accept-gzip=BOOL`**
- **Type**: `true` or `false`
- **Default**: `true`
- **Behavior**: Accept gzip-compressed responses from servers, reducing bandwidth at cost of CPU.
- **Examples**:
  - `--http-accept-gzip=true` – Compress for bandwidth savings
  - `--http-accept-gzip=false` – Disable on low-power systems

### File Handling

**`--file-allocation=METHOD`**
- **Type**: `falloc`, `prealloc`, or `none`
- **Default**: `falloc`
- **Differences**:
  - `falloc` – Fast fallocate system call (Linux); pre-allocates space instantly
  - `prealloc` – Slower but compatible with all filesystems; writes zeros to pre-allocate
  - `none` – No pre-allocation; files grow as data arrives (slower on HDDs)
- **Examples**:
  - `--file-allocation=falloc` – Modern Linux SSD
  - `--file-allocation=prealloc` – Older systems or network drives
  - `--file-allocation=none` – Embedded systems with tight I/O

**`--allow-overwrite=BOOL`**
- **Type**: `true` or `false`
- **Default**: `true`
- **Behavior**:
  - `true` – Overwrite existing files silently
  - `false` – Fail if file exists (safe for important data)
- **Examples**:
  - `--allow-overwrite=true` – Replace old versions
  - `--allow-overwrite=false` – Protect against accidental overwrites

**`--uri-selector=METHOD`**
- **Type**: `feedback`, `adaptive`, or `inorder`
- **Default**: `feedback`
- **Algorithms**:
  - `feedback` – Switch to fastest mirrors based on real-time performance (recommended)
  - `adaptive` – Gradual switching; less aggressive than feedback
  - `inorder` – Stick with the first URI (no switching)
- **Use case**: `feedback` mode is best for multi-mirror downloads; `inorder` for single sources.

### Timeout Control

**`--timeout=SECS`**
- **Type**: Integer (seconds)
- **Default**: `600` (10 minutes)
- **Range**: 10-3600
- **Examples**:
  - `--timeout=30` – Fail quickly on slow connections
  - `--timeout=300` – 5-minute timeout (normal networks)
  - `--timeout=1800` – 30-minute timeout (very slow/unreliable networks)
- **Behavior**: If no data is received for this duration, the connection is dropped and retried.

**`--saving-interval=SECS`**
- **Type**: Integer (seconds)
- **Default**: `5`
- **Range**: 1-60
- **Behavior**: Session file is updated this frequently, allowing recovery if the process crashes.
- **Impact**: Lower = more disk I/O; higher = less frequent recovery points.

### Tool Paths

**`--aria2-path=PATH`**
- **Type**: Full path to aria2c binary
- **Default**: `/usr/bin/aria2c`
- **Use**: Override if aria2c is installed in a non-standard location
- **Example**: `--aria2-path=/usr/local/bin/aria2c` or `--aria2-path=/opt/aria2c`

**`--curl-path=PATH`**
- **Type**: Full path to curl binary
- **Default**: `/usr/bin/curl`
- **Use**: Override if curl is in a custom location
- **Example**: `--curl-path=/usr/local/bin/curl`

**`--wget-path=PATH`**
- **Type**: Full path to wget binary
- **Default**: `/usr/bin/wget`
- **Use**: Override if wget is in a custom location
- **Example**: `--wget-path=/opt/wget/bin/wget`

---

## Notifications via ntfy.sh

The script integrates with [ntfy.sh](https://ntfy.sh/) to send real-time download status notifications to your phone or browser.

### How It Works

1. **Download starts** – Script sends "Download started" notification
2. **Download fails** – Retry notification sent with exit code
3. **Download completes** – Success notification with file location
4. **Final retry fails** – Failure notification with log file location

### Notification Flags

**`--ntfy-enabled=BOOL`**
- **Type**: `true` or `false`
- **Default**: `true`
- **Examples**:
  - `--ntfy-enabled=true` – Send notifications (default)
  - `--ntfy-enabled=false` – Suppress all notifications

**`--ntfy-topic=TOPIC`**
- **Type**: String (alphanumeric + hyphens)
- **Default**: `download_on_pc`
- **Examples**:
  - `--ntfy-topic=my_downloads` – Custom topic
  - `--ntfy-topic=john_large_files` – User-specific topic
- **Important**: Topics are public by default! Choose an unpredictable name (e.g., `downloads_abc123xyz`).
- **Privacy note**: Do not use sensitive information in topic names.

### Receiving Notifications

**On your phone:**
1. Download ntfy app (iOS: "Notifire" or "Ntfy"; Android: "ntfy")
2. Subscribe to your topic (e.g., `download_on_pc`)
3. Notifications appear in real-time

**In your browser:**
1. Visit `https://ntfy.sh/your_topic_name`
2. New notifications appear instantly on the page
3. Refresh to see history

### Notification Examples

```
Title: Download complete
Body: ubuntu-22.04.iso → /home/user/Downloads

---

Title: Failed (exit code 256), check /home/user/Downloads/aria2.log
Body: largefile.zip

---

Title: Attempt 2/3 failed (exit code 256)
Body: database-backup.tar.gz
```

---

## Retry Logic Explained

The script implements a two-level retry system:

### Script-Level Retries (Outer Loop)

Controlled by `--max-script-attempts`:
- If aria2c fails with any exit code, the wrapper waits **10 seconds**
- Then attempts the entire download again up to N times
- Each attempt is logged to `aria2.log` with timestamp
- On each failure, a notification is sent (if enabled)

**Example with default settings:**

```
Attempt 1: aria2c fails with exit code 1
  → Wait 10 seconds
  → Retry notification sent to ntfy

Attempt 2: aria2c fails with exit code 256
  → Wait 10 seconds
  → Retry notification sent to ntfy

Attempt 3: aria2c succeeds
  → Download complete notification sent
  → Script exits with status 0
```

### aria2c-Level Retries (Inner Loop)

Controlled by `--max-tries-inside-aria2` and `--retry-inside-aria2-wait`:
- aria2c itself retries individual connections up to N times
- Waits M seconds between each retry
- Handles temporary network glitches without wrapper intervention

**Default behavior:**
- aria2c retries each connection up to 600 times
- Waits 8 seconds between retries
- Maximum wait time: 600 × 8 = 4800 seconds (80 minutes) per connection

### Combining Retry Strategies

```bash
# Conservative: Fail fast
aria2 --max-script-attempts=2 --max-tries-inside-aria2=10 --retry-inside-aria2-wait=2 https://...

# Aggressive: Very patient (good for unstable networks)
aria2 --max-script-attempts=5 --max-tries-inside-aria2=1000 --retry-inside-aria2-wait=15 https://...

# Balanced (default)
aria2 --max-script-attempts=3 --max-tries-inside-aria2=600 --retry-inside-aria2-wait=8 https://...
```

---

## Folder Download Mode

When given a folder URL, the script automatically:

1. **Detects the URL type** – Checks if URL ends in `/` or resolves to a folder
2. **Crawls with wget** – Uses `wget --spider -r` to discover all files
3. **Preserves structure** – Maps relative paths from the source to your download directory
4. **Downloads all files** – Uses aria2c with per-file directory options

### How to Use

```bash
# Download all files from a directory
aria2 https://mirrors.example.com/ubuntu/releases/

# Result: Files downloaded to $HOME with preserved subdirectories
# $HOME/
# ├── 22.04/
# │   ├── ubuntu-22.04-live.iso
# │   └── SHA256SUMS
# ├── 20.04/
# │   ├── ubuntu-20.04-live.iso
# │   └── SHA256SUMS
```

### Override Folder Detection

If auto-detection fails:

```bash
# Force folder mode
aria2 --download-all-files=true https://example.com/files

# Force single-file mode
aria2 --download-all-files=false https://example.com/files
```

---

## Custom Flag Shortcuts

Create shortcuts for frequently-used flag combinations:

### Add a Shortcut

```bash
aria2 addflag fast for --streams=20 --download-limit=5M

aria2 addflag torrent for --file-allocation=prealloc --max-script-attempts=5

aria2 addflag cloud for --timeout=1800 --max-tries-inside-aria2=1000
```

### Use a Shortcut

```bash
aria2 fast https://example.com/file.zip
# Equivalent to: aria2 --streams=20 --download-limit=5M https://example.com/file.zip

aria2 cloud --filepath=/mnt/external/ https://google-drive-mirror.com/backup.tar.gz
# Shortcuts can be combined with other flags
```

### List and Remove Shortcuts

```bash
cat ~/.config/aria2dl/flags
# Output:
# fast:--streams=20 --download-limit=5M
# torrent:--file-allocation=prealloc --max-script-attempts=5
# cloud:--timeout=1800 --max-tries-inside-aria2=1000

aria2 removeflag fast
# Removes the 'fast' shortcut
```

---

## Output Files

After each download, the following files are created in the target directory:

| File | Purpose |
|------|---------|
| `aria2.session` | Checkpoint file for resuming interrupted downloads; deleted if not using `-c` flag |
| `aria2.log` | Full transcript of all download attempts with timestamps, commands, and exit codes |
| `[downloaded files]` | Your actual download(s) |

### Log File Example

```
=== 2026-03-30 15:23:45 ===
Running: /usr/bin/aria2c -x 12 -s 12 --max-tries=600 --retry-wait=8 --timeout=600 --check-certificate=true --save-session=/home/user/aria2.session --save-session-interval=5 --allow-overwrite=true --file-allocation=falloc --max-overall-download-limit=750K --max-overall-upload-limit=1M --uri-selector=feedback --dir=/home/user -o ubuntu.iso https://mirrors.ubuntu.com/iso/ubuntu-22.04-live.iso
[NF] Initializing ECC...
[NOTICE] Downloading 2 file(s)
[#abc123 0B/4.5GB(0%) CN:1 DL:512KB ETA:2h24m]

=== 2026-03-30 17:48:12 ===
[ATTEMPT 1/3] Failed with exit code 1.
Retrying in 10 seconds

=== 2026-03-30 17:48:25 ===
Running: /usr/bin/aria2c -x 12 -s 12 ... [CONTINUE]
```

---

## Examples

### Basic Download
```bash
aria2 https://example.com/file.zip
# Downloads to $HOME/file.zip with default settings
```

### Download with Custom Speed
```bash
aria2 --download-limit=2M --streams=8 https://example.com/large.iso
# Limits speed to 2 MB/sec with 8 parallel connections
```

### Download to Specific Location
```bash
aria2 --filepath=/mnt/external/backup.tar.gz https://backup.company.com/latest.tar.gz
# Downloads to /mnt/external with filename backup.tar.gz
```

### Resume Failed Download
```bash
aria2 -c https://example.com/file.iso
# Resumes from last checkpoint; retries up to 3 times
```

### Download Folder with Custom Settings
```bash
aria2 --filepath=/media/usb/data --streams=16 --ntfy-topic=usb_backup \
  https://archive.example.com/datasets/
# Downloads all files to /media/usb/data, uses 16 streams, sends notifications to 'usb_backup' topic
```

### Aggressive Retry for Unstable Networks
```bash
aria2 --max-script-attempts=5 --max-tries-inside-aria2=1000 --retry-inside-aria2-wait=15 \
  https://example.com/file.zip
# Will retry aggressively with long wait times between attempts
```

### Create & Use Shortcut
```bash
aria2 addflag slow for --streams=4 --download-limit=500K --max-script-attempts=5

aria2 slow https://example.com/file.zip
# Uses the "slow" preset: 4 streams, 500K limit, 5 retries
```

---

## Configuration Files

```
~/.config/aria2dl/
├── config          # Auto-generated; stores last-used settings
└── flags           # Custom flag shortcuts (format: SHORTCUT:FLAGS)

~/.config/aria2dl/config example:
NTFY_TOPIC="download_on_pc"
STREAMS="12"
DOWNLOAD_LIMIT="750K"
UPLOAD_LIMIT="1M"
FILE_ALLOCATION="falloc"
TIMEOUT="600"
...

~/.config/aria2dl/flags example:
fast:--streams=20 --download-limit=5M
torrent:--file-allocation=prealloc --max-script-attempts=5
```

---

## Troubleshooting

### "Command not found: aria2c / curl / wget"
**Solution**: All three utilities must be installed. Verify with `which aria2c curl wget`. Each should output a path.

### Download fails on first attempt but succeeds on retry
**Reason**: Temporary network glitch; normal behavior. Script retries automatically.

**Check**: Review `aria2.log` in the download directory to see the error code.

### Notifications not appearing
**Check 1**: Is `--ntfy-enabled=true`? (default)
```bash
aria2 --ntfy-enabled=true https://example.com/file.zip
```

**Check 2**: Is ntfy.sh reachable?
```bash
curl https://ntfy.sh/test_topic -d "test"
```

**Check 3**: Visit `https://ntfy.sh/your_topic_name` in a browser to verify messages are arriving.

### Slow download speeds
**Try**: Increase streams and/or bandwidth limit
```bash
aria2 --streams=32 --download-limit=10M https://example.com/file.zip
```

**Check**: Your actual internet speed with `speedtest-cli` or similar.

### Download hangs on folder crawling
**Reason**: Server might not be responding to `wget --spider` requests or is timing out.

**Solution**: Force single-file mode
```bash
aria2 --download-all-files=false https://example.com/files/index.html
```

### "Flag already exists" error
**Reason**: You tried to add a shortcut with a name that exists.

**Solution**: Remove first, then re-add
```bash
aria2 removeflag fast
aria2 addflag fast for --streams=20 --download-limit=10M
```

### SSL certificate verification failed
**Reason**: Server has self-signed or expired certificate (or MITM attack risk).

**Solution** (if trusted):
```bash
aria2 --check-certificate=false https://untrusted.example.com/file.zip
```

**Warning**: Only use on networks you trust!

---

## License

Use freely. Contributions welcome!