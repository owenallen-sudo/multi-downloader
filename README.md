# download
```
A Bash wrapper for downloading files using multiple tools with persistent config, retries, directory crawling, GitHub support, torrent support, zsync delta updates, and custom flag aliases. It uses the Mozilla Public License V. 2.0.
```
This project wasn't started for fun — it was built out of necessity.
My wifi connection comes from a 4G router with poor signal and no ability to add an external antenna. On a very good day it reaches approximately 40 Mbps download and 8 Mbps upload, but on a bad day it drops to 4–8 Mbps download and around 4 Mbps upload — and no matter how much optimisation I did, it remained unstable. Setting a custom MTU value of 1360, switching to native iwd for my Intel wifi card, forcing IPv4, and writing my own iwdwifi project as an nmtui alternative with no background scanning all helped to varying degrees, but instability was always there.
Before this script, I was running aria2c manually with 16+ flags that I had to keep written down in my phone's notes app — download limits, upload limits, retry counts, connection counts, and much more. Even with all of that, anything above 250MB would frequently fail overnight or take so many hours that I had no idea if it had finished or crashed. The ntfy push notification support exists precisely because of this — rather than checking back repeatedly or finding a failed download the next morning, the script notifies you the moment it succeeds or gives up.
This script was built to make downloads actually bearable on a slow, unstable connection. It hasn't had much real-world testing yet since it was only recently put together, but the combination of surge for HTTP/HTTPS (which handles connection instability significantly better than aria2c), per-tool speed limiting to reduce bufferbloat, automatic retries with session resumption, and tools like zsync for ISO updates should make a meaningful difference compared to typing out 16 flags from a notes app.
---

## How It Works

The script automatically selects the best tool based on the URL and flags:

- **Surge** — default for `http`/`https` URLs (fastest for direct downloads)
- **lftp** — automatically used for `ftp`/`sftp` URLs
- **transmission-remote** — automatically used for `magnet:` URLs and `.torrent` files
- **zsync** — automatically used when `--old-version` is passed (delta updates)
- **git** — default for non-http/https GitHub URLs (configurable via `--github-default`)
- **aria2c** — default for everything else, and fallback when no other tool applies

All `--use-*` flags override automatic detection for edge cases or personal preference.

---

## Dependencies

Only aria2c is strictly required, but installing all tools gives full functionality:

| Tool | Required? | Used For |
|------|-----------|----------|
| `aria2c` | Recommended | Default for most protocols, folder crawling, fallback |
| `surge` | Recommended | Fast HTTP/HTTPS downloads (default for http/https) |
| `curl` | Recommended | Notifications, folder URL detection, `--use-curl` mode |
| `wget` | Recommended | Folder crawling (spider mode), `--use-wget` mode |
| `lftp` | Recommended | FTP/SFTP downloads (auto-selected) |
| `transmission-remote` | Recommended | Torrent/magnet downloads (auto-selected) |
| `zsync` | Recommended | Delta updates for ISOs and large files (auto-selected) |
| `git` | Optional | GitHub repo cloning via `--use-git` or `--github-default=git` |
| `trickle` | Optional | Speed limiting for git (which has no native speed limit) |
| `jq` | Optional | Syncing connection count to surge's config |

---

## Usage

```bash
download <URL> [options]
```

Examples:
```bash
download https://example.com/file.zip --streams=16 --download-limit=1M
download https://github.com/user/repo --github-branch=dev --use-git
download magnet:?xt=... --filepath=~/Downloads/
download ftp://example.com/file.tar.gz
download https://distro.org/latest.iso --old-version=~/old.iso
download https://example.com/file.zip --use-transmission-remote
```

---

## Flags

### Tool Selection

* `--use-surge`
  Force surge regardless of protocol
* `--use-aria2`
  Force aria2c
* `--use-curl`
  Force curl
* `--use-wget`
  Force wget
* `--use-git`
  Clone GitHub repo with git instead of downloading a zipball
* `--use-lftp`
  Force lftp
* `--use-zsync`
  Force zsync
* `--use-transmission-remote`
  Force transmission-remote

---

### General

* `--filepath=<path>`
  Output file or directory
* `--continue=true|false`, `-c`
  Resume download
* `--download-all-files=true|false`
  Force directory crawling (uses wget + aria2c)
* `--file-allocation=<mode>`
  File allocation method (aria2c only)
* `--buffer-size=<size>`
  Buffer/chunk size synced to all tools that support it e.g. `4M`, `1M` (default: `4M`)

---

### Performance

* `--streams=<n>`
  Connections per download — synced to surge, lftp, and aria2c (default: `12`)
* `--download-limit=<rate>`
  Max download speed e.g. `750K`, `2M` — applies to aria2c, curl, wget, lftp, and git via trickle
