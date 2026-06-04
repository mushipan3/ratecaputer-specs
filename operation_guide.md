# ラテカピュータプロジェクト 運用ガイド

**最終更新：2026-06-04**

---

## 1. リポジトリ構成

| リポジトリ | 公開設定 | 用途 |
|-----------|---------|------|
| `mushipan3/ratecaputer-specs` | **パブリック** | 目論見書・仕様書（AI参照用） |
| `mushipan3/ratecaputer` | プライベート | ソースコード・回路図・CAD |

---

## 2. 環境別GitHub対応表

| 環境 | 役割 | GitHub操作 |
|------|------|-----------|
| claude.ai（スマホ・PC） | 仕様検討・生成 | temp/に手動アップロード |
| Claude Desktop（PC） | 整合性確認・正規化・specs/に登録 | GitHub MCP直接読み書き |
| Claude Code（PC） | 開発 | curl読み取り・git push |
| Codex（PC） | 開発 | curl読み取り・git push |

---

## 3. specs/フォルダ構成ルール

specs/直下の構成：

```
specs/
  master.md            ← 全体マスター仕様書
  connector.md         ← 拡張コネクタ共通仕様
  body/
    master.md          ← 本体目論見書
  keyboard/
    master.md          ← キーボード目論見書
  cartridge/
    master.md          ← カートリッジ目論見書
    bootloader.md      ← ブートローダー仕様
    novel_game.md      ← ノベルゲーム仕様
  toy/
    master.md          ← トイ目論見書
  nano/
    master.md          ← ナノ目論見書
temp/                  ← 作業中ファイル置き場（claude.aiからの一時ファイル）
```

各デバイスのメイン仕様書は必ず `specs/[デバイス名]/master.md` に配置すること。

---

## 4. 仕様書ファイル一覧（正式ファイルパス）

ファイルパスは以下で固定。AIへの指示・GitHubへのアップロード時も必ずこのパスを使うこと。

| ファイル | 内容 | 状態 |
|---------|------|------|
| `operation_guide.md` | 本ドキュメント | ✅ 作成済み |
| `README.md` | URL一覧 | ✅ 作成済み |
| `specs/master.md` | マスター仕様書 | ✅ 作成済み |
| `specs/connector.md` | 拡張コネクタ共通仕様 | ✅ 作成済み |
| `specs/body/master.md` | 本体目論見書 | 🔲 未作成 |
| `specs/keyboard/master.md` | キーボード目論見書 | 🔲 未作成 |
| `specs/cartridge/master.md` | カートリッジ目論見書 | 🔲 未作成 |
| `specs/cartridge/bootloader.md` | ブートローダー仕様 | 🔲 未作成 |
| `specs/cartridge/novel_game.md` | ノベルゲーム仕様 | 🔲 未作成 |
| `specs/toy/master.md` | トイ目論見書 | 🔲 未作成 |
| `specs/nano/master.md` | ナノ目論見書 | 🔲 未作成 |

---

## 5. 日常の作業フロー

### 5.1 スマホ・iPad（claude.ai）

```
① claude.aiで仕様検討・目論見書生成

② 生成した内容をコピー

③ GitHubブラウザ/アプリで temp/ に貼り付け・保存
   例：temp/cartridge_draft.md
```

### 5.2 PC：Claude Desktop

```
① temp/ を確認
   → 新しいファイルがあればレビュー

② 既存 specs/ と整合性確認

③ specs/ の適切な場所に移動・登録
   例：temp/cartridge_draft.md → specs/cartridge/master.md

④ 関連ファイルを更新（master.md等）

⑤ temp/ から削除
```

### 5.3 PC：Claude Code / Codex

```
① GitHubから最新仕様をcurlで取得
   → AGENTS.mdの指示に従い自動取得

② 開発・コード生成

③ 内容確認後にpush：
   git add .
   git commit -m "[デバイス名] 変更内容"
   git push
```

---

## 6. Claude Desktop 操作メモ

### 完全終了方法

通常の×ボタンではタスクトレイに常駐するため、設定変更後は以下で完全終了すること：

```
taskkill /F /IM claude.exe
```

