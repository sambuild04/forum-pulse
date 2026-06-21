---
name: forum-pulse
description: Parse Reddit AND Hacker News content with no auth and no API key. Pass `--source reddit` (default) or `--source hn` to switch platforms. Reddit goes through public .json endpoints or Arctic Shift (Pushshift successor, real-time, IP-block-resistant) automatically and verifies live post status via headless Chromium. HN goes through Firebase + Algolia and reads deleted/dead flags straight from the API. Handles search, single posts with comments, subreddit/frontpage feeds, user activity, and (Reddit-only) subreddit rules, sidebar, and wiki. Outputs markdown summaries with absolute (UTC) and relative timestamps on every post and comment. Use when the user pastes a reddit.com or news.ycombinator.com URL, mentions an `r/<sub>` or `u/<user>` or an HN handle, or asks things like "what are the rules of r/X", "summarize r/X", "what's hot on HN", "parse this reddit post", "what's on the HN frontpage", "show me top comments on this thread", "what did u/X post recently", "search HN for X", or "search reddit for X".
---

# forum-pulse

Parses Reddit and Hacker News content. Pick the source with `--source {reddit,hn}` (default: `reddit`). Reddit uses two backends with automatic routing:

- **Reddit's `.json` endpoints** (`www.reddit.com`): authoritative, supports everything, but blocks datacenter/sandbox/VPN IPs and rate-limits aggressively.
- **Arctic Shift** (`arctic-shift.photon-reddit.com`): community-run real-time Reddit archive. Works from blocked IPs, no auth, but only covers posts and comments — not subreddit metadata, wiki, or Reddit-computed feed rankings (hot/top/rising).

Default mode is `--via auto`: the skill picks Arctic Shift where it can, Reddit where it must, and falls back to Reddit if Arctic Shift fails. All commands stream markdown to stdout.

## Quick start

```bash
# Subreddit rules
python3 scripts/reddit.py rules askscience

# Subreddit metadata + sidebar
python3 scripts/reddit.py about askscience

# Subreddit wiki (defaults to index)
python3 scripts/reddit.py wiki askscience
python3 scripts/reddit.py wiki askscience rules

# Feed
python3 scripts/reddit.py feed askscience --sort top --t week --limit 10

# A specific post (full URL or just the post id)
python3 scripts/reddit.py post https://www.reddit.com/r/askscience/comments/abc123/title/
python3 scripts/reddit.py post abc123 --limit 15 --depth 2

# User
python3 scripts/reddit.py user spez --what overview --limit 20

# Search
python3 scripts/reddit.py search "black hole" --sub askscience --limit 10
python3 scripts/reddit.py search "founder mode" --sort top --t year

# Exclude archived posts (good for buyer/lead research — archived = no new comments)
python3 scripts/reddit.py search "tired of" --sub LearnJapanese --t year --archived exclude
python3 scripts/reddit.py feed askscience --sort top --t year --archived exclude
python3 scripts/reddit.py user spez --what submitted --archived exclude
```

## Picking the right command

| User says | Use |
|---|---|
| "rules of r/X", "what are r/X's rules" | `rules X` |
| "what is r/X", "summarize r/X" | `about X` (often plus `feed X --sort hot`) |
| "what's hot/new/top on r/X" | `feed X --sort hot\|new\|top\|rising` (add `--t week` for top) |
| pastes a reddit post URL | `post <url>` |
| "what did u/X post" | `user X --what submitted` |
| "what did u/X comment" | `user X --what comments` |
| "what has u/X been up to" | `user X --what overview` |
| "search reddit for Y" | `search "Y"` (add `--sub Z` if subreddit-scoped) |

`r/foo`, `/r/foo`, and `foo` are all accepted as subreddit names. Same for users (`u/`, `/user/`, bare).

## Output format

Markdown. Every post and every comment shows the date both ways: `2024-03-15 14:22 UTC (3d ago)`. Long bodies are truncated with `…` to keep output scannable — if the user needs the full body, re-run with a higher `--limit` or fetch the post directly.

## Backend routing (`--via`)

`feed`, `post`, `user`, `search` accept `--via {reddit,arctic-shift,auto}` (default `auto`):

| Command | `auto` picks | Why |
|---|---|---|
| `search` | arctic-shift | Live data, IP-block resistant |
| `post` | arctic-shift | Same — and posts/comments live in Arctic Shift's archive |
| `user` | arctic-shift | Profile (karma/created date) is skipped under arctic-shift since it's not in the archive |
| `feed --sort new` | arctic-shift | New = chronological, which Arctic Shift supports natively |
| `feed --sort hot/top/rising` | reddit | Those rankings are computed by Reddit, not raw data |
| `rules`, `about`, `wiki` | reddit only | Arctic Shift doesn't archive subreddit metadata |

