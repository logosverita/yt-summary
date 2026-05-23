---
name: yt-summary
description: "YouTube動画の文字起こしを取得し箇条書きで要約（MD + HTMLを同時保存）"
argument-hint: "<YouTube URL>"
allowed-tools: [Bash, Read, Write, Glob]
---

# YouTube動画 要約

YouTube動画のURLから字幕（文字起こし）を取得し、箇条書きで要約して `${OUTPUT_DIR:-./output}/youtube/` に **Markdown と HTML の両方** で保存する。

## 入力

`$ARGUMENTS` にYouTube URLが1つ渡される。引数には URL 以外の指示文（「htmlとmdで保存」等）が混在することがあるため、URL パターンを抽出して使用する。

追加指示の解釈：
- 「日本語に翻訳」「日本語訳」→ §5 要約に加え、文字起こし全文も日本語訳として併記する。原語動画でも日本語要約のみで完結したい場合は要約セクションを充実させること。
- 「要約だけ」「短く」→ 文字起こしブロックを省略可。
- 「mdだけ」「htmlだけ」→ 該当フォーマットのみ生成。デフォルトは両方。
- 「画像不要」「サムネなし」「テキストだけ」→ §5.5 ハイライト・フレーム抽出をスキップし、従来のテキストのみで生成。デフォルトはハイライト画像あり。

## 環境前提（Claude Code Bash の落とし穴）

Claude Code の Bash ツールは **コマンド毎に cwd と環境変数がリセット** される。次の点に注意：

1. **`TMPDIR` 変数は次回 Bash 呼び出しで消える**。対策：
   ```bash
   TMPDIR=$(mktemp -d) && echo "$TMPDIR" > /tmp/yt_tmpdir.txt
   ```
   後続呼び出しでは `TMPDIR=$(cat /tmp/yt_tmpdir.txt)` で復元。
2. **`cd` も次回には失われる**。yt-dlp の出力は **絶対パス `-o "$TMPDIR/sub"`** で指定するか、`cd && yt-dlp && ls -la` を同一コマンド内で連結する。
3. **`yt-dlp` の `[info] Writing video subtitles to: sub.ja.srt` ログは"宣言"であり、実体ファイルがない場合でも表示される**。必ず直後に `ls -la "$TMPDIR"` で実体を確認すること。
4. zsh の no-match エラーを避けるため、複数候補のファイル検出は **for ループ + `[ -f ]`** で行う（後述）。
5. **`ffmpeg` の stdin 消費**: §5.5 のフレーム抽出で `while read` ループ内から `ffmpeg` を呼ぶと、ffmpeg がループの stdin を食い潰してフィールドがずれる。**必ず `ffmpeg -nostdin ... </dev/null`** を付ける。
6. **ハイライト画像には `ffmpeg` が必須**。未インストールならフレーム抽出のみスキップし、テキストレポートは通常どおり生成する（`brew install ffmpeg`）。

## 実行手順

### 1. URLバリデーション

`$ARGUMENTS` から正規表現で YouTube URL を抽出する。受け付けるパターン：
- `https://www.youtube.com/watch?v=XXXXX`
- `https://youtu.be/XXXXX`
- `https://m.youtube.com/watch?v=XXXXX`

URLが抽出できない場合はエラーメッセージを表示して終了。

URLから `video_id`（11文字）を抽出する（ファイル名に使用）。

### 2. メタデータ取得

```bash
yt-dlp --dump-json --no-download "$URL" 2>"$TMPDIR/stderr_meta.log" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for k in ('title', 'upload_date', 'duration_string', 'channel'):
    print(f'{k.upper()}: {d.get(k, \"\")}')"
```

失敗時は `cat "$TMPDIR/stderr_meta.log"` でエラー内容を確認し、ユーザーに報告。

抽出する項目：
- `title`: 動画タイトル
- `upload_date`: 投稿日（YYYYMMDD形式）
- `duration_string`: 再生時間
- `channel`: チャンネル名
- `description`: 動画説明文（任意）

**ハイライト用に `heatmap` と `chapters` も取得**（§5.5 で使用）。JSON 全体を `$TMPDIR/meta.json` に保存しておくと再利用が楽：

