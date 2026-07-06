---
name: apple-developer-forums-research
description: "Research Apple Developer Forums using tag RSS feeds, tag pages, and Apple Developer documentation. Use this skill whenever the user wants to investigate Apple Developer Forums, iOS/macOS/watchOS/visionOS developer discussions, SwiftUI/UIKit/SpriteKit/Core Haptics/StoreKit issues, Apple platform implementation gotchas, or asks for 'Apple Developer Forums', 'developer.apple.com/forums', 'iOS Dev Forum', 'AppDeveloper Forum', 'Appleフォーラム', or forum-based evidence for Apple-platform decisions. Prefer RSS feeds first because forum thread pages can require security verification."
---

# Apple Developer Forums Research

Use this skill to research Apple Developer Forums without relying on browser-only pages. Apple forum thread pages often hit security verification, but tag pages and tag RSS feeds are usually readable.

## When to use

Use for:

- Apple Developer Forums research
- iOS/macOS/watchOS/visionOS implementation issues
- SwiftUI, UIKit, SpriteKit, RealityKit, Core Haptics, StoreKit, AVFoundation, GameController forum signals
- OS-version regressions, crash signatures, framework gotchas
- User asks for Apple Developer Forum / AppDeveloper Forum / developer.apple.com/forums evidence

Do not use for:

- General web search with no Apple-platform angle
- Official API docs only, unless forum discussion/context is also needed
- Stack Overflow / Reddit community research

## Source strategy

Use sources in this order:

1. **Tag RSS feed** — fastest, stable, includes title/link/excerpt/date/author.
2. **Tag page** — useful to discover RSS URL and current post list.
3. **Official Apple docs / HIG / WWDC** — use to verify the canonical API/design guidance.
4. **Thread page** — try only when needed; may show security verification.

## Discover tag RSS

Apple tag pages usually expose an “RSS for tag” link.

```text
https://developer.apple.com/forums/tags/{tag}
https://developer.apple.com/forums/tags/rssFeed/{tag}
```

Examples:

```text
https://developer.apple.com/forums/tags/rssFeed/core-haptics
https://developer.apple.com/forums/tags/rssFeed/spritekit
https://developer.apple.com/forums/tags/rssFeed/swiftui
https://developer.apple.com/forums/tags/rssFeed/storekit
https://developer.apple.com/forums/tags/rssFeed/avfoundation
```

## Fetch and parse RSS

```bash
python3 - <<'PY'
import urllib.request, xml.etree.ElementTree as ET, re, html

tag = '{tag}'
url = f'https://developer.apple.com/forums/tags/rssFeed/{tag}'
req = urllib.request.Request(url, headers={'User-Agent': 'apple-forums-research/1.0'})
with urllib.request.urlopen(req, timeout=25) as r:
    root = ET.fromstring(r.read())

channel = root.find('channel')
print('#', channel.findtext('title', default=tag))
print(channel.findtext('description', default='').strip())

for item in channel.findall('item')[:15]:
    title = item.findtext('title', default='').strip()
    link = item.findtext('link', default='').strip()
    date = item.findtext('pubDate', default='').strip()
    author = item.findtext('author', default='').strip()
    encoded = ''
    for child in item:
        if child.tag.endswith('encoded'):
            encoded = child.text or ''
            break
    text = html.unescape(re.sub('<[^<]+?>', ' ', encoded))
    text = re.sub(r'\s+', ' ', text).strip()
    print(f'\n- {title}\n  {link}\n  {date} · {author}\n  {text[:500]}')
PY
```

## Search within RSS results

Apple's forum RSS is tag-scoped, not query-scoped. For keyword research:

1. Pick likely tags from the technology area.
2. Fetch each tag RSS.
3. Filter titles/excerpts locally for keywords.

```bash
python3 - <<'PY'
import urllib.request, xml.etree.ElementTree as ET, re, html

tags = ['core-haptics', 'spritekit', 'swiftui']
keywords = ['haptic', 'audio', 'sound', 'SpriteKit', 'crash', 'performance']

for tag in tags:
    url = f'https://developer.apple.com/forums/tags/rssFeed/{tag}'
    try:
        req = urllib.request.Request(url, headers={'User-Agent': 'apple-forums-research/1.0'})
        root = ET.fromstring(urllib.request.urlopen(req, timeout=25).read())
    except Exception as e:
        print('ERR', tag, e)
        continue
    for item in root.find('channel').findall('item'):
        title = item.findtext('title', default='')
        link = item.findtext('link', default='')
        text = ''
        for child in item:
            if child.tag.endswith('encoded'):
                text = html.unescape(re.sub('<[^<]+?>', ' ', child.text or ''))
        haystack = (title + ' ' + text).lower()
        if any(k.lower() in haystack for k in keywords):
            excerpt = re.sub(r'\s+', ' ', text).strip()[:350]
            print(f'[{tag}] {title}\n  {link}\n  {excerpt}')
PY
```

## Verify with official Apple docs

Forum posts are evidence of real-world failures and edge cases, not canonical guidance. Cross-check decisions against official docs:

- HIG: `https://developer.apple.com/design/human-interface-guidelines/...`
- Documentation: `https://developer.apple.com/documentation/{framework}`
- WWDC videos: `https://developer.apple.com/videos/play/...`

When docs pages require JavaScript in normal HTML, try the tutorials data path:

```text
https://developer.apple.com/tutorials/data/documentation/{framework-lower}/{page}.md
https://developer.apple.com/tutorials/data/design/human-interface-guidelines/{page}.md
```

Examples:

```text
https://developer.apple.com/tutorials/data/documentation/corehaptics/preparing-your-app-to-play-haptics.md
https://developer.apple.com/tutorials/data/design/human-interface-guidelines/playing-haptics.md
https://developer.apple.com/tutorials/data/design/human-interface-guidelines/motion.md
```

## Report format

Structure results like this:

```markdown
## Apple Developer Forums findings

### Sources checked
| Source | Status | Notes |
|---|---|---|
| core-haptics RSS | OK | 15 items scanned |
| spritekit RSS | OK | 15 items scanned |
| thread page | blocked | security verification |

### Forum signals
| Signal | Evidence | Product implication |
|---|---|---|
| Core Haptics can fail on device/OS edge cases | thread title + link | add capability checks and fallback |

### Canonical Apple guidance
- Official docs/HIG/WWDC summary with links.

### Recommendation
- Concrete implementation/design guidance.

### Caveats
- RSS excerpts may omit accepted answers or deep replies.
- Thread pages can require security verification.
```

## Interpretation rules

- Treat forum posts as **risk signals**, not definitive platform behavior.
- Prefer repeated signals across multiple posts/tags over a single anecdote.
- Preserve OS versions, device models, framework names, and error codes.
- If a thread page is blocked, cite the RSS link/title and say the full thread was not accessible.
- For implementation plans, turn forum risk into fallback, logging, capability checks, or test coverage.