If Arctic Shift fails (network error, missing post, etc.) the skill falls back to Reddit automatically when `--via auto`. Force a specific backend with `--via reddit` or `--via arctic-shift`.

The output header annotates which backend served the request (`— via arctic-shift` or `_(via reddit)_`).

## Liveness heuristic (`--liveness`, `--max-age-days`)

Arctic Shift snapshots are taken at post-creation time and never updated, so a post that was later deleted, mod-removed, or auto-archived still appears "live" in the data. The skill compensates with heuristics:

- `selftext == "[removed]"` → status `removed`
- `selftext == "[deleted]"` or `author == "[deleted]"` → status `deleted`
- Posted >`--max-age-days` ago → status `likely_archived` (default Reddit archive window is ~180 days)
- `score == 0` with >5 comments → status `suspicious` (mod-removal pattern: comments stay but vote-count zeroes)

`--liveness` modes:
- `include` (default) — show everything, no filter or annotation
- `exclude` — drop non-live posts entirely. Best for buyer-research lead lists.
- `only` — show only non-live posts. Debugging.
- `flag` — show everything, annotate non-live with `` `[status]` `` inline. Best when you want to see your data with confidence labels.

Example:
```bash
# Lead research: drop dead posts AND anything likely-archived
python3 scripts/reddit.py search "tired of anki" --sub LearnJapanese --t year \
    --liveness exclude --max-age-days 180

# Audit: show all results with status annotations
python3 scripts/reddit.py search "alternative" --sub LearnJapanese --t year \
    --liveness flag --max-age-days 180
```

**Heuristics aren't ground truth.** A post deleted by the OP keeps its title and score; a mod-removed post may have a non-zero score. To actually confirm status, the skill can open each suspect URL in a real headless browser.

## Live verification (`--verify-live`)

`--verify-live {none,suspect,all}` opens posts in a headless Chromium (Playwright) and parses the page for the actual deletion / removal / archive banners. Overwrites the heuristic `_liveness` with ground truth.

- `none` (default) — no browser, heuristic only.
- `suspect` — verify only posts the heuristic flagged. Fast (~3s per suspect).
- `all` — verify every result. Slow (~3s per post × N). Use this when correctness matters more than time, e.g. handing a buyer-research shortlist to a human.

**Setup:**
```bash
pip3 install --user playwright
python3 -m playwright install chromium
```

**Example:**
```bash
# Ground-truth ICP hunt — heuristic prefilters, Playwright verifies everything
python3 scripts/reddit.py search "tired" --sub LearnJapanese --t year \
    --liveness exclude --max-age-days 180 --verify-live all
```

This is the only path that catches OP-deleted posts that retain title + comments (which the heuristic can't see in Arctic Shift snapshots). Reddit's `.json` endpoint blocks scripted clients in 2026 even with TLS-impersonation libraries like `curl_cffi`, so a real browser is required.

## Archived filter

`feed`, `search`, and `user` accept `--archived {include,exclude,only}` (default `include`). Reddit auto-archives posts ~6 months old, which locks comments. Use `--archived exclude` for lead/buyer research — archived posts can't be engaged with, so they're noise. `post` always shows a `📦 Archived` marker in the header when the thread is archived; listing commands tag archived items with `📦`.

## Rate limiting

Reddit's unauthenticated endpoints allow roughly 10 requests/minute per IP. On HTTP 429 the script exits with a clear error — surface it to the user and wait, don't loop. If the user hits 429 repeatedly, suggest spacing out requests.

## Caveats

- **HTTP 403 from every endpoint** usually means Reddit is blocking the *requesting IP* (common on datacenter, VPN, CI, or sandbox IPs), not that the sub is private. Retry from a normal residential connection. If 403s persist there too, the sub is genuinely private/banned/quarantined — or it's time to switch to OAuth.
- Deleted users/posts return 404.
- The `post` command takes a URL or a bare post id (e.g. `abc123`). It returns top `--limit` comments at `--depth` nesting (defaults 20 / 2). Bigger numbers can balloon output.
- The User-Agent header is set to `reddit-parser-skill/0.x` (kept for historical continuity) — Reddit blocks the default Python UA, so don't strip it if you edit the script.
- The `.json` endpoints don't expose every field of the official API (e.g. no modmail, no private wiki). For that you'd need OAuth, which this skill deliberately avoids.