```bash
yt-dlp --dump-json --no-download "$URL" 2>"$TMPDIR/stderr_meta.log" > "$TMPDIR/meta.json"
python3 -c "
import json
d=json.load(open('$TMPDIR/meta.json'))
print('heatmap:', 'present' if d.get('heatmap') else 'NONE')
for c in (d.get('chapters') or []):
    s=int(c['start_time']); print(f'  {s:5d}s {s//60:02d}:{s%60:02d} {c.get(\"title\",\"\")}')"
```

- `heatmap`: `[{start_time, end_time, value}]`。**Most Replayed（多くのユーザーが再生/巻き戻した区間）**。再生数の少ない動画や投稿直後は存在しないことが多い（`None`）。
- `chapters`: `[{start_time, end_time, title}]`。投稿者定義の章。タイトル付きで信頼性が高い。

### 3. 字幕取得

一時ディレクトリを作成し、字幕をダウンロード。**初回から `--sleep-subtitles 5` を指定して 429 を予防** する：

```bash
TMPDIR=$(mktemp -d)
echo "$TMPDIR" > /tmp/yt_tmpdir.txt
yt-dlp --skip-download --write-sub --write-auto-sub \
    --sub-lang "ja,en" --sub-format "srt/vtt/best" \
    --sleep-subtitles 5 \
    -o "$TMPDIR/sub" "$URL" 2>"$TMPDIR/stderr_sub.log"
ls -la "$TMPDIR"   # ← 実体ファイルがあるか必ず確認
```

- `--sub-format "srt/vtt/best"`: SRTをネイティブ取得し、なければVTTにフォールバック
- `--sleep-subtitles 5`: 言語ごとに5秒スリープし、レート制限を回避（必須）
- stderrは一時ファイルに保存。失敗時は `cat "$TMPDIR/stderr_sub.log"` でエラー内容を確認

**HTTP 429（レート制限）リトライ**：
stderr に `HTTP Error 429` が含まれていた場合、20秒待機して言語を絞り（まず `en` 自動字幕のみ）+ より長いスリープで再試行する。それでも失敗したらエラー報告。

```bash
if grep -q "HTTP Error 429" "$TMPDIR/stderr_sub.log"; then
    sleep 20
    yt-dlp --skip-download --write-auto-sub --sub-lang "en" \
        --sub-format "vtt/srt/best" --sleep-subtitles 10 \
        -o "$TMPDIR/sub" "$URL" 2>"$TMPDIR/stderr_sub2.log"
    ls -la "$TMPDIR"
fi
```

**注意**: `[info] Writing video subtitles to: sub.ja.srt` というログが出ても、その直後に 429 で実体ファイルが作られないケースがある。**`ls` でファイル存在を確認するまで成功と判定しないこと**。

**言語優先順序**（拡張子 `.srt` と `.vtt` の両方をチェック）:
1. 日本語字幕（`.ja.srt` → `.ja.vtt`）
2. 英語字幕（`.en.srt` → `.en.vtt`）

ダウンロード後、字幕ファイルを検出（zsh で no-match エラーにならないよう個別チェック）：
```bash
for cand in "$TMPDIR/sub.ja.srt" "$TMPDIR/sub.ja.vtt" "$TMPDIR/sub.en.srt" "$TMPDIR/sub.en.vtt"; do
    if [ -f "$cand" ]; then SUB_FILE="$cand"; break; fi
done
```

**字幕なしの場合**: `$SUB_FILE` が空なら、stderrログの内容を確認した上でエラーメッセージを表示し、一時ディレクトリを削除して終了。

### 4. 字幕パース（SRT/VTT両対応）

以下のPythonスクリプトで字幕ファイルをプレーンテキストに変換する。SRT・VTT（YouTube自動生成のカラオケ形式含む）の両方に対応：

```bash
python3 - "$SUB_FILE" << 'PYEOF' > "$TMPDIR/transcript.txt"
import re, sys

with open(sys.argv[1], "r", encoding="utf-8") as f:
    content = f.read()

# Remove VTT/SRT headers
content = re.sub(r'^(WEBVTT|Kind:.*|Language:.*)\n', '', content, flags=re.MULTILINE)

lines = content.split('\n')
text_lines = []
for line in lines:
    line = line.strip()
    if not line:
        continue
    if '-->' in line:                          # タイムスタンプ行
        continue
    if re.match(r'^\d+$', line):               # SRT連番行
        continue
    if '<c>' in line or re.search(r'<\d{2}:', line):  # VTTカラオケ行（スキップ）
        continue
    line = re.sub(r'<[^>]+>', '', line)        # HTMLタグ除去
    if line.strip():
        text_lines.append(line.strip())

# 連続する重複行を除去
deduped = []
for line in text_lines:
    if not deduped or line != deduped[-1]:
        deduped.append(line)

# 英語字幕はスペース、日本語字幕は無区切りで結合
sep = ' ' if sys.argv[1].endswith(('.en.srt', '.en.vtt')) else ''
print(sep.join(deduped))
PYEOF
```

