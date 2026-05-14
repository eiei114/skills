---
name: fandom-wiki-research
description: "Research Fandom Wiki sites using PlayLite CLI (Playwright). Fandom Wikis are Cloudflare-protected and require JavaScript rendering — HTTP requests fail. Use this skill whenever the user wants to extract information FROM a Fandom Wiki (e.g. *.fandom.com), research game wikis, look up Fandom articles, scrape wiki tables/infoboxes, or investigate community-maintained knowledge bases. PROACTIVELY activate on requests mentioning 'Fandom Wiki', 'fandom.com', 'game wiki', 'Roblox wiki', 'wiki for X', 'wikiで調べて', or any research task involving a Fandom-hosted wiki. Also trigger when the user asks about game mechanics, item databases, character lists, or lore that would be documented on a Fandom Wiki."
---

# Fandom Wiki Research via PlayLite CLI

Fandom Wikis (`*.fandom.com`) are protected by Cloudflare and require full JavaScript rendering. Static HTTP requests return Cloudflare challenge pages. This skill uses **PlayLite CLI** (`npx playlite`) to drive a real Chromium browser, bypass Cloudflare, and extract structured data.

## Choosing the right approach: PlayLite vs MediaWiki API

The correct tool depends on whether the wiki is Fandom-hosted (Cloudflare-protected) or independent:

| Wiki type | URL pattern | Approach | Why |
|-----------|------------|----------|-----|
| **Fandom Wiki** | `*.fandom.com` | PlayLite CLI (browser) | Cloudflare JS challenges block HTTP requests |
| **Independent wiki** | `minecraft.wiki`, `wiki.factorio.com`, etc. | MediaWiki API (HTTP) | No Cloudflare, API returns structured JSON directly |

**Decision rule:** If the URL contains `.fandom.com`, use PlayLite. For everything else, try the MediaWiki API first (`/api.php` endpoint) — it's faster and more reliable. Only fall back to PlayLite if the API is unavailable or returns empty results.

### MediaWiki API quick reference (for non-Fandom wikis)

```bash
# Get page content
URL="https://minecraft.wiki/api.php?action=parse&page=Sword&format=json&prop=wikitext"
curl -s "$URL" | python3 -c "import sys,json; print(json.load(sys.stdin)['parse']['wikitext']['*'][:2000])"

# Search pages
URL="https://minecraft.wiki/api.php?action=query&list=search&srsearch=weapon&format=json"
curl -s "$URL" | python3 -c "import sys,json; [print(r['title']) for r in json.load(sys.stdin)['query']['search']]"

# Get category members
URL="https://minecraft.wiki/api.php?action=query&list=categorymembers&cmtitle=Category:Weapons&format=json"
curl -s "$URL" | python3 -c "import sys,json; [print(r['title']) for r in json.load(sys.stdin)['query']['categorymembers']]"
```

For non-Fandom wikis, prefer API calls over browser automation — they're faster, more reliable, and don't require browser setup.

## Why PlayLite (for Fandom Wikis)

- Fandom Wikis sit behind Cloudflare JS challenges — `curl`, `browser_http_get`, and other HTTP tools fail
- PlayLite connects to Chromium via CDP and executes JS in the fully rendered page
- The `tree` command gives a compact DOM outline optimized for LLM consumption
- The `run` command executes extraction scripts with Playwright's full API

## Prerequisites

Before any Fandom Wiki research, ensure a browser session is running:

```bash
npx playlite launch --url "https://TARGET_WIKI.fandom.com/" 2>&1 &
sleep 8
npx playlite connect 2>&1
```

If a browser is already running, just connect:

```bash
npx playlite connect 2>&1
```

### When PlayLite shows "0 tabs"

PlayLite sometimes launches with `--no-startup-window`, resulting in "0 tabs" even though the browser process is running. Fix by creating a tab via CDP:

```bash
curl -s -X PUT "http://localhost:9222/json/new?https://TARGET_WIKI.fandom.com/" 2>&1
sleep 10
npx playlite connect 2>&1
```

This always works. After creating the tab, `npx playlite connect` should report "1 tab".

## Core Workflow

### Step 1: Identify the target Wiki

Determine the Fandom Wiki URL. Common patterns:

- `https://GAME-NAME.fandom.com/` — most game wikis
- `https://GAME-NAME.fandom.com/wiki/PAGENAME` — specific article
- `https://franchise.fandom.com/` — franchise hubs

If the user gives a game name but not a URL, try `https://GAME-NAME.fandom.com/` first (replace spaces with hyphens, lowercase).

### Step 2: Navigate and wait for Cloudflare

