---
name: leverage-finder
description: Find specific named humans on the internet who can USE, BUY, BENEFIT FROM, or AMPLIFY an early-stage project. Combines the eager-buyer-finder candidate-profile approach with Forum Pulse (Reddit + Hacker News) for community power users and WebSearch for newsletter / podcast / YouTube creator rosters. Outputs a tiered deck split into FOUNDER PEERS to interview, BUYERS who'd actually pay, AMPLIFIERS who'd help spread (newsletters, podcasts, niche YouTubers, Reddit/HN power users, accessibility advocates, language-immersion creators, competitor-switcher communities), and DISCOVERY CHANNELS to submit to. Use when the user asks "find customers / users / leverage people for my project", "who can help spread X", "find amplifiers / influencers / my ICP / my first users", "where do I post X", "how do I market my open source project", or pastes a README and asks for an outreach list.
---

# leverage-finder

A composite workflow for finding *named individuals* on the internet who can help an early-stage project — across buyers, beneficiaries, and amplifiers — in one ranked deck. Built on top of Forum Pulse (Reddit + HN) and WebSearch.

## What this skill is for

The user has a project (URL, README, or description) and wants specific *people* to reach out to. Not market segments, not personas — actual humans with handles, channels, and a way to contact them.

Three intents are common; this skill handles all three together:

| Intent | What the user calls it | What you're finding |
|---|---|---|
| **Buyers** | "find customers", "find my first users", "ICP" | Humans whose public behavior signals they'd USE or PAY for this |
| **Beneficiaries** | "who could benefit", "who could use this" | Humans in adjacent niches who'd quietly love this but aren't shopping yet |
| **Amplifiers** | "people who can spread it", "help me know more people", "leverage" | Newsletter writers, podcast hosts, niche YouTubers, Reddit/HN power users, Discord/Slack runners, mods, recognized voices |

Default behavior: produce **all three layers in one deck**. The user can ask to focus on just one.

## Workflow

### Step 1 — Profile the project

From the user's README / description, extract:

1. **Core capability** — one sentence on what it does.
2. **Differentiators** — the things ONLY this does. (Pull from a competitor-comparison table if there is one.)
3. **Stated audiences** — copy the "who is this for" list verbatim if it exists.
4. **Adjacent products / competitors** — whose users would switch?
5. **Wedge niches** — small audiences where the differentiator is *uniquely* valuable (accessibility, language immersion, ADHD, RSI, hard-of-hearing, niche hobbyists). These are usually under-claimed and high-converting.
6. **Stage signal** — star count, version, age. <100 stars / <0.2.0 / <3mo = "needs early adopters, not mass market." This biases toward founder-peer + niche-newsletter outreach over broad YouTubers.

Render a quick candidate profile so the user can correct anything before you spend tool calls.

### Step 2 — Pick categories

Run through this table and select what's relevant to the project. Don't run all of them — pick the 4–7 with the strongest fit. Recording why a category was skipped is fine.

| Category | Where they live | Scan rule |
|---|---|---|
| **Founder peers (Tier S — interview, not pitch)** | r/macapps, r/SaaS, r/indiehackers, HN Show HN, Lobsters | "Posted similar product in past year. Active in same subreddit. Open-source preferred if your project is." |
| **AI-tools discovery channels** | TLDR AI, Ben's Bites, Future Tools, TAAFT, Ahead of AI, Interconnects, Import AI, The Batch, Superhuman AI, The Rundown AI, Simon Willison's Weblog | WebSearch round-ups, then cross-reference with project's specific angle (novelty? open source? agentic?). |
| **Reddit power users in target subreddits** | Subreddit-by-subreddit Forum Pulse search | Commenters/posters who responded substantively in the last year on threads about your competitors or methodology. |
| **HN regulars discussing competitors** | Forum Pulse HN search for each competitor name | Posters with substantive comments, especially "I switched from X to Y because..." pattern. |
| **Niche YouTubers** | WebSearch + manual cross-check | "top {niche} YouTubers {current_year}" — language learning, Mac power user, Apple ecosystem, AI agents, productivity, accessibility, sales coaching. |
| **Niche podcast hosts** | Spotify / Apple Podcasts via WebSearch | "top {niche} podcasts {current_year}". |
| **Newsletter writers in the niche** | WebSearch | "best {niche} newsletter {current_year}". Specifically: founder/indie-hacker, AI tools, accessibility, language. |
| **Accessibility advocates** | r/Blind, r/Accessibility, r/RSI, r/disability, Apple Disability community | Underserved + high evangelism — if the project has *any* accessibility wedge, prioritize this. Usually the cleanest wedge for products that genuinely help. |
| **Language-immersion creators** | r/LanguageLearning, r/learnjapanese, r/Korean, r/ChineseLanguage, AJATT / Refold / MIA / TheMoeWay communities | Only if product genuinely supports their workflow (real-time translation, ambient listening, transcript analysis). |
| **ADHD / body-doubling community** | r/ADHD, r/getdisciplined, ADHD-specific Twitter | Only if product offers a body-doubling or voice-first thinking angle. |
| **Sales/CS thought leaders** | LinkedIn, r/sales, r/customersuccess | Only if product has a live-meeting or coaching angle. |
| **Competitor-switcher threads** | Forum Pulse search for "alternative to {competitor}" | The author + 50 commenters of a thread complaining about a competitor are a pre-qualified shortlist. |