パース後、`$TMPDIR/transcript.txt` を `Read` ツールで読み取る。

### 4.5 タイムスタンプ付き transcript（ハイライト選定用）

`heatmap` も `chapters` も無い場合のフォールバックとして、SRT から `start_sec|テキスト` 形式の **timed transcript** を別途生成する。これを使い、要約の論点に対応する代表時刻をキーフレーズ検索で特定できる（VTT のみの動画では `--write-sub` で SRT 化を優先）。

```bash
python3 - "$SUB_FILE" << 'PYEOF' > "$TMPDIR/timed.txt"
import re, sys
content = open(sys.argv[1], encoding="utf-8").read()
for b in re.split(r'\n\n+', content.strip()):
    lines = b.strip().split('\n')
    if len(lines) < 2: continue
    m = re.search(r'(\d{2}):(\d{2}):(\d{2})', lines[1])
    if not m: continue
    h, mi, s = map(int, m.groups())
    text = re.sub(r'<[^>]+>', '', "".join(lines[2:]))
    if text.strip(): print(f"{h*3600+mi*60+s}|{text.strip()}")
PYEOF
```

### 5. 要約生成

文字起こし全文をもとに、**日本語で**箇条書き要約を生成する。

要約のガイドライン：
- 動画の主要なポイントを箇条書きで列挙
- 各箇条書きは1〜2文で簡潔に
- 重要度順に並べる
- 英語の動画でも要約は日本語で書く
- 動画の長さや密度に応じて、論理的なまとまりごとに `### 小見出し` でセクション化してよい

#### 5.1 ハイライト時刻の選定（品質保証）

「勘で選ぶ」ことを避け、**権威ある 3 ソースの優先順位**で時刻を決める。最終ハイライト = 各ソースの和集合を**重複排除・時刻順**で全採用（**上限なし**）。長尺動画（1時間超など）では数十枚に達することがあり、HTML のファイルサイズも比例して大きくなる（base64 埋込で 1 枚あたり 100〜200KB 想定）。

| 優先 | ソース | 選び方 |
|---|---|---|
| 1 | `heatmap`（Most Replayed） | `value` 降順の上位ピーク。区間中点を時刻に。**ユーザー注目度をそのまま反映**。ピークが近接1区間に密集する場合は代表1枚に間引く（全部採ると同じ場面ばかりになる） |
| 2 | `chapters` | 各章の代表フレーム（先頭は除外可: Digest/Ending）。figcaption に章タイトルを使う |
| 3 | timed transcript のキーフレーズ | 章が長く複数スライドを含む箇所を補完（例: UI 画面・効果数値・アーキ図） |

- **トランジション回避**: 区間の先頭そのものではなく **start + 3 秒**（heatmap はピーク中点）を抽出時刻にする。
- 各ハイライトに `(時刻秒, キャプション, ソース種別)` を持たせ、**どの要約セクションに対応するか**もマッピングする（§7.2 で記事中インライン配置に使う）。ソース種別は `Most Replayed` / `章` / `本編`。
- 抽出後は **必ず `Read` ツールで各 JPEG を目視確認**し、内容と一致するようキャプションを実画像に合わせて確定する（スライドでなく登壇者が映る場合は正直にそう書く）。

#### 5.5 ハイライト・フレーム抽出（base64）

「画像不要」指示時はスキップ。`ffmpeg` 必須（無ければスキップしテキストのみ）。

**720p の直 URL を取得 → ffmpeg の入力シークでフレーム抽出**（HTTP Range で部分取得、フル DL 不要）:

```bash
VURL=$(yt-dlp -f "bestvideo[height<=720][ext=mp4]/bestvideo[height<=720]/best[height<=720]" -g "$URL" 2>/dev/null | head -1)
echo "$VURL" > "$TMPDIR/vurl.txt"
```

`$TMPDIR/highlights.txt` に `idx|sec|caption|source` を 1 行ずつ用意し、抽出:

```bash
extract_one() {  # $1=idx $2=sec   ★ -nostdin と </dev/null が必須（環境前提 §5）
  ffmpeg -nostdin -hide_banner -loglevel error -ss "$2" -i "$(cat "$TMPDIR/vurl.txt")" \
    -frames:v 1 -q:v 2 -y "$TMPDIR/frame_$1.jpg" </dev/null 2>/dev/null
}
while IFS='|' read -r idx sec cap src; do
  extract_one "$idx" "$sec"
  sz=$(stat -f%z "$TMPDIR/frame_$idx.jpg" 2>/dev/null || echo 0)
  # 品質ガード: 黒/単色フレームは極小 JPEG になる。<8KB なら ±2 秒ずらして救済
  if [ "${sz:-0}" -lt 8000 ]; then
    extract_one "$idx" $((sec+2)); sz=$(stat -f%z "$TMPDIR/frame_$idx.jpg" 2>/dev/null || echo 0)
    [ "${sz:-0}" -lt 8000 ] && extract_one "$idx" $((sec-2))
  fi
done < "$TMPDIR/highlights.txt"
```

- `-ss` を `-i` の前に置く（高速なキーフレームシーク）。`-q:v 2` は JPEG 高品質（720p で 1 枚 100〜200KB）。
- **フォールバック**: `VURL` が空 or ffmpeg が全滅したら `yt-dlp -f "bestvideo[height<=720]/best[height<=720]" -o "$TMPDIR/video.%(ext)s" "$URL"` でローカル保存後、`ffmpeg -nostdin -ss "$sec" -i "$TMPDIR/video."* ...` で抽出。
- **base64 化**は HTML 生成スクリプト内で `base64.b64encode(open(path,'rb').read())` を使い `data:image/jpeg;base64,...` に埋め込む（§7.2）。
- 抽出した JPEG は `Read` で目視確認し、整合性チェック（先頭 `\xff\xd8`・末尾 `\xff\xd9`）も任意で実施。

#### 5.6 コンタクトシートで一括点検 → 重複・話者のみコマを差し替え（必須）

機械選定した時刻は **(a) 同じ作例写真の使い回しで視覚的に重複** したり、**(b) スライド/作例ではなく登壇者ショットだけ** に当たることが多い（特に講義・解説・トーク動画）。1 枚ずつ `Read` するより、**全フレームをタイル化した 1 枚で一括点検**するのが速い：

```bash
# 全フレームを 1 枚にタイル化（例: 10枚なら 2x5、20枚なら 4x5、30枚なら 5x6 等、N に応じてレイアウト）。Read で俯瞰し重複/話者コマを特定
ffmpeg -nostdin -hide_banner -loglevel error \
  -i "$TMPDIR/frame_1.jpg" ... -i "$TMPDIR/frame_N.jpg" \
  -filter_complex "[0][1]...[N-1]xstack=inputs=N:layout=0_0|w0_0|...,scale=1600:-1[v]" \
  -map "[v]" -q:v 3 -y "$TMPDIR/contact.jpg" </dev/null 2>/dev/null
```

点検後の対応：
- **視覚的に重複**したコマは片方を捨て、別の論点・別の作例が映る時刻へ差し替える。
- **話者のみ**のコマは、近傍で作例/スライドが映る時刻を `timed transcript` から探して再抽出（候補を数点まとめて抽出→もう一度コンタクトシートで確認すると速い）。
- 確定後、**キャプションは実画像に厳密に合わせる**（推測でなく、映っているもので書く）。
- 採用フレームを `h_1.jpg` … の連番にコピーしてから HTML/MD 生成に渡すと、捨てたコマと混ざらない。

### 6. ファイル保存（MD + HTML 同時出力）

出力先（拡張子のみ異なる2ファイル）：
- `${OUTPUT_DIR:-./output}/youtube/YYYYMMDD-<video_id>-<sanitized-title>.md`
- `${OUTPUT_DIR:-./output}/youtube/YYYYMMDD-<video_id>-<sanitized-title>.html`

- `YYYYMMDD` は動画の投稿日（メタデータの `upload_date`）
- `<video_id>` はURLから抽出した11文字のID（一意性を保証）
- `<sanitized-title>` はタイトルから生成：
  - ファイル名に使えない文字（`/ \ : * ? " < > |`）を除去
  - スペースを `-` に置換
  - 40文字以内に切り詰め（日本語はそのまま使用可）

