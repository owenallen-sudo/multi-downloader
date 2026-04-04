# download

A Bash wrapper for downloading files using multiple tools with persistent config, retries, directory crawling, GitHub support, and custom flag aliases. It uses the Mozilla Public License V. 2.0.

---

## How It Works

By default the script picks the best tool automatically:

- **Surge** — used for `http`/`https` URLs (fastest for direct downloads)
- **Wget** — used for crawling URL folders to find all the filenames and structure without actually downloading.
- **aria2c** — used for everything else (`ftp`, `magnet`, `sftp`, `s3`, `gs`, `rsync`, folder crawling, GitHub API downloads, etc.)

You can override this with `--use-aria2`, `--use-curl`, `--use-wget`, `--use-surge`, or `--use-git` where it uses that one for downloading.

---

## Dependencies

Only one tool is strictly required, but installing all gives full functionality:

| Tool | Required? | Used For |
|------|-----------|----------|
| `aria2c` | Recommended | Non-HTTP downloads, folder crawling, GitHub private repos |
| `surge` | Recommended | Fast HTTP/HTTPS downloads (default for http/https) |
| `curl` | Recommended | Notifications, folder detection, `--use-curl` mode |
| `wget` | Recommended | Folder crawling, `--use-wget` mode |
| `git` | Optional | `--use-git` cloning of GitHub repos |
| `jq` | Optional | Syncing stream count to surge's config |

---

## Usage

```bash
download <URL> [options]
```

Example:
```bash
download https://example.com/file.zip --streams=16 --download-limit=1M
download https://github.com/user/repo --branch=dev --use-git
download magnet:?xt=... --filepath=~/Downloads/
```

---

## Flags

### Tool Selection

* `--use-surge`
  Force surge regardless of protocol (useful if surge adds new protocol support)
* `--use-aria2`
  Force aria2c instead of surge for http/https
* `--use-curl`
  Force curl
* `--use-wget`
  Force wget
* `--use-git`
  Clone GitHub repo with git instead of downloading a zipball

---

### General

* `--filepath=<path>`
  Output file or directory
* `--continue=true|false`, `-c`
  Resume download
* `--download-all-files=true|false`
  Force directory crawling (always uses aria2c)
* `--file-allocation=<mode>`
  File allocation method (aria2c only)

---

### Performance

* `--streams=<n>`
  Connections per download (aria2c; also synced to surge's `max_connections_per_host` in its config)
* `--download-limit=<rate>`
  Max download speed e.g. `750K`, `2M` (aria2c only)
* `--upload-limit=<rate>`
  Max upload speed (aria2c only)
* `--timeout=<seconds>`
  Connection timeout (aria2c only)
* `--uri-selector=<method>`
  URI selection strategy (aria2c only)
* `--http-accept-gzip=true|false`
  Enable gzip compression (aria2c only)

---

### Retry

* `--max-script-attempts=<n>`
  How many times the wrapper retries the download
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
  Personal access token for private repos (works with aria2c, curl, wget and git)
* `--branch=<branch>`
  Branch to download or clone (default: `main`)

GitHub URLs are automatically transformed to use the GitHub API zipball endpoint. With `--use-git`, the repo is cloned directly instead.

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

---

## URL Support

Supported protocols matched automatically:

- `http`, `https` → surge (default) or aria2c
- `ftp` → aria2c
- `magnet` → aria2c
- `sftp` → aria2c
- `s3`, `gs` → aria2c
- `rsync` → aria2c
- Any URL containing `://` → treated as a URL regardless of protocol

If the URL is a directory (ends with `/` or resolves to one), files are crawled and downloaded recursively using wget + aria2c.

GitHub URLs are automatically handled — private repos work with `--github-token`.

---

## Config

Stored at:
```
~/.config/aria2dl/config
```

Last used values for most settings are saved and reused on the next run. Tool paths, limits, retry counts, tokens, and notification settings all persist. Branch and per-download settings do not persist.

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

Note: custom flags are passed to aria2c only and are ignored when surge, curl, wget or git is used.

---

## License

This project is licensed under the Mozilla Public License 2.0. Any modified version of this file that gets distributed must remain under this license and stay open-source due to legal rights. Please see the LICENSE file or https://mozilla.org/MPL/2.0/ for the full terms. Other users are free to modify and distribute this file if they follow the MPL 2.0 licensing agreements.