### Step 3 — Execute searches in parallel

For each selected category, decide which tool(s) to use:

#### Forum Pulse (Reddit/HN) searches
Use `mcp__Forum_Pulse__search_reddit` (or the equivalent skill command). Defaults:
- `liveness: "exclude"`, `verify_live: "all"`, `max_age_days: 180` (the engageability defaults)
- `with_context: 2` or `3` so you can read the body + top comments inline
- `t: "year"` for recency
- `limit: 6` to keep it manageable

For Reddit, ALWAYS pass `subreddit` — global Reddit text-search is bot-blocked. For global cross-community searches, use `source: "hn"` (HN supports global search natively).

Multiple subreddits → multiple parallel calls. Don't serialize them.

#### WebSearch queries
Use the WebSearch tool. Phrasing patterns that work:
- `"best {niche} newsletter {current_year}"` — newsletter rosters
- `"top {niche} YouTubers {current_year}"` — YouTube creators
- `"{competitor} alternative review {current_year}"` — competitor-switcher reviewers
- `"{niche} influencer Twitter list {current_year}"` — X/Twitter handles (the round-ups are public even when X isn't)

#### Get-user-activity follow-ups
For any Tier S/A candidate from Reddit, run `mcp__Forum_Pulse__get_user_activity` with `what: "overview"` to vet their actual posting history before adding them to the shortlist. A user with one anomalous post isn't a real lead.

### Step 4 — Score by leverage

For each candidate, mentally compute:

```
Leverage ≈ AudienceSize × RelevanceFit × Engageability × Recency
```

- **AudienceSize**: subscribers/followers/comment count on their best post. Order-of-magnitude is enough; don't obsess.
- **RelevanceFit**: how close their public output is to the project's exact wedge. A Mac-power-user creator is high-fit for a Mac project; a generic-tech YouTuber is low.
- **Engageability**: is the candidate active, the thread live, the user not shadowbanned? Forum Pulse's verify_live handles this for Reddit/HN; for newsletters/YouTubers, check recent post dates.
- **Recency**: when did they last engage with this niche? Drop dormant accounts.

Don't try to compute exact scores. Bucket into Tier S / A / B.

### Step 5 — Output as one combined deck

Default structure:

```markdown
# Project — leverage deck

> One-line restatement of project + stage so the user can correct.

## 🟢 Tier S — Founder peers (collaborate, don't pitch)
Treat as peers. Open with respect, share your project, suggest a comparison conversation.
| Person | Project | Why they matter | How to reach |

## 🟢 Tier S — Discovery channels (submit / tip)
Submit, tip line, or DM the editor. Lead with your single biggest novelty.
| Channel | Audience | Submission path |

## 🟢 Tier S — Active shoppers (immediate buyers)
Verified-live threads where someone is asking for exactly what you have.
| User | Thread (live URL) | Quote of intent |

## 🟡 Tier A — Reddit / HN power users (respond, don't cold-DM)
Reply in their existing thread by name, *not* a cold DM.
| Username | Last engaged on | Pitch hook |

## 🟡 Tier A — Niche communities to post in
Where you should post your Show HN / r/X launch, in priority order.

## 🟠 Tier B — Outside-Reddit/HN amplifiers
Newsletter / podcast / YouTube candidates from WebSearch.
Flag which need a sanity-check before outreach.

## What's missing from this pass
Always include this section. Be honest about which categories you skipped and which need a second pass.
```

### Step 6 — Offer a deeper second pass

End every output with the top 2–3 categories that would benefit from a focused second pass, ordered by recommendation. Don't fish — just name them.

## Important rules

- **Every Reddit/HN candidate must have a live, verified thread URL.** No exceptions. If the thread isn't live, drop the candidate.
- **Every WebSearch-sourced name must have a roster URL** in the Sources section so the user can spot-check.
- **Don't invent names.** If you don't have a real candidate, say "needs WebSearch pass for {niche} creators" rather than guessing.
- **Pre-warn about WebSearch listicle decay.** Round-ups from earlier in the year may be stale. Suggest the user spot-check 2–3 names from each list before mass-outreach.
- **Founder peers are NOT amplifiers.** Don't pitch them; suggest a peer conversation. Mark this in the deck.
- **For very early projects (<100 stars), bias toward niche newsletter + accessibility + competitor-switcher categories.** Mass-market YouTubers won't return value at this stage.

## Tools required

- `mcp__Forum_Pulse__search_reddit` — Reddit and HN searches
- `mcp__Forum_Pulse__get_user_activity` — vetting Reddit/HN power users
- `mcp__Forum_Pulse__get_subreddit_feed` — surveying a subreddit's current activity before deciding to post
- `WebSearch` — newsletter / podcast / YouTuber rosters
- (Optional) `anthropic-skills:eager-buyer-finder` — if installed, use it as the candidate-profile generator for buyer hunts specifically

## Anti-patterns

- Don't dump 50 names. Tier them and cap each tier at ~5–10 actionable picks.
- Don't conflate intent. Founder peers, buyers, and amplifiers go in *separate sections*.
- Don't recommend X/Twitter API hunts. Free X scraping is dead; tell the user to hand-build a list from who their existing Tier S follows.
- Don't surface dead threads. `verify_live: "all"` is non-negotiable.
- Don't list a category just because the table lists it. Skip categories that don't fit the project.
