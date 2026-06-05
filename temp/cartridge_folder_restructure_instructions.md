# カートリッジフォルダ再構成 作業指示

**作成日：2026-06-05**
**作業環境：Claude Desktop（GitHub MCP）**

---

## あなたはratecaputer-specsリポジトリの管理者です。
以下を作業開始時に必ず実行してください：
1. mushipan3/ratecaputer-specsのファイル構成を確認する
2. temp/フォルダに新しいファイルがあれば報告する

---

## 今回の作業

`specs/cartridge/` 配下のファイルを以下の新フォルダ構成に従って移動・再配置してください。
GitHubにはファイル移動APIがないため、**新パスに作成→旧パスを削除**の手順で実施すること。

---

## 新フォルダ構成

```
specs/cartridge/
  ハード/
    A_board/
    B_board/
    C_board/
    D_board/
  ソフト/
    UIAPduino/
```

---

## ファイル移動対応表

### specs/cartridge/ハード/ → 各基板フォルダへ

| 現在のファイル名 | 移動先 |
|----------------|--------|
| `A_board_spec_20260601.md` | `specs/cartridge/ハード/A_board/` |
| `a_board_power_spec_20260527.md` | `specs/cartridge/ハード/A_board/` |
| `CAR-01_B-board_spec.md` | `specs/cartridge/ハード/B_board/` |
| `c_board_storage_spec_20260527.md` | `specs/cartridge/ハード/C_board/` |
| `d_board_wnn_prospectus_20260601.md` | `specs/cartridge/ハード/D_board/` |
| `car01_enclosure_layout_prospectus_right_20260601.md` | `specs/cartridge/ハード/D_board/` |
| `panel_order_spec_20260527.md` | `specs/cartridge/ハード/`（直下に残す） |
| `handoff_to_integrated_pcb_chat_20260601.md` | `specs/cartridge/ハード/`（直下に残す） |

### specs/cartridge/ソフト/ → UIAPduinoフォルダへ

以下の全ファイルを `specs/cartridge/ソフト/UIAPduino/` に移動：

- `cartridge_spec_master_20260522.1.md`
- `bootloader_dev_spec_20260522.1.md`
- `claude_code_instructions_20260522.1.md`
- `claude_code_bootloader_instructions_20260522.1.md`
- `emulator_dev_prospectus_20260601.md`
- `engine_design_prospectus_20260601.md`
- `novel_game_port_spec_20260520.1.md`
- `ratecaputer_cartridge_prospectus_20260601.md`
- `test_environment_spec_20260520.1.md`
- `latcaputa_wnn_prospectus_rev2.docx`

---

## 開発環境フォルダの新規作成

以下の空フォルダを `.gitkeep` で作成：

```
開発環境/
  デバイス開発者向け/
    cartridge/
  アプリ開発者向け/
    cartridge/
```

---

## operation_guide.md の更新

`specs/直下の構成` セクションを以下に更新する：

```
specs/
  master.md
  connector.md
  body/
    ハード/
    ソフト/
  keyboard/
    ハード/
    ソフト/
  cartridge/
    ハード/
      A_board/
      B_board/
      C_board/
      D_board/
    ソフト/
      UIAPduino/
  toy/
    ハード/
    ソフト/
  nano/
    ハード/
    ソフト/
開発環境/
  デバイス開発者向け/
    cartridge/
  アプリ開発者向け/
    cartridge/
temp/
logs/
docs/
```

---

## logs/decisions.md への追記

2026-06-05付けで以下を追記：

### specs/cartridge/ フォルダ構成確定

- **ハード**：共通基板群をA〜D基板ごとにフォルダ分け
  - A基板（電源管理）・B基板（FM音源）・C基板（ストレージ）は全機種共通
  - D基板（WNN・筐体依存）はFRISK向け
  - パネル発注仕様・横断ドキュメントはハード直下
- **ソフト**：CPU種別でフォルダ分け（現在はUIAPduinoのみ）
  - 将来的にCPU共通部分を切り出す方針
- **開発環境**フォルダをルート直下に新設
  - デバイス開発者向け・アプリ開発者向けの2系統

---

## コミットメッセージ

```
[specs] cartridgeフォルダ構成を確定・ファイル移動・開発環境フォルダ新設
```

---

## 作業完了後

このファイル（`temp/cartridge_folder_restructure_instructions.md`）を削除すること。
