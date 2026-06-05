# CAR-01規格 C基板（ストレージモジュール）設計仕様書

## 完全確定版 — Claude Code投入用マスタードキュメント

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（C基板）は、CAR-01規格に準拠したストレージモジュール基板である。FRAM（1MB）と外部SRAM（512KB）を搭載し、ゲームセーブデータおよびフレームバッファとして機能する。10mm × 28mmの片面実装スティック形状。

### 機能

- FRAM（FM25V10）: 1MB、SPIインターフェース
- SRAM（IS62WVS5128FBLL）: 512KB、SPIインターフェース
- TFJカードコネクタ（Molex 104031-0811）

---

## 2. 内部接続

| 信号 | 方向 | 接続先 | 説明 |
|---|---|---|---|
| SPI_MOSI | Input | UIAPduino | データ出力 |
| SPI_MISO | Output | UIAPduino | データ入力 |
| SPI_SCK | Input | UIAPduino | クロック |
| SPI_CS_FRAM | Input | UIAPduino | FRAMチップセレクト |
| SPI_CS_SRAM | Input | UIAPduino | SRAMチップセレクト |
| SPI_CS_TF | Input | UIAPduino | TFカードチップセレクト |

---

## 3. 変更履歴

| Rev | 日付 | 内容 |
|---|---|---|
| 1.0 | 2026-05-27 | 初期版 |