```bash
# Navigate to main page or specific article
cat > /tmp/fandom_nav.js << 'JSEOF'
await page.goto('https://TARGET_WIKI.fandom.com/wiki/PAGE_NAME', { waitUntil: 'load', timeout: 30000 });
await page.waitForTimeout(5000);
const url = await page.url();
const title = await page.title();
console.log(JSON.stringify({ url, title }));
JSEOF

npx playlite run /tmp/fandom_nav.js 2>&1
```

**Important**: Use `page.goto()` with `waitUntil: 'load'` and a `waitForTimeout(5000)` after, NOT `npx playlite navigate`. The CLI `navigate` command can fail on Fandom because it intercepts navigation events from ad/tracking iframes. The `run` command with `page.goto()` is more reliable.

**Alternative**: If `page.goto` throws `ERR_BLOCKED_BY_RESPONSE`, it means the current tab context is stale. Create a new tab:

```bash
curl -s -X PUT "http://localhost:9222/json/new?https://TARGET_WIKI.fandom.com/" 2>&1
sleep 10
```

Then connect and use `page.evaluate()` on the already-loaded page.

### Step 3: Extract main page content

```bash
cat > /tmp/fandom_extract.js << 'JSEOF'
const content = await page.evaluate(() => {
  const parserOutput = document.querySelector('.mw-parser-output');
  if (!parserOutput) return { error: 'No parser output found', url: location.href, title: document.title };
  
  // Paragraphs
  const ps = parserOutput.querySelectorAll('p');
  const paragraphs = [];
  ps.forEach(p => { const t = p.textContent.trim(); if (t.length > 5) paragraphs.push(t); });
  
  // Wiki-internal links
  const links = [];
  parserOutput.querySelectorAll('a[href^="/wiki/"]').forEach(a => {
    const t = a.textContent.trim();
    if (t.length > 0 && t.length < 80 && !t.includes(' edits')) {
      links.push({ text: t, href: a.href });
    }
  });
  // Deduplicate
  const seen = new Set();
  const uniqueLinks = links.filter(l => { if (seen.has(l.href)) return false; seen.add(l.href); return true; });
  
  // Infobox data (if present)
  const infobox = {};
  const aside = parserOutput.querySelector('aside') || parserOutput.querySelector('.portable-infobox');
  if (aside) {
    aside.querySelectorAll('.pi-item').forEach(item => {
      const label = item.querySelector('.pi-label')?.textContent?.trim();
      const value = item.querySelector('.pi-value')?.textContent?.trim() || item.querySelector('img')?.src;
      if (label && value) infobox[label] = value;
    });
  }
  
  return {
    title: document.querySelector('#firstHeading')?.textContent?.trim() || document.title,
    url: location.href,
    paragraphs,
    links: uniqueLinks.slice(0, 60),
    infobox: Object.keys(infobox).length > 0 ? infobox : null
  };
});
console.log(JSON.stringify(content, null, 2));
JSEOF

npx playlite run /tmp/fandom_extract.js 2>&1
```

### Step 4: Extract tables

Fandom Wikis often have data tables (item stats, character lists, etc.):

```bash
cat > /tmp/fandom_tables.js << 'JSEOF'
const tables = await page.evaluate(() => {
  const parserOutput = document.querySelector('.mw-parser-output');
  if (!parserOutput) return [];
  
  const result = [];
  parserOutput.querySelectorAll('table').forEach((table, idx) => {
    const rows = [];
    table.querySelectorAll('tr').forEach(tr => {
      const cells = [];
      tr.querySelectorAll('th, td').forEach(cell => {
        cells.push(cell.textContent.trim().replace(/\n+/g, ' '));
      });
      if (cells.length > 0) rows.push(cells);
    });
    if (rows.length > 1) result.push({ tableIndex: idx, rows });
  });
  return result;
});
console.log(JSON.stringify(tables, null, 2));
JSEOF

npx playlite run /tmp/fandom_tables.js 2>&1
```

### Step 5: Search within a Wiki

```bash
cat > /tmp/fandom_search.js << 'JSEOF'
await page.goto('https://TARGET_WIKI.fandom.com/wiki/Special:Search?query=SEARCH_TERM', { waitUntil: 'load', timeout: 30000 });
await page.waitForTimeout(3000);

const results = await page.evaluate(() => {
  const items = document.querySelectorAll('.unified-search__result');
  const out = [];
  items.forEach(item => {
    const title = item.querySelector('.unified-search__result__title')?.textContent?.trim();
    const href = item.querySelector('.unified-search__result__title a')?.href;
    const snippet = item.querySelector('.unified-search__result__content')?.textContent?.trim();
    if (title) out.push({ title, href, snippet: snippet?.substring(0, 300) });
  });
  return out;
});
console.log(JSON.stringify(results, null, 2));
JSEOF

npx playlite run /tmp/fandom_search.js 2>&1
```

