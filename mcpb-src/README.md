# Forum Pulse — Claude Desktop Extension

Parse Reddit AND Hacker News content from Claude.app with no OAuth or API key. Pass `source: 'reddit'` (default) or `source: 'hn'` on any of the main tools. Reddit routes through public JSON or Arctic Shift with optional Playwright live-verification; HN goes through Firebase + Algolia with deleted/dead flags read straight from the API.

## Tools exposed

- `search_reddit` — keyword search, subreddit-scoped (Reddit) or global. `source: 'hn'` searches HN via Algolia. Supports `with_context: N` to inline body + top comments per result.
- `get_post` — full body + nested top comments tree, by URL or bare id. Works for both sources.
- `get_subreddit_feed` — hot/new/top/rising listings. With `source: 'hn'`, also supports best/ask/show/job.
- `get_user_activity` — submissions, comments, or overview. Works for both sources.
- `get_subreddit_rules`, `get_subreddit_about`, `get_subreddit_wiki` — Reddit-only (HN is a single board with one [sitewide guidelines page](https://news.ycombinator.com/newsguidelines.html)).

## Requirements

- Python 3 on your machine. Claude Desktop will ask for the path during install (default `/usr/bin/python3`).
- Optional: `playwright` + `chromium` for `verify_live` ground-truth status checks:
  ```bash
  pip3 install --user playwright
  python3 -m playwright install chromium
  ```

## Install

1. Download / build `forum-pulse-0.6.1.mcpb` (or latest).
2. Open Claude.app → drag the `.mcpb` file into the Extensions area, or use the Extensions installer.
3. On the config screen, confirm your Python 3 path.

## Backends

- **Reddit** (`www.reddit.com/.../.json`) — authoritative, but blocks datacenter / VPN IPs and most bot-shaped requests in 2026.
- **Arctic Shift** (`arctic-shift.photon-reddit.com`) — community Reddit archive, near real-time, IP-block-resistant. Doesn't have subreddit rules / about / wiki, and its post snapshots don't update (so `archived`/`removed_by_category` can be stale — see `verify_live`).

`via: auto` picks Arctic Shift where it can, Reddit elsewhere, and falls back to Reddit on Arctic Shift failure.

## Liveness vs verify_live

Arctic Shift snapshots are frozen at post-creation time. Use:
- `liveness=exclude` to drop posts the heuristic flags as removed / deleted / archived / suspicious.
- `max_age_days=180` to also drop posts past Reddit's auto-archive window.
- `verify_live=all` (or `suspect`) to open each candidate URL in headless Chromium and ground-truth its actual status. Requires playwright.

## License

MIT.
