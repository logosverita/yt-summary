# yt-summary

> **Language**: [English](./README.md) | 日本語

YouTube 動画の字幕を取得し、箇条書き要約と、ハイライト画像を埋め込んだダーク/ライト両対応の HTML を生成する Claude Code スキル。

## できること

- `yt-dlp` で動画メタデータ (タイトル・チャンネル・再生時間・章構成・視聴ヒートマップ) を取得
- 日本語/英語の字幕を取得 (HTTP 429 レート制限に自動対応)
- `ffmpeg` で 720p ストリームから heatmap・章・キーフレーズ由来のハイライトフレームを抽出し、base64 で埋め込み（上限なし。長尺動画では数十枚に達することあり）
- Claude が日本語で構造化した要約を生成
- `<video-id>.md` と `<video-id>.html` の 2 ファイルを出力 (テーマ切替、レスポンシブ、CSS 変数)

## 必要環境

```bash
# macOS
brew install yt-dlp ffmpeg python3

# Debian / Ubuntu
sudo apt install yt-dlp ffmpeg python3
```

API キーは不要。

## インストール

Claude Code のスキルフォルダにクローン:

```bash
git clone https://github.com/logosverita/yt-summary.git ~/.claude/skills/yt-summary
```

またはシンボリックリンク:

```bash
ln -s "$PWD" ~/.claude/skills/yt-summary
```

## 使い方

Claude Code で YouTube URL を貼り付け、または:

> 「この YouTube 動画を要約して: https://youtu.be/XXXX」

`yt-summary` の語で明示的に呼び出すこともできる。

## 設定

出力先は `OUTPUT_DIR` 環境変数で制御。デフォルトは `./output/youtube/`。

```bash
export OUTPUT_DIR=~/notes/references
```

`<OUTPUT_DIR>/youtube/<video-id>.md` と `<video-id>.html` が生成される。

## 関連スキル

- [x-save](https://github.com/logosverita/x-save) — X (Twitter) 投稿を自動翻訳付きで保存
- [video-transcribe](https://github.com/logosverita/video-transcribe) — ローカル動画/音声を whisper.cpp で文字起こし

## ライセンス

MIT — [LICENSE](./LICENSE) 参照

## 著者

**Miyanogawa Yuya (宮野川 裕也)**
- Email: inspirationlife.jp@gmail.com
- X: [@logosverita](https://x.com/logosverita)