例: `20260125-CulYLbtdbMg-効率2倍の暗記術.{md,html}`

ディレクトリが存在しない場合は作成：
```bash
mkdir -p ${OUTPUT_DIR:-./output}/youtube
```

### 7. 出力フォーマット

#### 7.1 Markdown（`.md`）

```markdown
# <動画タイトル>

- **URL**: <YouTube URL>
- **チャンネル**: <チャンネル名>
- **投稿日**: YYYY-MM-DD
- **再生時間**: <duration_string>

## 目次

- [要約](#要約)
  - [<小見出し1>](#小見出し1)
  - [<小見出し2>](#小見出し2)
- [文字起こし（全文）](#文字起こし全文)

## 要約

### <小見出し1>

🖼 `04:12` 目標設定チャットの UI 画面 / `07:27` 裏側アーキテクチャ

- 箇条書き1
- 箇条書き2

### <小見出し2>

🖼 `12:54` 効果：約3割減・118時間

- 箇条書き1
- ...

## 文字起こし（全文）

<全文テキスト>
```

> MD には base64 画像を入れない。各小見出しの**直下**に `🖼 \`時刻\` 該当スライド` の参照行を添える（画像本体は HTML 版で見出し直下にインライン埋め込み）。

**目次（`## 目次`）はメタデータ直後・`## 要約` の前に置く**。`##`/`###` 見出しから GitHub 互換スラッグ（小文字化、`：｜（）・、。,./／` 等の記号除去、空白→`-`）でアンカーリンクを生成し、`##` をトップ・`###` をネストした箇条書きにする。`目次` 見出し自体は除外。

#### 7.2 HTML（`.html`）

MDと同じ内容を読みやすいスタイル付きHTMLで出力する。**文字起こしブロック内のテキストはHTMLエスケープ**（`& → &amp;`, `< → &lt;`, `> → &gt;`）し、`<div class="transcript">` でスクロール可能なコンテナに格納する。

