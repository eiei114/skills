---
name: gemini-video-analysis
description: "Download YouTube videos and analyze them with Gemini CLI. Use when the user wants to analyze video content, extract visual information from gameplay footage, review video evidence, or says 'Gemini video', 'video analysis', 'analyze this video', 'download and analyze', or mentions YouTube analysis. Also use when user has local video files (.mp4, .webm, .mov) they want Gemini to analyze. Handles download size management, Gemini token limits, and rate limiting automatically."
---

# Gemini Video Analysis

Download videos (YouTube or local) and analyze them with Gemini CLI. This skill handles the tricky parts: keeping file sizes under Gemini's token limit, choosing the right download format, and retrying on rate limits.

## Why this skill exists

Gemini CLI can analyze video files directly — you pass a local file path and Gemini "sees" the content. But there are traps:

1. **Token limit**: Videos expand to ~2-3M tokens per MB. Gemini's limit is ~1M tokens. A 3.4MB Shorts video can exceed the limit.
2. **No ffmpeg**: Most systems don't have ffmpeg installed, so you can't split videos. Must select the right format upfront.
3. **Rate limits**: Gemini CLI hits 429 errors under load. Need retries with backoff.
4. **Binary stdin breaks**: You cannot pipe binary video data to Gemini via stdin. Must reference the file path in the prompt text.

This skill avoids all of these.

## Prerequisites

- `yt-dlp` must be installed (`brew install yt-dlp` or `pip install yt-dlp`)
- `gemini` CLI must be installed and authenticated (`npm i -g @anthropic-ai/gemini-cli` or `brew install gemini-cli`)
- Check both before starting:
  ```bash
  which yt-dlp && yt-dlp --version
  which gemini && gemini --version
  ```

## Workflow

### Step 1: Determine input source

**If user provides a YouTube URL:**
- Extract the video ID
- Go to Step 2

**If user provides a local file path:**
- Check file size: `ls -lh <path>`
- If under 2MB: proceed directly to Step 4
- If 2-5MB: warn the user that Gemini may reject it; try anyway
- If over 5MB: tell the user the file is too large for Gemini's token limit and suggest alternatives (download a smaller version from YouTube, or describe specific timestamps they want analyzed)

**If user provides a search query:**
- Tell the user to find the specific YouTube URL first. Don't attempt to search YouTube programmatically — search results scraping is unreliable.

### Step 2: Download video (size-aware)

The target is **under 1.5MB** for reliable Gemini analysis. Here's the strategy:

```bash
# Create working directory
WORKDIR="/tmp/gemini-video-$(date +%s)"
mkdir -p "$WORKDIR"

# First, check available formats
yt-dlp --list-formats "$URL" 2>&1
```

**Format selection priority** (try in order until you find one under 2MB):

1. **Check duration first** — Videos under 60 seconds (Shorts) usually work with any format.
   ```bash
   yt-dlp --get-duration "$URL"
   ```

2. **For videos under 60 seconds:**
   ```bash
   yt-dlp -f 18 -o "$WORKDIR/%(id)s.%(ext)s" --force-overwrites "$URL"
   ```

3. **For videos 60 seconds to 5 minutes:**
   ```bash
   # Try smallest audio+video format
   yt-dlp -f "worst[ext=mp4]" -o "$WORKDIR/%(id)s.%(ext)s" --force-overwrites "$URL"
   ```

4. **For videos over 5 minutes:**
   - Warn the user: "This video is long. Gemini can only analyze ~1-2 minutes of footage per call. I'll download the smallest format, but it may still exceed Gemini's token limit."
   - Try format 160 (video-only, very small) + suggest the user tell you which segment to focus on:
   ```bash
   yt-dlp -f 160 -o "$WORKDIR/%(id)s.%(ext)s" --force-overwrites "$URL"
   ```
   - Note: format 160 has NO AUDIO. Tell the user.

**After download, verify size:**
```bash
ls -lh "$WORKDIR"/*.mp4
```

If the file exceeds 2MB:
- Check if there's a smaller format available
- If not, inform the user of the limitation and ask if they want to try anyway (it sometimes works for files up to ~3MB depending on content complexity)
- For Shorts that exceed 2MB, it's usually because they're 60-second vertical videos — try re-downloading with lower quality:
  ```bash
  yt-dlp -f "worstvideo+worstaudio/worst" -o "$WORKDIR/%(id)s.%(ext)s" --force-overwrites "$URL"
  ```

### Step 3: Prepare the analysis prompt

Use one of the templates below, or let the user provide their own prompt. Always include the file path reference in the prompt.

#### Template: Gameplay Combat Analysis

```
動画ファイル {FILEPATH} を分析してください。
これは{GAME_NAME}のゲームプレイ映像です。

以下を秒タイムスタンプ付きで分析:
1. M1（通常攻撃）: モーション速度、エフェクトの色、ダメージ数字の色と値
2. Skill Z/X/C: エフェクトの色・範囲・大きさ、敵の反応
3. Awakening/Ultimate: 演出の規模、画面効果
4. ダメージ数字: 通常色、Crit色と大きさ、確認できる最大値
5. 敵の死に方: バリベーション、同時消滅数
6. 最も爽快な瞬間とその秒タイミング

最後に、この映像から見える「戦闘が面白い理由」を3つ言語化してください。
```

