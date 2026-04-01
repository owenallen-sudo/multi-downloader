# aria2dl

Minimal Bash wrapper for `aria2c` with persistent config, retries, directory crawling, multiple flags and custom flag aliases. It uses the Mozilla Public License v. 2.0.

---

## Usage

```bash
aria2dl <URL> [options]
```

Example:

```bash
aria2dl https://example.com/file.zip --streams=16 --download-limit=1M
```

---

## Core Flags

### General

* `--filepath=<path>`
  Output file or directory

* `--continue=true|false`, `-c`
  Resume download

* `--download-all-files=true|false`
  Force directory crawling

* `--file-allocation=<mode>`
  File allocation method

---

### Performance

* `--streams=<n>`
  Connections per download

* `--download-limit=<rate>`
  Max download speed (e.g. `750K`, `2M`)

* `--upload-limit=<rate>`
  Max upload speed

* `--timeout=<seconds>`
  Connection timeout

* `--uri-selector=<method>`
  URI selection strategy

---

### Retry

* `--max-script-attempts=<n>`
  Wrapper retry count

* `--max-tries-inside-aria2=<n>`
  aria2 retry count

* `--retry-inside-aria2-wait=<sec>`
  Delay between retries

---

### File Behaviour

* `--allow-overwrite=true|false`
  Overwrite existing files

* `--saving-interval=<sec>`
  Session save interval

---

### Network

* `--check-certificate=true|false`
  Verify SSL certificates

* `--http-accept-gzip=true|false`
  Enable gzip compression

---

### Notifications

* `--ntfy-enabled=true|false`
  Enable notifications

* `--ntfy-topic=<topic>`
  Notification topic

---

### Binary Paths

* `--curl-path=<path>`
* `--wget-path=<path>`
* `--aria2-path=<path>`

---

## URL Handling

Supports:

* http(s)
* ftp
* magnet

If the URL is a directory (or ends with `/`), files are crawled and downloaded recursively.

---

## Config

Stored at:

```
~/.config/aria2dl/config
```

Last used values are saved and reused.

---

## Custom Flags

Add shorthand aliases:

```bash
aria2dl addflag fast for --min-split-size=1M
```

Use:

```bash
aria2dl <URL> fast
```

Remove:

```bash
aria2dl removeflag fast
```

Stored in:

```
~/.config/aria2dl/flags
```

Format:

```
fast:--min-split-size=1M
```

---

## Dependencies

* aria2c
* curl
* wget
This project is licensed under the Mozilla Public License 2.0. Any modified version of my aria2 file that gets distributed must remain under this license and stay open-source due to legal rights. Please see the LICENSE file or https://mozilla.org/MPL/2.0/. for the full terms. Other users are free to modify and distribute this file if they follow the MPL 2.0 licensing agreements.
