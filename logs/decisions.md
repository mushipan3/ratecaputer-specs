# ラテカピュータプロジェクト 決定事項ログ

## このファイルの役割

全体構成の変更・複数ドキュメントにまたがる決定事項を時系列で記録する。
個別デバイスの仕様変更は各 `specs/[デバイス名]/master.md` の変更履歴に記録すること。

---

## 2026-06-04

### シリーズ名称確定

各デバイスの正式シリーズ名を以下に確定した。

| シリーズ名 | デバイス | MCU |
|-----------|---------|-----|
| ラテカキーボード | キーボード | ESP32-S3 |
| ラテカカートリッジ | カートリッジ | CH32V003 |
| ラテカトイ | トイ | AVR64DD14 |
| ラテカナノ | ナノ | ATtiny10 |

### specs/フォルダ構成確定

各デバイスの仕様書を `specs/[デバイス名]/master.md` に配置する構成に確定した。
旧構成（`specs/body.md` 等のフラットな配置）から移行済み。

```
specs/
  master.md
  connector.md
  body/master.md
  keyboard/master.md
  cartridge/master.md
  cartridge/bootloader.md
  cartridge/novel_game.md
  toy/master.md
  nano/master.md
temp/
logs/
```

### GitHub連携管理機構の構築完了

以下の環境・役割分担を確立した。

| 環境 | 役割 | GitHub操作 |
|------|------|-----------|
| claude.ai（スマホ・PC） | 仕様検討・生成 | temp/に手動アップロード |
| Claude Desktop（PC） | 整合性確認・正規化・specs/に登録 | GitHub MCP直接読み書き |
| Claude Code（PC） | 開発 | curl読み取り・git push |
| Codex（PC） | 開発 | curl読み取り・git push |

関連ドキュメント：
- `operation_guide.md` — 日常フロー・環境別対応表
- `claude_desktop_github_mcp_setup.md` — Claude Desktop MCP設定手順
- `ratecaputer` リポジトリ各 `AGENTS.md` — curlパス設定

### docs/フォルダ追加決定

リポジトリ全体に関わる補足ドキュメント（セットアップ手順・参考資料等）を格納する `docs/` フォルダを追加することを決定した。

### logs/decisions.mdの役割定義

本ファイル（`logs/decisions.md`）を以下の方針で運用することを決定した。

- **記録対象**：全体構成の変更・複数ドキュメントにまたがる決定事項
- **記録対象外**：個別デバイスの仕様変更（各 `master.md` の変更履歴に記録）
- **記録フォーマット**：日付セクション → 決定事項ごとの見出し → 内容
