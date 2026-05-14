---
name: devforum-research
description: "Research the Roblox Developer Forum (devforum.roblox.com) using PlayLite CLI (Playwright). Use this skill whenever the user wants to investigate what Roblox developers are discussing, search DevForum for topics, analyze developer sentiment, or gather technical discussions about Roblox game development. PROACTIVELY activate on requests mentioning 'DevForum', 'Roblox developer forum', 'Roblox developers think', 'DevForumで調べて', '開発者フォーラム', or any research task involving Roblox's developer community. Also trigger when the user asks about game genre viability on Roblox, developer tooling discussions, platform policy feedback, or competitive analysis of Roblox games from a developer perspective."
---

# DevForum Research via PlayLite CLI

DevForum (`devforum.roblox.com`) is a Discourse-based forum where Roblox developers discuss game design, scripting, platform features, and community issues. It requires JavaScript rendering — `curl` and HTTP GET tools return empty or Cloudflare-challenge pages. This skill uses **PlayLite CLI** (`npx playlite`) to drive a real Chromium browser and extract structured data.

## Why PlayLite

- DevForum is a single-page JavaScript app — static HTTP requests fail
- PlayLite connects to Chromium via CDP and executes JS in the fully rendered page
- The `tree` command gives a compact DOM outline optimized for LLM consumption
- The `run` command executes extraction scripts with Playwright's full API

## Prerequisites

Before any DevForum research, ensure a browser session is running:

```bash
# Launch Chromium with remote debugging
npx playlite launch --url "https://devforum.roblox.com" 2>&1 &
sleep 5

# Verify connection
npx playlite connect 2>&1
```

If a browser is already running (from a previous session), just connect:

```bash
npx playlite connect 2>&1
```

## Core workflow

Follow these steps in order.

### Step 1: Search DevForum

Navigate to the search page with a query:

```bash
npx playlite navigate "https://devforum.roblox.com/search?q={QUERY}" 2>&1
sleep 5
```

**Search tips:**
- Use `%20` for spaces, `%22` for exact phrase matching
- Append `&category=3` to restrict to Community category
- Common categories: 3 (Community), 54 (Help & Feedback), 12 (Forum Help)
- Try multiple query variations — DevForum search is keyword-based and can miss relevant posts

**Check for results:**

```bash
npx playlite tree ".search-results" 2>&1 | head -10
```

If you see `div.no-results-container` with "何も見つかりませんでした" (nothing found), try broader queries or different keywords.

### Step 2: Extract search results

Create an extraction script and run it:

```bash
cat > /tmp/devforum_extract.js << 'JSEOF'
const items = await page.$$('.fps-result');
const results = [];
for (const item of items) {
  const title = await item.$eval('.topic-title', el => el.textContent.trim()).catch(() => '');
  const href = await item.$eval('.search-link', el => el.href).catch(() => '');
  const blurb = await item.$eval('.blurb', el => el.textContent.trim().substring(0, 300)).catch(() => '');
  const author = await item.$eval('.author a', el => el.href.replace('https://devforum.roblox.com/u/', '')).catch(() => '');
  const tags = [];
  const tagEls = await item.$$('.discourse-tag');
  for (const t of tagEls) {
    tags.push(await t.textContent());
  }
  results.push({ title, href, blurb, author, tags });
}
console.log(JSON.stringify(results, null, 2));
JSEOF

npx playlite run /tmp/devforum_extract.js 2>&1
```

This returns a JSON array with title, URL, blurb, author, and tags for each result.

### Step 3: Read individual posts

For relevant threads, navigate and extract posts:

```bash
npx playlite navigate "https://devforum.roblox.com/t/{slug}/{id}" 2>&1
sleep 5
```

Then extract posts:

```bash
cat > /tmp/devforum_posts.js << 'JSEOF'
const posts = await page.$$('.topic-post');
const results = [];
for (const post of posts.slice(0, 15)) {
  const author = await post.$eval('.username', el => el.textContent.trim()).catch(() => '');
  const content = await post.$eval('.cooked', el => el.textContent.trim().substring(0, 500)).catch(() => '');
  const likes = await post.$eval('.like-count', el => el.textContent.trim()).catch(() => '0');
  results.push({ author, content, likes });
}
console.log(JSON.stringify(results, null, 2));
JSEOF

npx playlite run /tmp/devforum_posts.js 2>&1
```

### Step 4: Navigate pagination (if needed)

For long threads, scroll down to load more posts:

```bash
npx playlite eval -e "window.scrollTo(0, document.body.scrollHeight)" 2>&1
sleep 3
```

Then re-run the extraction script.

### Step 5: Browse by category or tag

If search doesn't yield results, browse directly:

**By category:**
```bash
npx playlite navigate "https://devforum.roblox.com/c/help-and-feedback/creations-feedback/20" 2>&1
sleep 5
```

**By tag:**
```bash
npx playlite navigate "https://devforum.roblox.com/tag/zombies" 2>&1
sleep 5
```

Extract topic listings from category/tag pages:

```bash
cat > /tmp/devforum_topics.js << 'JSEOF'
const rows = await page.$$('tr.topic-list-item');
const results = [];
for (const row of rows.slice(0, 25)) {
  const title = await row.$eval('.title', el => el.textContent.trim()).catch(() => '');
  const href = await row.$eval('.title', el => el.href).catch(() => '');
  const replies = await row.$eval('.replies .number', el => el.textContent.trim()).catch(() => '');
  const views = await row.$eval('.views .number', el => el.textContent.trim()).catch(() => '');
  results.push({ title, href, replies, views });
}
console.log(JSON.stringify(results, null, 2));
JSEOF

npx playlite run /tmp/devforum_topics.js 2>&1
```

### Step 6: Analyze findings

Structure analysis around these dimensions:

| Dimension | What to extract |
|---|---|
| **Topic themes** | Game design, scripting help, platform feedback, hiring, show-and-tell |
| **Developer sentiment** | Frustration with platform changes, excitement about new features, concern about discovery |
| **Genre viability** | Is the genre saturated? What gaps exist? What do developers wish existed? |
| **Technical discussions** | Common challenges, recommended tools/patterns, performance concerns |
| **Platform issues** | Discovery algorithm complaints, moderation concerns, monetization discussions |
| **Community dynamics** | Who are the active contributors? What gets upvoted? What gets ignored? |

### Step 7: Synthesize into report

Output format:

1. **Search summary** — queries tried, results found, relevant threads identified
2. **Key discussions** — top threads with representative quotes and like counts
3. **Developer sentiment** — what developers love, hate, and want from the platform
4. **Genre/topic insights** — viability assessment, gaps, opportunities
5. **Cross-reference** — how DevForum data complements Reddit, YouTube, and other sources

## Important notes

- DevForum is primarily in **English** but the UI may appear in the user's locale (Japanese, etc.)
- The forum is a Discourse instance — class names like `.fps-result`, `.topic-post`, `.cooked` are stable
- DevForum has rate limits but they're generous for normal browsing
- Some threads require login to view — these will show a login prompt instead of content
- Tags are a powerful discovery tool — check `/tag/{tag-name}` for topics like `zombies`, `survival`, `scripting`, `gamedesign`
- DevForum content is developer-focused — for player opinions, use Reddit and YouTube instead
- Always run `sleep 3-5` after navigation — Discourse loads content asynchronously
- The `tree` command is your best friend for understanding page structure before extraction