テンプレート骨子（変数は MD と同じソースから埋め込む）：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
<meta charset="UTF-8">
<title>{TITLE}</title>
<script>(function(){try{var t=localStorage.getItem('yt-theme')||(matchMedia('(prefers-color-scheme: dark)').matches?'dark':'light');document.documentElement.setAttribute('data-theme',t);}catch(e){}})();</script>
<style>
  :root {
    --color-bg: #fbf8f2;
    --color-surface: #fefdf8;
    --color-text: #3b3a36;
    --color-muted: #807c73;
    --color-accent: #45433f;
    --color-line: #efeae0;
    --font-serif: "Hiragino Mincho ProN", "Yu Mincho", serif;
    --font-sans: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Yu Gothic", sans-serif;
  }
  html[data-theme="dark"] {
    --color-bg: #2b2926;
    --color-surface: #34322e;
    --color-text: #cdc9c1;
    --color-muted: #948f86;
    --color-accent: #d9d5cd;
    --color-line: #45413b;
  }
  * { box-sizing: border-box; }
  body { margin: 0; background: var(--color-bg); color: var(--color-text);
    font-family: var(--font-sans); line-height: 1.75; font-size: 16px;
    transition: background .25s ease, color .25s ease; }
  .wrap { max-width: 760px; margin: 0 auto;
    padding: clamp(2rem, 4vw, 4rem) clamp(1rem, 4vw, 2rem); }
  header { border-bottom: 2px solid var(--color-accent);
    padding-bottom: 1.5rem; margin-bottom: 2.5rem; }
  h1 { font-family: var(--font-serif);
    font-size: clamp(1.5rem, 3vw, 2.1rem); line-height: 1.35;
    margin: 0 0 1rem; letter-spacing: 0.01em; }
  .meta { display: grid; grid-template-columns: max-content 1fr;
    gap: 0.35rem 1.2rem; font-size: 0.9rem; color: var(--color-muted); }
  .meta dt { font-weight: 600; color: var(--color-text); }
  .meta dd { margin: 0; }
  .meta a { color: var(--color-accent); text-decoration: none; word-break: break-all; }
  .meta a:hover { text-decoration: underline; }
  /* 目次：フォントサイズ・装飾は記事本体と同じ。H タグごとに改行＋インデントし行間のみ詰める */
  .toc { font-size: 1rem; color: var(--color-text); margin-bottom: 1.6rem;
    padding-bottom: 0.6rem; border-bottom: 1px solid var(--color-line); }
  .toc .toc-title { font-weight: 700; color: var(--color-text); margin: 0 0 0.2rem; }
  .toc ul { list-style: none; margin: 0; padding: 0; line-height: 1.5; }
  .toc ul ul { padding-left: 1.1rem; margin: 0; }
  .toc li { margin: 0; }
  .toc a { color: var(--color-text); text-decoration: none; }
  .toc a:hover { color: var(--color-accent); text-decoration: underline; }
  :target { scroll-margin-top: 1rem; }
  h2 { font-family: var(--font-serif); font-size: 1.5rem;
    margin: 3rem 0 1rem; padding-bottom: 0.4rem;
    border-bottom: 1px solid var(--color-line); }
  h3 { font-size: 1.1rem; margin: 2rem 0 0.7rem; color: var(--color-accent); }
  ul { padding-left: 1.4rem; margin: 0.5rem 0 1.5rem; }
  li { margin-bottom: 0.5rem; }
  li > ul { margin-top: 0.5rem; margin-bottom: 0.5rem; }
  strong { font-weight: 700; }
  /* 記事中インライン図版（要約セクション内に配置） */
  figure.shot { margin: 1.2rem 0 1.8rem; background: var(--color-surface);
    border: 1px solid var(--color-line); border-radius: 8px; overflow: hidden;
    box-shadow: 0 1px 3px rgba(0,0,0,0.06); }
  /* 図版は広視野でのみ記事カラムより左右に 10% ずつはみ出す（狭小画面では横スクロール防止のため等幅） */
  @media (min-width: 1100px) { figure.shot { width: 120%; margin-left: -10%; margin-right: -10%; } }
  figure.shot img { width: 100%; height: auto; display: block; }
  figure.shot figcaption { padding: 0.6rem 0.9rem; font-size: 0.84rem;
    color: var(--color-muted); line-height: 1.55; border-top: 1px solid var(--color-line); }
  figure.shot .ts { color: var(--color-accent); font-weight: 700; margin-right: 0.5rem;
    font-variant-numeric: tabular-nums; }
  figure.shot .src { display: inline-block; margin-left: 0.5rem; padding: 0.05rem 0.45rem;
    font-size: 0.72rem; color: var(--color-muted); background: var(--color-bg);
    border: 1px solid var(--color-line); border-radius: 999px; vertical-align: middle; }
  .transcript { background: var(--color-surface); border: 1px solid var(--color-line);
    border-radius: 6px; padding: 1.5rem; margin-top: 1rem;
    font-size: 0.9rem; line-height: 1.85; color: var(--color-text);
    max-height: 600px; overflow-y: auto;
    white-space: pre-wrap; word-break: break-word; }
  footer { margin-top: 4rem; padding-top: 1.5rem;
    border-top: 1px solid var(--color-line);
    font-size: 0.85rem; color: var(--color-muted); text-align: center; }
  /* テーマトグル：記事ヘッダー直上に右寄せ。アイコンは data-theme で出し分け */
  .theme-bar { display: flex; justify-content: flex-end; margin-bottom: 0.75rem; }
  .theme-toggle { width: 2.4rem; height: 2.4rem; display: inline-flex;
    align-items: center; justify-content: center; padding: 0;
    border: 1px solid var(--color-line); border-radius: 999px;
    background: var(--color-surface); color: var(--color-text); cursor: pointer;
    font-size: 1.15rem; line-height: 1;
    transition: background .2s, border-color .2s, color .2s; }
  .theme-toggle:hover { border-color: var(--color-accent); color: var(--color-accent); }
  .theme-toggle .i-sun { display: none; }
  .theme-toggle .i-moon { display: inline; }
  html[data-theme="dark"] .theme-toggle .i-sun { display: inline; }
  html[data-theme="dark"] .theme-toggle .i-moon { display: none; }
