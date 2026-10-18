# Anna's Archive Calibre Store

A [Calibre](https://calibre-ebook.com/) store plugin for [Anna's Archive](https://en.wikipedia.org/wiki/Anna's_Archive).

> The largest truly open library in human history. Mirrors Sci-Hub and LibGen. Scrapes and open-sources Z-Lib, DuXiu, and more.

This is a fork of [scotjam/calibre_annas_archive](https://github.com/scotjam/calibre_annas_archive)
(itself a fork of [ScottBot10/calibre_annas_archive](https://github.com/ScottBot10/calibre_annas_archive))
with additional bug fixes, DNS-over-HTTPS fallback for ISP-level blocking, and CI releases.

> **The green arrow can be slow.** It fetches the book's page, resolves the real download link, and
> (the first time) clears the DDoS-Guard check in the background — this can take **a few tens of
> seconds**. To skip the wait, **double-click the book's title** to open it in the in-app browser
> and download directly.

## Installation

No extra prerequisites — the plugin uses calibre's bundled browser (QtWebEngine) for the DDoS-Guard
check and Python's standard library for everything else.

**From the release zip:**
1. Download `calibre_annas_archive-vX.X.X.zip` from the
   [Releases](https://github.com/seekermarcel/calibre_annas_archive/releases) page.
2. In calibre: **Preferences > Advanced > Plugins > Load plugin from file**, and select the zip.
3. **Restart calibre.** The store appears under **Get books**.

**From source:**
```shell
calibre-customize -b <path to cloned repo>
```
or build the zip:
```shell
./zip_release.sh && calibre-customize -a calibre_annas_archive-v*.zip
```
Restart calibre afterwards.

## Features

### Slow Partner Server downloads
`get_details` resolves the `/slow_download/` pages to the real partner-CDN file URL (e.g. on
`momot.rs`, which is not challenge-gated) so calibre's green-arrow download button works directly.
Servers are tried **#5 first** (no waitlist), then #6-#8, then the waitlisted #1-#4.

### DDoS-Guard auto-clearance
Search, detail and slow-download pages sit behind a DDoS-Guard JavaScript challenge that calibre's
JS-free downloader (mechanize) cannot clear. `clearance.py` runs a **minimised** in-process Chromium
window on the GUI thread that clears the challenge — usually with no interaction — and captures the
clearance cookies. Those cookies are re-injected into mechanize so subsequent pages resolve
directly. The clearance is reused until it expires; if an interactive captcha appears, the window is
surfaced for a one-time solve. Parallel lookups share a single check, so you get at most one window
per search. If a slow-download page is challenged during the background lookups anyway, that book
just skips the slow link (double-click it to use the manual browser below).

### Manual fallback browser
`slow_browser.py` opens Slow Server #5 in an in-process browser (double-click a result) that
auto-detects the "Download now" link, downloads the file, and adds it to your calibre library.
Used when auto-clearance is disabled or unavailable.

### Self-updating domains
Anna's Archive rotates domains frequently due to takedowns. The plugin pulls the current official
domains from the [Anna's Archive Wikipedia infobox](https://en.wikipedia.org/wiki/Anna's_Archive)
at runtime, falling back to a hardcoded domain list.

### DNS-over-HTTPS fallback
Some ISPs block Anna's Archive domains at the DNS level (returning NXDOMAIN). Before connecting to
mirrors, the plugin checks each hostname against system DNS and, on failure, resolves it via
[Cloudflare DNS-over-HTTPS](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/).
Only blocked domains are patched; all other DNS resolution is untouched. This is automatic — no
router or system configuration needed.

## Configuration

**Preferences > Plugins > Store**, then double-click **Anna's Archive** to open settings.

### Search options
Same search options as the site. For each checkbox group (filetype, language, etc.): if nothing is
checked it doesn't filter on that option; if anything is checked, only matching results are shown.

### Download link options
- **Direct download via Slow Partner Servers (#5 first):** resolve Anna's Archive's own slow servers
  to a direct file URL for the download button. Disable to use only external mirrors
  (Libgen/Z-Library).
- **If blocked, open Slow Server #5 in calibre's browser:** when a slow download is captcha-gated,
  open it in the embedded browser so you can solve the check and download in-app.
- **Auto-pass the DDoS-Guard check:** a minimised browser window opens briefly to clear the
  DDoS-Guard challenge automatically. Disable if it causes GPU/display issues on your system (see
  Troubleshooting below).
- **Verify Content-Type / Verify url extension:** validation applied to external mirror links.

### Mirrors
- **Auto-update domains from Wikipedia:** pull current official domains at runtime and try them first.
- The editable list is the fallback/ordering used when auto-update is off or unavailable.

## Troubleshooting

### GPU / display crashes on Linux
On some Linux systems, the in-process QtWebEngine (used for DDoS-Guard clearance) can crash the GPU
driver, which may take down your desktop environment. If this happens, disable these two options in
plugin settings:
- **Auto-pass the DDoS-Guard check**
- **If blocked, open Slow Server #5 in calibre's browser**

Downloads will then use external mirrors (Libgen, Z-Library) instead. Alternatively, try launching
calibre with software rendering: `LIBGL_ALWAYS_SOFTWARE=1 calibre`.

### No search results
If the plugin returns no results, your ISP may be DNS-blocking the Anna's Archive domains. As of
v0.3.4 the plugin automatically falls back to Cloudflare DNS-over-HTTPS, so this should resolve
itself. On older versions, switch your system or router DNS to a public resolver like `1.1.1.1`
(Cloudflare) or `8.8.8.8` (Google).

## Limitations

- The captured download URL is **time-limited** (the partner-CDN link carries an expiry token), so
  calibre should download it promptly — which it does on the green-arrow click.
- DDoS-Guard keys clearance on IP + User-Agent and the cookie expires after minutes to hours; the
  next attempt transparently re-clears via the minimised browser.
- Sometimes DDoS-Guard shows an interactive captcha instead of auto-passing; the minimised window is
  surfaced for a one-time solve.
- External mirrors (Libgen, Z-Library) can be temporarily unavailable (HTTP 503). Retry later or
  enable Slow Partner Servers for more reliable downloads.

## Credits

- [ScottBot10](https://github.com/ScottBot10/calibre_annas_archive) — original plugin (search, external mirrors, configuration UI)
- [scotjam](https://github.com/scotjam/calibre_annas_archive) — Slow Partner Server downloads, DDoS-Guard auto-clearance, Wikipedia domain updates, in-app slow browser
