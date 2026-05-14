# gemini-video-analysis

Download YouTube videos and analyze them with Gemini CLI.

## Features

- YouTube URL → download → Gemini analysis pipeline
- Local video file analysis
- Automatic size management (keeps files under Gemini's token limit)
- Built-in prompt templates (gameplay, general, visual design)
- Rate limit retry with exponential backoff

## Prerequisites

- [yt-dlp](https://github.com/yt-dlp/yt-dlp)
- [Gemini CLI](https://github.com/google-gemini/gemini-cli)

## Usage

```
https://www.youtube.com/watch?v=XXXXX をGeminiで分析して。
各スキルのエフェクトを秒スタンプ付きで。
```

```
/tmp/video.mp4 をGeminiで分析して。UIデザインの観点から。
```