</style>
</head>
<body>
<div class="wrap">
  <div class="theme-bar">
    <button id="theme-toggle" class="theme-toggle" type="button" aria-label="ライト/ダークモード切替" aria-pressed="false"><span class="i-sun">☀</span><span class="i-moon">☾</span></button>
  </div>
  <header>
    <h1>{TITLE}</h1>
    <dl class="meta">
      <dt>URL</dt><dd><a href="{URL}" target="_blank" rel="noopener">{URL}</a></dd>
      <dt>チャンネル</dt><dd>{CHANNEL}</dd>
      <dt>投稿日</dt><dd>{DATE}</dd>
      <dt>再生時間</dt><dd>{DURATION}</dd>
    </dl>
  </header>

  <!-- 目次：H タグごとに改行・インデント。h2 トップ / h3 ネスト。行間は CSS で詰める -->
  <nav class="toc" aria-label="目次">
    <p class="toc-title">目次</p>
    <ul>
      <li><a href="#sec-1">要約</a>
        <ul>
          <li><a href="#sub-1-1">{小見出し1}</a></li>
          <!-- ... 各小見出し ... -->
        </ul></li>
      <li><a href="#sec-2">文字起こし（全文）</a></li>
    </ul>
  </nav>

  <section>
    <h2 id="sec-1">要約</h2>
{SUMMARY_HTML}
  </section>

  <section>
    <h2 id="sec-2">文字起こし（全文）</h2>
    <div class="transcript">{TRANSCRIPT_ESCAPED}</div>
  </section>

  <footer>Generated by /yt-summary · {TODAY}</footer>
</div>
<script>(function(){var b=document.getElementById('theme-toggle');if(!b)return;function s(){b.setAttribute('aria-pressed',document.documentElement.getAttribute('data-theme')==='dark');}s();b.addEventListener('click',function(){var n=document.documentElement.getAttribute('data-theme')==='dark'?'light':'dark';document.documentElement.setAttribute('data-theme',n);try{localStorage.setItem('yt-theme',n);}catch(e){}s();});})();</script>
</body>
</html>
```

`{SUMMARY_HTML}` は Markdown 要約を `<h3>` と `<ul><li>` 構造に変換する（外部ライブラリ不要）。各 `<h2>`/`<h3>` には連番 id（`sec-N` / `sub-N-M`）を付与し、`<header>` 直後の `<nav class="toc">` から `#id` でリンクする。**目次のフォントサイズ・装飾（色）は記事本体と同じ**（`font-size: 1rem`・`--color-text`）。**H タグごとに改行＋インデント**（`h2` トップ・`h3` ネスト）するが、`line-height` と `margin`/`padding` を最小化して**行間の余白を詰める**。`:target { scroll-margin-top }` でアンカー位置を微調整。

**ハイライト画像は見出し直下に配置**：並び順は `<h3 id="...">` → `<figure class="shot">`（そのセクションに対応する画像を 1 つ以上）→ `<ul>` の順とし、独立ギャラリーは作らない。図版テンプレート：

```html
    <figure class="shot">
      <img src="data:image/jpeg;base64,{BASE64}" alt="{CAPTION}" loading="lazy">
      <figcaption><span class="ts">{MMSS}</span>{CAPTION}<span class="src">{SOURCE}</span></figcaption>
    </figure>
```

**テーマトグル（必須）**：配色は `:root`（ライト）と `html[data-theme="dark"]`（ダーク）の
CSS 変数で完結させる。`<head>` の `<title>` 直後に anti-flash スクリプト（localStorage の
`yt-theme` → なければ OS の `prefers-color-scheme` の順で初期テーマを解決し `data-theme` を
即設定）を必ず置く。トグルは `.theme-bar`（右寄せ）として `<div class="wrap">` 直下・
`<header>` の**直上**に配置し、`</body>` 直前のスクリプトでクリック反転＋localStorage 保存＋
`aria-pressed` 同期を行う。

**配色方針**：白黒グレーのモノクロ基調（赤・オレンジ系は使わない）。ライトは暖色寄りの
クリーム系オフホワイト、ダークは**純黒を避けた暖色寄りダークグレー**（背景を持ち上げ、文字は
明度を抑えた灰白）にし、コントラストは約7:1前後の中〜低めに抑える。純黒×純白の高コントラストは
halation（文字の滲み・チカチカ）の原因になるため避ける。`.transcript` 等のハードコード色は
使わず必ず変数（`var(--color-text)` など）経由にする。

**図版のはみ出し**：`figure.shot` はビューポート幅 1100px 以上のときだけ `width:120%` /
`margin-left/right:-10%` で記事カラムより左右にせり出す。狭小画面では等幅に戻し横スクロールを防ぐ。

### 8. クリーンアップ