* `--upload-limit=<rate>`
  Max upload speed — applies to aria2c, lftp, transmission-remote, and git via trickle
* `--timeout=<seconds>`
  Connection timeout (aria2c only)
* `--uri-selector=<method>`
  URI selection strategy (aria2c only)
* `--http-accept-gzip=true|false`
  Enable gzip compression (aria2c only)

---

### Speed Limits Per Tool

Each tool's speed limiting can be toggled independently:

* `--aria2-speed-limits=true|false`
* `--curl-speed-limits=true|false`
* `--wget-speed-limits=true|false`
* `--git-speed-limits=true|false` (uses trickle)
* `--lftp-speed-limits=true|false`
* `--transmission-speed-limits=true|false`

All default to `true`. Useful on very limited wifi to reduce bufferbloat by limiting specific tools.

---

### Retry

* `--max-script-attempts=<n>`
  How many times the wrapper retries the whole download (default: `3`)
* `--max-tries-inside-aria2=<n>`
  How many times aria2c retries internally (aria2c only)
* `--retry-inside-aria2-wait=<sec>`
  Delay between aria2c retries (aria2c only)

---

### File Behaviour

* `--allow-overwrite=true|false`
  Overwrite existing files (aria2c only)
* `--saving-interval=<sec>`
  Session save interval (aria2c only)

---

### Network

* `--check-certificate=true|false`
  Verify SSL certificates (aria2c only)

---

### GitHub

* `--github-token=<token>`
  Personal access token for private repos — works with aria2c, curl, wget, and git
* `--github-branch=<branch>`
  Branch to download or clone (default: `main`)
* `--github-default=<tool>`
  Default tool for non-http/https GitHub URLs. Options: `git`, `aria2c`, `aria2`, `curl`, `wget`, `lftp`, `transmission-remote` (default: `git`)
* `--depth=<n>`
  Shallow clone depth for git (omit for full history)

GitHub `http`/`https` URLs are always handled by surge by default via the GitHub API zipball endpoint. Non-http/https GitHub URLs default to git clone. Private repos work automatically when `--github-token` is set. If a repo has already been cloned, `git fetch` is used instead of `git clone` on subsequent runs.

---

### zsync

* `--update`
  Enable zsync update mode
* `--old-version=<path>`
  Path to the existing local file to diff against — automatically enables zsync

Useful for keeping Linux ISOs up to date without downloading the full image each time. Only works if the server provides a `.zsync` file alongside the download.

---

### Transmission

* `--transmission-host=<host:port>`
  Transmission daemon host (default: `localhost:9091`)
* `--transmission-auth=<user:pass>`
  Authentication for the Transmission daemon

---

### Notifications

* `--ntfy-enabled=true|false`
  Enable push notifications via ntfy.sh
* `--ntfy-topic=<topic>`
  Notification topic (default: `download_on_pc`)

---

### Binary Paths

* `--curl-path=<path>`
* `--wget-path=<path>`
* `--aria2-path=<path>`
* `--surge-path=<path>`
* `--git-path=<path>`
* `--lftp-path=<path>`
* `--zsync-path=<path>`
* `--transmission-path=<path>`

---

## URL Support

Protocols are matched and routed automatically:

| Protocol | Default Tool |
|----------|-------------|
| `http`, `https` | surge |
| `ftp`, `sftp` | lftp |
| `magnet:`, `.torrent` | transmission-remote |
| `s3`, `gs`, `rsync` | aria2c |
| GitHub http/https | surge (zipball via API) |
| GitHub non-http | git |
| Any `://` URL | treated as URL regardless of protocol |

Directory URLs (ending with `/` or resolving to one) are crawled recursively using wget + aria2c.

---

## Config

Stored at:
```
~/.config/aria2dl/config
```

Most settings persist between runs including tool paths, speed limits, retry counts, tokens, buffer size, stream count, and notification settings. The stream count is also automatically synced to surge's `~/.config/surge/settings.json` and lftp's `~/.lftp/rc` on every run.

---

## Custom Flags

Add shorthand aliases for any aria2c flag:

```bash
download addflag fast for --min-split-size=1M
```

Use:
```bash
download <URL> fast
```

Remove:
```bash
download removeflag fast
```

Stored in:
```
~/.config/aria2dl/flags
```

Format:
```
fast:--min-split-size=1M
```

Custom flags are passed to aria2c only and ignored by all other tools.

---

## License

This project is licensed under the Mozilla Public License 2.0. Any modified version of this file that gets distributed must remain under this license and stay open-source due to legal rights. Please see the LICENSE file or https://mozilla.org/MPL/2.0/ for the full terms. Other users are free to modify and distribute this file if they follow the MPL 2.0 licensing agreements.
