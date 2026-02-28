# OpenClaw ノート同期設計（write → iCloud）

## 🎯 目的
OpenClawのサンドボックス制約を守りつつ、
`document/write` に書き込まれたノートを iCloud（Obsidian）へ高速同期する。

---

# ✅ ディレクトリ構成

```
workspace/
  document/
    read/   → iCloud への symlink（参照専用）
    write/  → 実体（OpenClawが書き込む）
```

OpenClawが書き込む場所：
```
~/.openclaw/workspace/document/write/note/OpenClaw
```

同期先：
```
/Users/tbtsub/Library/Mobile Documents/com~apple~CloudDocs/Obsidian/note/OpenClaw
```

---

# ✅ 実装手順

## ① ディレクトリ作成

```bash
mkdir -p ~/.openclaw/workspace/document/write/note/OpenClaw
mkdir -p ~/.openclaw/workspace/document/read
```

---

## ② read を iCloud にリンク

```bash
ln -s \
"/Users/tbtsub/Library/Mobile Documents/com~apple~CloudDocs/Obsidian" \
~/.openclaw/workspace/document/read/icloud
```

read は参照専用。

---

## ③ fswatch インストール

```bash
brew install fswatch
```

---

## ④ 同期スクリプト作成

`~/bin/openclaw-note-sync.sh`

```bash
#!/bin/zsh

SRC="$HOME/.openclaw/workspace/document/write/note/OpenClaw/"
DST="/Users/tbtsub/Library/Mobile Documents/com~apple~CloudDocs/Obsidian/note/OpenClaw/"

rsync -a --delete "$SRC" "$DST"
```

```bash
chmod +x ~/bin/openclaw-note-sync.sh
```

---

## ⑤ 監視スクリプト

`~/bin/openclaw-note-watch.sh`

```bash
#!/bin/zsh

WATCH_DIR="$HOME/.openclaw/workspace/document/write/note/OpenClaw"

fswatch -o "$WATCH_DIR" | while read; do
  sleep 1
  ~/bin/openclaw-note-sync.sh
done
```

```bash
chmod +x ~/bin/openclaw-note-watch.sh
```

---

## ⑥ launchd 常駐設定

`~/Library/LaunchAgents/com.openclaw.notesync.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>

    <key>Label</key>
    <string>com.openclaw.notesync</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/tbtsub/bin/openclaw-note-watch.sh</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

</dict>
</plist>
```

有効化：

```bash
launchctl load ~/Library/LaunchAgents/com.openclaw.notesync.plist
```

---

# ✅ 特徴

- ✅ 片方向同期（write → iCloud）
- ✅ サンドボックス安全
- ✅ ほぼリアルタイム（約1秒以内）
- ✅ ループしない設計
- ✅ cron不要

---

# ⚠ 注意

- `--delete` は片方向前提
- 双方向リアルタイム同期は行わない
- write 側は常に正とする

---

## 🧠 設計思想

- 書き込み領域と参照領域を分離
- サンドボックスを尊重
- シンプルな片方向同期

安定性を最優先した構成。