一時ディレクトリを削除：
```bash
rm -rf "$TMPDIR"
```

### 8.5 reference インデックス再生成

HTML 保存後、`${OUTPUT_DIR:-./output}/index.html`（全資料のリンク集）を再生成する。失敗しても本処理は完了扱い（非ブロッキング）：

```bash
python3 # (optional) your own index rebuild script 2>/dev/null || true
```

### 9. 完了報告

保存した **両方** のファイルパス（`.md` と `.html`）、埋め込んだハイライト画像の枚数、
および再生成した `${OUTPUT_DIR:-./output}/index.html` のパスを表示して完了。

## エラーハンドリング

- **URL未指定**: 「YouTube URLを引数として指定してください。例: `/yt-summary https://www.youtube.com/watch?v=XXXXX`」
- **yt-dlp未インストール**: 「yt-dlpがインストールされていません。`brew install yt-dlp` でインストールしてください。」
- **字幕なし**: 「この動画には利用可能な字幕がありません。」と表示し、stderrログがあれば内容を確認して原因を補足
- **動画取得失敗**: stderrログの内容を確認し、「動画情報の取得に失敗しました。URLを確認してください。」とともにエラー詳細を報告
- **HTTP 429等のレート制限**: 上記§3 のリトライロジックで自動再試行。二度目も失敗した場合のみユーザーに明示
- **impersonation warning**: yt-dlp が `WARNING: The extractor specified to use impersonation ...` を出すのは無視してよい。impersonate ターゲットがなくても字幕取得は成功する場合が多い
- **ffmpeg 未インストール / 動画 DL 不可（geo/age 制限等）**: §5.5 のフレーム抽出のみスキップし、ハイライト・セクションを省略。テキストレポート（要約・文字起こし）は通常どおり完成させ、HTML フッターにその旨を注記
- **フレーム抽出が一部失敗**: 取得できたものだけ掲載（最低 1 枚を目標）。全滅時のみハイライト・セクション省略

## トラブルシューティング履歴

過去のセッションで実際に発生した問題と対処（再発防止メモ）：

| 症状 | 原因 | 対処 |
|------|------|------|
| `Writing video subtitles to: sub.ja.srt` と出るのに `ls` で見当たらない | 直後の HTTP 429 でダウンロードが中断。ログは宣言行 | `ls` で実体確認、429 リトライ手順を実行 |
| 2 回目の `ls "$TMPDIR"` で「No such file or directory」 | `TMPDIR` 変数が次の Bash 呼び出しで失われる | `/tmp/yt_tmpdir.txt` に書き出して復元 |
| `ls "$TMPDIR"/sub.ja.srt "$TMPDIR"/sub.ja.vtt ...` で zsh が `no matches found` で停止 | グロブ展開失敗 | `for cand in ...; do [ -f "$cand" ] && ...; done` で個別チェック |
| 日本語字幕で連続 429、英語に切替えると即座に成功 | 同じ動画でも言語別にレート制限が独立カウントされる | フォールバック順序を `ja → en (auto)` にし、`--sleep-subtitles 5` 以上を併用 |
| §5.5 の `while read` ループでフィールド (idx/sec) がずれて壊れる | ループ内 `ffmpeg` がループの stdin を消費 | `ffmpeg -nostdin ... </dev/null` を必ず付ける |
| 章境界 +3 秒のフレームが登壇者で、スライドが映っていない | プレゼン動画はカメラがスライドと登壇者を切替。章境界は登壇者ショットになりがち | スライドが要るハイライトは timed transcript のキーフレーズ時刻を使う。抽出後 `Read` で目視確認しキャプションを実画像に合わせる |
| 別々の時刻なのに同じ作例写真が映って重複 / 一部が話者のみ | 動画内で同じ写真を使い回す・カメラが話者へ切替 | §5.6 のコンタクトシートで一括点検し、重複コマ・話者コマを別時刻へ差し替える（近接 heatmap ピークは代表1枚に間引く） |
| キャプションと実画像が食い違う（例: スーツ写真なのに「カジュアル」と記載） | 機械選定の想定と実映像のタイミングがずれた | 抽出後に必ず目視。キャプションは推測でなく**映っているもの**で書く |
| `heatmap` が `None` | 投稿直後や再生数の少ない動画は Most Replayed 未生成 | `chapters` → timed transcript の順でフォールバック |
