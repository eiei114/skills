---
name: reddit-research
description: "Research Reddit communities using the old.reddit.com JSON API (curl + jq). Use this skill whenever the user wants to gather information FROM Reddit — posts, comments, subreddit stats, user demographics, sentiment, complaints, or community analysis. PROACTIVELY activate on any request to investigate, analyze, research, or explore Reddit opinions, discussions, or community feedback — even if the user does not explicitly say 'Reddit research'. Triggers include: 'search Reddit for', 'what does Reddit say about', 'check Reddit posts', 'Reddit sentiment', 'community analysis', 'subreddit investigation', 'user opinions on Reddit', 'Redditで調べて', 'Redditの評判', or any request about understanding audience reactions on Reddit. Works across all languages."
---

# Reddit Research via JSON API

Reddit's `old.reddit.com` endpoint serves raw JSON — no browser, no Cloudflare, no auth. This skill provides a structured workflow to mine posts, comments, subreddit stats, and user behavior patterns from any topic.

## Why this approach

- `reddit.com` blocks headless browsers and WebFetch tools
- `old.reddit.com/.json` returns clean, parseable data via `curl`
- Works from any terminal — no browser, no API key, no OAuth
- Rate-limit friendly if you keep requests under ~1/second

## Core workflow

Follow these steps in order. Each step builds on the previous one.

### Step 1: Discover subreddits

Search for the topic across Reddit to find where discussion happens.

```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/search.json?q={QUERY}&sort=relevance&t=year&limit=20" | \
  jq -r '.data.children[]? | "\(.data.subreddit)|\(.data.title)|\(.data.ups)|\(.data.num_comments)"'
```

Extract unique subreddit names from results. These become your investigation targets.

Also try searching within known communities:

```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/r/{subreddit}/search.json?q={QUERY}&sort=top&t=year&limit=20" | \
  jq -r '.data.children[]? | "\(.data.title)|\(.data.ups)|\(.data.num_comments)|https://old.reddit.com\(.data.permalink)"'
```

### Step 2: Get subreddit metadata

For each relevant subreddit, fetch its description and subscriber count:

```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/r/{subreddit}/about.json" | \
  jq '{subscribers: .data.subscribers, created: .data.created_utc, description: (.data.public_description | .[0:500]), display_name: .data.display_name}'
```

This tells you community size and whether it's worth deep-diving.

### Step 3: Fetch hot and new posts

Get both hot (popular) and new (recent) posts for breadth:

**Hot posts** (what the community values):
```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/r/{subreddit}/hot.json?limit=25" | \
  jq -r '.data.children[]? | "\(.data.title)|\(.data.ups)|\(.data.num_comments)|\(.data.author)|https://old.reddit.com\(.data.permalink)"'
```

**New posts** (what's happening right now):
```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/r/{subreddit}/new.json?limit=25" | \
  jq -r '.data.children[]? | "\(.data.title)|\(.data.ups)|\(.data.num_comments)|\(.data.author)|https://old.reddit.com\(.data.permalink)"'
```

### Step 4: Read comment threads

For posts with significant comments (or that seem relevant), fetch the full thread:

```bash
curl -s -H "User-Agent: topic-research/1.0 (research tool)" \
  "https://old.reddit.com/r/{subreddit}/comments/{post_id}/{slug}.json?limit=20" | \
  jq -r '.[1].data.children[]? | "\(.data.author)|\(.data.body|gsub("\n";" ")|.[0:200])|\(.data.ups)"'
```

The URL format uses the post's permalink — extract `post_id` and `slug` from Step 3 results.

**Note**: `.[0]` contains the original post, `.[1]` contains the comments. Always use index `[1]` for comments.

### Step 5: Categorize and analyze

Based on the data gathered, categorize posts into themes:

| Common categories | What they reveal |
|---|---|
| Bug reports | Quality issues, ALPHA instability |
| Build/strategy questions | Depth of player engagement |
| Gacha/loot reactions | Monetization sentiment |
| Community/drama | Social dynamics, moderation issues |
| Meta complaints | Balance concerns, game direction |
| New player questions | Onboarding friction points |

### Step 6: Analyze user demographics

Extract demographic signals from the data:

**From usernames**: Number suffixes (`name12345`), gaming keywords (`gamer`, `pro`, `rbx`), anime references — these indicate age groups.

**From comment language**: Scan for non-English text to estimate geographic distribution. Common patterns:
- Spanish: `el, los, por, que, juego`
- Portuguese: `eu, não, muito, jogo, brasil`
- French: `je, pas, pour, jeu, suis`
- Filipino: `ng, ang, sa, ako, naman`
- Arabic characters, Thai characters, etc.

**From slang**: Map to generational cohorts:
- Gen Alpha (8-14): `skibidi, sigma, rizz, aura`
- Gen Z (10-25): `bro, fr, ngl, fire, lit, cap, slay`
- Both: `cooked, goat, W`

**From post content**: School references, parent mentions, spending language — these triangulate age and purchasing power.

### Step 7: Synthesize findings

Structure the output as:

1. **Community overview** — subreddit size, activity level, creation date
2. **Post categorization** — breakdown by theme with representative examples
3. **User persona insights** — age range, geography, engagement depth
4. **Sentiment analysis** — what users love, hate, and want
5. **Key quotes** — representative comments that illustrate each finding
6. **Cross-reference signals** — how Reddit data confirms or contradicts other sources

## Important notes

- Always set a descriptive `User-Agent` header — Reddit blocks generic/default agents
- Keep requests to ~1 per second to avoid rate limiting
- `old.reddit.com` is the key — `www.reddit.com` will not work with this approach
- The `.json` suffix on any `old.reddit.com` URL returns the JSON version
- For search, `sort` can be `relevance`, `top`, `new`, or `hot`
- `t` (time range) can be `hour`, `day`, `week`, `month`, `year`, or `all`
- Some subreddits are private or quarantined — these will return errors, just skip them
- Comment depth is limited to top-level by default; use `depth=N` parameter for nesting control