#### Template: General Video Analysis

```
動画ファイル {FILEPATH} を分析してください。

内容を秒タイムスタンプ付きで時系列に描写してください。
重要な場面転換、視覚的変化、注目すべき瞬間をすべて含めてください。
```

#### Template: Visual Design Analysis

```
動画ファイル {FILEPATH} を分析してください。
UI/UXデザインの観点から以下を分析:

1. 色使い: 主要な色とその意味・役割
2. エフェクト: 種類、色、画面占有率
3. 数字表示: 色、サイズ、出現タイミング
4. 画面効果: 揺れ、フラッシュ、ヒットストップ
5. タイポグラフィ: フォント、サイズ階層

各観点について秒タイムスタンプ付きで実例を挙げてください。
```

**If the user provides their own prompt**, prepend the file path reference:
```
動画ファイル {FILEPATH} を分析してください。
{USER_PROMPT}
```

### Step 4: Run Gemini analysis

```bash
cd "$WORKDIR"
gemini --skip-trust -o text -p "{PROMPT_TEXT}" 2>&1
```

**Important flags:**
- `--skip-trust`: Skip trust dialog (required for non-interactive use)
- `-o text`: Output plain text (avoids JSON wrapping issues)

**The prompt text must reference the file by its absolute path.** Do NOT pipe the file via stdin — binary data through stdin causes garbled path errors.

**Timeout:** Use a generous timeout (180-300 seconds). Video analysis is slow.

### Step 5: Handle errors

| Error | Meaning | Action |
|-------|---------|--------|
| `input token count exceeds maximum` | Video too large for Gemini | Try a smaller format or tell user to pick a shorter video |
| `429 / rate limit / model capacity exhausted` | Too many requests | Wait 30 seconds and retry. Maximum 3 retries with exponential backoff (30s, 60s, 120s) |
| `File size exceeds the 20MB limit` | File too large even for upload | Need smaller format |
| `Ripgrep is not available` | Warning only, can ignore | This is a harmless warning from Gemini CLI |
| Tool call blocked / `run_shell_command` not found | Gemini tried to use tools internally | Harmless — it falls back to direct analysis |
| Empty output with no error | Gemini processed but returned nothing | Retry once. If still empty, the video may be too small/low-quality for Gemini to parse |

**Rate limit retry pattern:**
```bash
MAX_RETRIES=3
RETRY_DELAY=30
for i in $(seq 1 $MAX_RETRIES); do
  result=$(gemini --skip-trust -o text -p "$PROMPT" 2>&1)
  if echo "$result" | grep -qi "429\|capacity\|rate limit"; then
    echo "Rate limited (attempt $i/$MAX_RETRIES). Waiting ${RETRY_DELAY}s..."
    sleep $RETRY_DELAY
    RETRY_DELAY=$((RETRY_DELAY * 2))
    continue
  fi
  echo "$result"
  break
done
```

### Step 6: Batch analysis (multiple videos)

When the user wants to analyze multiple videos:

1. Download all videos first (sequentially)
2. Verify all sizes are under 2MB
3. Analyze one at a time (Gemini doesn't support multiple video files in a single call — the token count multiplies)
4. If any file exceeds 2MB, skip it and warn the user
5. Collect all results

```bash
for file in "$WORKDIR"/*.mp4; do
  size=$(stat -f%z "$file" 2>/dev/null || stat -c%s "$file" 2>/dev/null)
  if [ "$size" -gt 2097152 ]; then
    echo "SKIP: $(basename $file) is too large ($(($size/1024))KB)"
    continue
  fi
  echo "=== Analyzing $(basename $file) ==="
  # Run analysis with retry...
done
```

## Size Reference Table

Use this to set expectations with the user:

| Video Length | Typical Size (format 18) | Gemini Status |
|---|---|---|
| < 15 seconds | 500KB - 1MB | ✅ Always works |
| 15-30 seconds | 1MB - 2MB | ✅ Usually works |
| 30-60 seconds | 1.5MB - 3MB | ⚠️ May fail |
| 1-3 minutes | 3MB - 15MB | ❌ Too large |
| 3-10 minutes | 15MB - 50MB | ❌ Too large |
| 10+ minutes | 50MB+ | ❌ Way too large |

For longer videos, the best approach is to find YouTube Shorts or clips that show the specific moments the user cares about.

## Tips

- **Shorts are your friend.** YouTube Shorts are 15-60 seconds and naturally fall in the sweet spot for Gemini analysis. When the user wants to analyze gameplay, suggest searching for Shorts/showcases rather than full gameplay videos.
- **The `--skip-trust` flag is mandatory.** Without it, Gemini CLI opens an interactive trust dialog that blocks execution.
- **Don't combine multiple video files in one prompt.** Token count adds up. Even two 1MB files can exceed the limit.
- **Gemini's output includes its "thinking" process** (tool calls it tries, errors it hits). The actual analysis is usually at the end. Filter out the tool-call noise when presenting results.
- **Workspace context matters.** If the current directory has lots of files, Gemini may try to read them all. Always `cd` to the video directory before running.
- **Clean up.** Videos in `/tmp/` are lost on reboot. If the user wants to keep them, copy to a persistent location.
