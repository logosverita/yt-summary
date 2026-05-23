# yt-summary

> **Language**: English | [日本語](./README.ja.md)

A Claude Code skill that fetches YouTube transcripts, generates bullet-point summaries, and saves both Markdown and a self-contained dark/light HTML page — with key-moment screenshots embedded inline.

## What it does

- Fetches video metadata (title, channel, duration, chapters, heatmap) via `yt-dlp`
- Pulls Japanese or English subtitles (with HTTP 429 retry handling)
- Extracts highlight frames (heatmap / chapters / keyphrases) from the 720p stream via `ffmpeg` and embeds them as base64 — no upper limit, so long videos may yield dozens of frames
- Lets Claude write a sectioned Japanese summary
- Emits both `<video-id>.md` and `<video-id>.html` (theme toggle, responsive, CSS variables)

## Requirements

```bash
# macOS
brew install yt-dlp ffmpeg python3

# Debian / Ubuntu
sudo apt install yt-dlp ffmpeg python3
```

No API keys required.

## Installation

Copy this directory into your Claude Code skills folder:

```bash
git clone https://github.com/logosverita/yt-summary.git ~/.claude/skills/yt-summary
```

Or symlink it from a checkout:

```bash
ln -s "$PWD" ~/.claude/skills/yt-summary
```

## Usage

In Claude Code, paste a YouTube URL or say:

> "Summarize this YouTube video: https://youtu.be/XXXX"

Triggers also fire on the literal word `yt-summary`.

## Configuration

The output location is controlled by the `OUTPUT_DIR` environment variable. Default: `./output/youtube/`.

```bash
export OUTPUT_DIR=~/notes/references
```

The skill writes `<OUTPUT_DIR>/youtube/<video-id>.md` and `<video-id>.html`.

## Related skills

- [x-save](https://github.com/logosverita/x-save) — Save X (Twitter) posts with auto-translation
- [video-transcribe](https://github.com/logosverita/video-transcribe) — Local audio/video transcription with whisper.cpp

## License

MIT — see [LICENSE](./LICENSE)

## Author

**Miyanogawa Yuya**
- Email: inspirationlife.jp@gmail.com
- X: [@logosverita](https://x.com/logosverita)