その後、Claude Desktop を再起動すると設定が反映される。

---

## 7. GitHubへの反映手順（ブラウザ操作）

**ダウンロードは使わない。コピー＆ペーストで直接編集すること。**
これによりファイル名の齟齬を防ぐ。

### 7.1 既存ファイルを更新する場合

1. ブラウザで `github.com/mushipan3/ratecaputer-specs` を開く
2. 該当ファイルをクリック
3. 右上の鉛筆アイコン（Edit this file）をクリック
4. 内容を全選択（Ctrl+A）して削除
5. Claudeが生成した内容をコピーして貼り付け
6. 画面下の「Commit changes」をクリック
7. コミットメッセージを入力して「Commit changes」

### 7.2 新規ファイルを作成する場合

1. ブラウザで `github.com/mushipan3/ratecaputer-specs` を開く
2. 「Add file」→「Create new file」をクリック
3. ファイル名欄に正式ファイルパスを入力
   - 例：`specs/cartridge/master.md`（`/`を入力するとフォルダが自動生成される）
4. Claudeが生成した内容をコピーして貼り付け
5. 「Commit changes」をクリック
6. コミットメッセージを入力して「Commit changes」

---

## 8. 新しい目論見書を作るとき

```
① claude.aiで該当デバイスの仕様を議論して決定

② Claudeに「[デバイス名]の目論見書を生成してください」と依頼

③ 生成された内容をコピー

④ GitHubブラウザで「Add file」→「Create new file」
   ファイル名：specs/[デバイス名]/master.md
   内容を貼り付け→Commit

⑤ specs/master.mdの未決事項欄を更新
   → 同様にGitHubブラウザで編集・Commit
```

---

## 9. 仕様が変更になったとき

```
① 変更内容をClaudeまたはGeminiに伝えて該当ファイルを更新生成

② 生成された内容をコピー

③ GitHubブラウザで該当ファイルを編集・貼り付け・Commit
   コミットメッセージ例：
   「master: システム構成を修正」
   「cartridge: DMA転送仕様を追記」

④ 変更履歴（ファイル末尾のテーブル）にRevを追記する
```

---

## 10. AIチャットを新たに起こすときの冒頭テンプレート

新しいチャットを始めるときに以下を冒頭に貼り付ける：

```
以下のURLにあるマスター仕様書を読んでからこのチャットを始めてください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/master.md

今回は[デバイス名]についての作業です。以下も読んでください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/[デバイス名]/master.md
```

---

## 11. コミットメッセージ規則

| プレフィックス | 用途 | 例 |
|-------------|------|---|
| `[body]` | 本体関連 | `[body] ESP32-C6 IR制御実装` |
| `[kbd]` | キーボード関連 | `[kbd] キースキャン修正` |
| `[ctrg]` | カートリッジ関連 | `[ctrg] ブートローダー初期実装` |
| `[toy]` | トイ関連 | `[toy] DAC音声出力実装` |
| `[nano]` | ナノ関連 | `[nano] NeoPixel駆動実装` |
| `[specs]` | 仕様書関連 | `[specs] master.md システム構成修正` |

---

## 12. Raw URL一覧（AIに渡す用）

```
運用ガイド：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/operation_guide.md

マスター仕様書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/master.md

拡張コネクタ共通仕様：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/connector.md

本体目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/body/master.md

キーボード目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/keyboard/master.md

カートリッジ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/cartridge/master.md

トイ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/toy/master.md

ナノ目論見書：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/nano/master.md
```

---

## 13. トラブルシューティング

| 症状 | 対処 |
|------|------|
| AIが古い仕様で回答する | 最新のmaster.mdを貼り付けて「これが最新仕様です」と伝える |
| GitHubブラウザで編集できない | ログイン状態を確認する |
| Claude Codeがcurlでfetchできない | ネットワーク設定確認、URLのtypo確認 |
| embedded git repositoryの警告 | `git rm --cached [フォルダ名]`で解除後に`.git`を削除 |
| Claude Desktopの設定が反映されない | `taskkill /F /IM claude.exe` で完全終了後に再起動 |
