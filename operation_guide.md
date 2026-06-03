# ラテカピュータプロジェクト 運用ガイド

**最終更新：2026-06-03**

---

## 1. リポジトリ構成

| リポジトリ | 公開設定 | 用途 |
|-----------|---------|------|
| `mushipan3/ratecaputer-specs` | **パブリック** | 目論見書・仕様書（AI参照用） |
| `mushipan3/ratecaputer` | プライベート | ソースコード・回路図・CAD |

---

## 2. 仕様書ファイル一覧（正式ファイル名）

ファイル名は以下で固定。AIへの指示・GitHubへのアップロード時も必ずこの名前を使うこと。

| ファイル | 場所 | 内容 | 状態 |
|---------|------|------|------|
| `operation_guide.md` | ルート直下 | 本ドキュメント | ✅ 作成済み |
| `README.md` | ルート直下 | URL一覧 | ✅ 作成済み |
| `specs/master.md` | specsフォルダ内 | マスター仕様書 | ✅ 作成済み |
| `specs/body.md` | specsフォルダ内 | 本体目論見書 | 🔲 未作成 |
| `specs/keyboard.md` | specsフォルダ内 | キーボード目論見書 | 🔲 未作成 |
| `specs/cartridge.md` | specsフォルダ内 | カートリッジ目論見書 | 🔲 未作成 |
| `specs/toy.md` | specsフォルダ内 | トイ目論見書 | 🔲 未作成 |
| `specs/nano.md` | specsフォルダ内 | ナノ目論見書 | 🔲 未作成 |

---

## 3. GitHubへの反映手順（ブラウザ操作）

**ダウンロードは使わない。コピー＆ペーストで直接編集すること。**
これによりファイル名の齟齬を防ぐ。

### 3.1 既存ファイルを更新する場合

1. ブラウザで `github.com/mushipan3/ratecaputer-specs` を開く
2. 該当ファイルをクリック
3. 右上の鉛筆アイコン（Edit this file）をクリック
4. 内容を全選択（Ctrl+A）して削除
5. Claudeが生成した内容をコピーして貼り付け
6. 画面下の「Commit changes」をクリック
7. コミットメッセージを入力して「Commit changes」

### 3.2 新規ファイルを作成する場合

1. ブラウザで `github.com/mushipan3/ratecaputer-specs` を開く
2. 「Add file」→「Create new file」をクリック
3. ファイル名欄に正式ファイル名を入力
   - specsフォルダ内の場合：`specs/`と入力するとフォルダが自動生成される
   - 例：`specs/cartridge.md`
4. Claudeが生成した内容をコピーして貼り付け
5. 「Commit changes」をクリック
6. コミットメッセージを入力して「Commit changes」

---

## 4. 日常の作業フロー

### 4.1 仕様検討・目論見書の更新（スマホ/iPad/PC）

```
① claude.aiまたはGeminiでチャットして仕様を検討・決定

② AIが更新版ファイルを生成
   → 画面上の内容をコピー

③ GitHubブラウザで該当ファイルを開いて編集・貼り付け・Commit
   （3.1または3.2の手順）

④ 次回のAIチャット冒頭で最新仕様として参照できる
```

### 4.2 開発作業（PC・VSCode中心）

```
① VSCodeでratecaputerフォルダを開く

② Claude CodeまたはCodexに作業指示の前に最新仕様を取得させる：
   「specs/master.mdとspecs/[デバイス].mdを読んでから作業してください」
   → AGENTS.mdの指示に従いcurlで自動取得

③ 開発・コード生成

④ 内容確認してOKを出したタイミングでpushを指示：
   「GitHubにpushしてください」
   → Claude Code/Codexが以下を実行：
      git add .
      git commit -m "[デバイス名] 変更内容"
      git push
```

### 4.3 複数AIの使い分け

| AI | 得意な作業 | 仕様書参照方法 |
|---|-----------|--------------|
| claude.ai | 仕様検討・目論見書作成・設計議論 | 内容をコピーして貼り付け |
| Gemini | 仕様検討・先行議論 | raw URLを渡してfetch |
| Claude Code | ソースコード生成・修正 | curlで自動fetch |
| Codex | ソースコード生成・修正 | curlで自動fetch |

---

## 5. 新しい目論見書を作るとき

```
① claude.aiで該当デバイスの仕様を議論して決定

② Claudeに「[デバイス名]の目論見書を生成してください」と依頼

③ 生成された内容をコピー

④ GitHubブラウザで「Add file」→「Create new file」
   ファイル名：specs/[正式ファイル名]（例：specs/cartridge.md）
   内容を貼り付け→Commit

⑤ specs/master.mdの未決事項欄を更新
   → 同様にGitHubブラウザで編集・Commit
```

---

## 6. 仕様が変更になったとき

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

## 7. AIチャットを新たに起こすときの冒頭テンプレート

新しいチャットを始めるときに以下を冒頭に貼り付ける：

```
以下のURLにあるマスター仕様書を読んでからこのチャットを始めてください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/master.md

今回は[デバイス名]についての作業です。以下も読んでください。
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/specs/[ファイル名].md
```

---

## 8. コミットメッセージ規則

| プレフィックス | 用途 | 例 |
|-------------|------|---|
| `[body]` | 本体関連 | `[body] ESP32-C6 IR制御実装` |
| `[kbd]` | キーボード関連 | `[kbd] キースキャン修正` |
| `[ctrg]` | カートリッジ関連 | `[ctrg] ブートローダー初期実装` |
| `[toy]` | トイ関連 | `[toy] DAC音声出力実装` |
| `[nano]` | ナノ関連 | `[nano] NeoPixel駆動実装` |
| `[specs]` | 仕様書関連 | `[specs] master.md システム構成修正` |

---

## 9. Raw URL一覧（AIに渡す用）

```
運用ガイド：
https://raw.githubusercontent.com/mushipan3/ratecaputer-specs/main/operation_guide.md

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

## 10. トラブルシューティング

| 症状 | 対処 |
|------|------|
| AIが古い仕様で回答する | 最新のmaster.mdを貼り付けて「これが最新仕様です」と伝える |
| GitHubブラウザで編集できない | ログイン状態を確認する |
| Claude Codeがcurlでfetchできない | ネットワーク設定確認、URLのtypo確認 |
| embedded git repositoryの警告 | `git rm --cached [フォルダ名]`で解除後に`.git`を削除 |