### Step 6: Navigate to sub-pages

After extracting links from Step 3, navigate to specific articles of interest:

```bash
cat > /tmp/fandom_page.js << 'JSEOF'
await page.goto('https://TARGET_WIKI.fandom.com/wiki/ARTICLE_NAME', { waitUntil: 'load', timeout: 30000 });
await page.waitForTimeout(4000);

const content = await page.evaluate(() => {
  const po = document.querySelector('.mw-parser-output');
  if (!po) return { error: 'No content' };
  
  // Full text content (paragraphs + headings)
  const sections = [];
  let currentHeading = 'Introduction';
  let currentText = [];
  
  for (const child of po.children) {
    if (child.tagName?.match(/^H[2-6]$/)) {
      if (currentText.length > 0) {
        sections.push({ heading: currentHeading, text: currentText.join('\n') });
      }
      currentHeading = child.textContent.trim();
      currentText = [];
    } else if (child.tagName === 'P') {
      const t = child.textContent.trim();
      if (t.length > 5) currentText.push(t);
    } else if (child.tagName === 'UL' || child.tagName === 'OL') {
      const items = [];
      child.querySelectorAll('li').forEach(li => {
        const t = li.textContent.trim();
        if (t.length > 2) items.push(t);
      });
      if (items.length > 0) currentText.push(items.map(i => '- ' + i).join('\n'));
    }
  }
  if (currentText.length > 0) {
    sections.push({ heading: currentHeading, text: currentText.join('\n') });
  }
  
  return { title: document.querySelector('#firstHeading')?.textContent?.trim(), sections };
});
console.log(JSON.stringify(content, null, 2));
JSEOF

npx playlite run /tmp/fandom_page.js 2>&1
```

### Step 7: Get Wiki statistics and category listing

```bash
cat > /tmp/fandom_stats.js << 'JSEOF'
const stats = await page.evaluate(() => {
  // Wiki stats from main page
  const statsEl = document.querySelector('.mw-parser-output');
  const statsText = statsEl?.textContent || '';
  
  // Category links
  const categories = [];
  document.querySelectorAll('#catlinks a').forEach(a => {
    categories.push({ name: a.textContent.trim(), href: a.href });
  });
  
  // Navigation menu items
  const navItems = [];
  document.querySelectorAll('.wds-dropdown__content a').forEach(a => {
    const t = a.textContent.trim();
    if (t.length > 0) navItems.push({ text: t, href: a.href });
  });
  
  return { categories, navItems, pageStats: statsText.match(/\d+\s*(articles?|files?|edits?|pages?)/gi)?.join(' · ') };
});
console.log(JSON.stringify(stats, null, 2));
JSEOF

npx playlite run /tmp/fandom_stats.js 2>&1
```

## Tips and Troubleshooting

| Problem | Solution |
|---------|----------|
| "No pages found" | Create tab via CDP: `curl -s -X PUT "http://localhost:9222/json/new?URL"` |
| `ERR_BLOCKED_BY_RESPONSE` | Tab context is stale. Create new tab via CDP. |
| `Execution context was destroyed` | Page navigated (ads/trackers). Re-navigate and wait longer. |
| Cloudflare loop | Wait 10-15 seconds. Real Chromium almost always passes JS challenges. |
| Empty `.mw-parser-output` | Page still loading. Increase `waitForTimeout` to 8000+. |
| Truncated content | Fandom uses lazy loading. Scroll first: `await page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));` then wait 2s. |

## Output Format

Structure your research output as:

1. **Wiki Overview** — name, article count, scope, activity level
2. **Key Content** — main articles, categories, navigation structure
3. **Detailed Findings** — specific article content organized by section
4. **Data Tables** — extracted tables with structured data
5. **Related Pages** — links worth investigating further
6. **Cross-reference** — how this Wiki data complements other sources (Reddit, DevForum, YouTube)

## Important Notes

- Fandom Wikis use **MediaWiki** markup — class names like `.mw-parser-output`, `#firstHeading`, `.portable-infobox` are stable
- Wikis vary greatly in quality and completeness — check the edit count and last-updated dates
- Some Wikis have custom CSS/JS that changes DOM structure — `tree` first, then adapt selectors
- Fandom injects ads and tracking iframes — ignore these, focus on `.mw-parser-output` content
- Always run `waitForTimeout(3000-5000)` after navigation — Fandom loads content asynchronously
- For large Wikis, use the search function rather than trying to browse all pages
