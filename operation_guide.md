# ラテカピュータプロジェクト 運用ガイド

**最終更新：2026-06-03**

---

## 1. リポジトリ構成

| リポジトリ | 公開設定 | 用途 |
|-----------|---------|------|
| `mushipan3/ratecaputer-specs` | **パブリック** | 目論見書・仕様書（AI参照用） |
| `mushipan3/ratecaputer` | プライベート | ソースコード・回路図・CAD |

---

## 2. 仕様書ファイル一覧

| ファイル | 内容 | 状態 |
|---------|------|------|
| `specs/master.md` | マスター仕様書（全体俯瞰） | ✅ 作成済み |
| `specs/body.md` | 本体目論見書 | 🔲 未作成 |
| `specs/keyboard.md` | キーボード目論見書 | 🔲 未作成 |
| `specs/cartridge.md` | カートリッジ目論見書 | 🔲 未作成 |
| `specs/toy.md` | トイ目論見書 | 🔲 未作成 |
| `specs/nano.md` | ナノ目論見書 | 🔲 未作成 |

---

## 3. 日常の作業フロー

### 3.1 仕様検討・目論見書の更新（スマホ/iPad中心）

```
① claude.aiまたはGeminiでチャットして仕様を検討・決定

② AIが更新版ファイルを生成
   → 画面上のダウンロードリンクからファイルを保存

③ GitHubアプリでratecaputer-specsリポジトリを開く
   → specs/ フォルダを開く
   → 該当ファイルを選択して「Edit」または「Upload」
   → 内容を貼り付けまたはファイルを上書き
   → コミットメッセージを入力して「Commit」

④ 次回のAIチャット冒頭で「最新仕様を反映済み」として扱える
```

### 3.2 開発作業（PC・VSCode中心）

```
① VSCodeでratecaputerフォルダを開く

② Claude CodeまたはCodexに作業を指示する前に
   最新仕様を取得させる：
   「specs/master.mdとspecs/[デバイス].mdを読んでから作業してください」
   → AGENTS.mdの指示に従いcurlで自動取得

③ 開発・コード生成

④ ユーザーが内容を確認してOKを出したタイミングでpushを指示：
   「GitHubにpushしてください」
   → Claude Code/Codexが以下を実行：
      git add .
      git commit -m "[デバイス名] 変更内容"
      git push
```

### 3.3 複数AIの使い分け

| AI | 得意な作業 | 仕様書参照方法 |
|---|-----------|--------------|
| claude.ai | 仕様検討・目論見書作成・設計議論 | ファイルを貼り付け |
| Gemini | 仕様検討・先行議論 | raw URLを渡してfetch |
| Claude Code | ソースコード生成・修正 | curlで自動fetch |
| Codex | ソースコード生成・修正 | curlで自動fetch |

---

## 4. 新しい目論見書を作るとき

```
① claude.aiで該当デバイスの仕様を議論して決定

② Claudeに「[デバイス名]の目論見書を生成してください」と依頼
   → Markdownファイルとしてダウンロード

③ GitHubアプリでratecaputer-specsを開く
   → specs/フォルダに新規ファイルとしてアップロード
   → ファイル名：body.md / keyboard.md / cartridge.md / toy.md / nano.md

④ master.mdの「未決事項」欄を更新
   → 同様にGitHubアプリで編集・コミット
```

---

## 5. 仕様が変更になったとき

```
① 変更内容をclaudeまたはGeminiに伝えて該当ファイルを更新生成

② GitHubアプリで該当specs/ファイルを上書き更新
   コミットメッセージ例：
   「master: システム構成を修正」
   「cartridge: DMA転送仕様を追記」

③ 変更履歴（ファイル末尾のテーブル）にRevを追記する
```

---

## 6. AIチャットを新たに起こすときの冒頭テンプレート

新しいチャットを始めるときに以下を冒頭に貼り付ける：

```
以下のURLにあるマスター仕様書を読んでからこのチャットを始めてください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/master.md

また、今回は[デバイス名]についての作業です。以下も読んでください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/[ファイル名].md
```

---

## 7. コミットメッセージ規則

| プレフィックス | 用途 | 例 |
|-------------|------|---|
| `[body]` | 本体関連 | `[body] ESP32-C6 IR制御実装` |
| `[kbd]` | キーボード関連 | `[kbd] キースキャン修正` |
| `[ctrg]` | カートリッジ関連 | `[ctrg] ブートローダー初期実装` |
| `[toy]` | トイ関連 | `[toy] DAC音声出力実装` |
| `[nano]` | ナノ関連 | `[nano] NeoPixel駆動実装` |
| `[specs]` | 仕様書関連 | `[specs] master.md システム構成修正` |

---

## 8. raw URL一覧（AIに渡す用）

```
マスター仕様書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/master.md

本体目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/body.md

キーボード目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/keyboard.md

カートリッジ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/cartridge.md

トイ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/toy.md

ナノ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/nano.md
```

---

## 9. トラブルシューティング

| 症状 | 対処 |
|------|------|
| AIが古い仕様で回答する | 最新のmaster.mdを貼り付けて「これが最新仕様です」と伝える |
| GitHubアプリでアップロードできない | ファイルサイズ確認（25MB以下）、ネットワーク確認 |
| Claude Codeがcurlでfetchできない | ネットワーク設定確認、URLのtypo確認 |
| embedded git repositoryの警告 | `git rm --cached [フォルダ名]`で解除後に`.git`を削除 |
