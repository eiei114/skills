# skills

Personal collection of reusable [pi](https://github.com/earendil-works/pi-coding-agent) skills.

## Skills

| Skill | Description |
|---|---|
| [gemini-video-analysis](./skills/gemini-video-analysis/) | Download YouTube videos and analyze with Gemini CLI |

## Install

### gh command (recommended)

```bash
# Install all skills to global skill directory
gh repo clone eiei114/skills ~/.pi/agent/skills/eiei114
```

### Specific skill only

```bash
gh repo clone eiei114/skills /tmp/pi-skills && \
mkdir -p ~/.pi/agent/skills && \
cp -r /tmp/pi-skills/skills/gemini-video-analysis ~/.pi/agent/skills/gemini-video-analysis && \
rm -rf /tmp/pi-skills
```

### Project-local install

```bash
gh repo clone eiei114/skills /tmp/pi-skills && \
mkdir -p .pi/skills && \
cp -r /tmp/pi-skills/skills/gemini-video-analysis .pi/skills/gemini-video-analysis && \
rm -rf /tmp/pi-skills
```

## License

MIT
