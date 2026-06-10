# CAR-01規格 C基板（ストレージモジュール）設計仕様書

## 確定版 Rev.1.1 — 2026年6月10日

## ※Rev.1.0（2026-05-27）からの変更箇所は【変更】タグで明示する。

-----

## 1. 実装目論見書

### プロジェクト概要

本ボード（C基板）は、ラテカピュータ（CAR-01規格）のストレージ機能を担う極小ドーターボードである。メインアセット格納用の大容量SPI Flash（128Mbit / 16MB）と、超高速・高寿命なSRAM代替作業領域およびセーブ領域用のSPI FRAM（4Mbit / 512KB）を1枚の基板に統合する。

10mm × 28mmのスティック形状は、A基板（電源管理：9.5×27.8mm）・B基板（FM音源：19×27.8mm）と縦幅を統一しており、3基板をV-CUTのみで切り離せる格子状パネルを構成する。

### 物理仕様・製造要件

|項目    |仕様                                     |
|------|---------------------------------------|
|基板サイズ |9.5mm × 27.8mm（縦27.8mmはA基板・B基板とパネライズグリッドを統一）【変更】|
|基板厚   |0.8mm（JLCPCB 2層 FR-4）                  |
|実装方式  |Top面のみへの片面実装。裏面は全面ベタGNDプレーン            |
|基板色   |Green または Black（最安・最短納期を優先）            |
|発注枚数  |パネル5枚発注により15枚取得（パネリング目論見書参照）           |
|外部接続端子|基板端エッジ寄りベタパッド × 7個（カスタレーションなし）         |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** パスコン C1・C2（1608 Basic Parts）をJLCPCBの自動実装に委託する。Basic Parts扱いのため追加手数料なし。
- **手はんだ（ユーザー後付け）：** U1（W25Q128JVSIM）・U2（MB85RS4MTYPF）はDigiKeyにて実調達済み（U1: 891円、U2: 2,888円）の現物を手はんだで実装する。

### システムブロックおよび電気的トポロジー

#### ストレージアーキテクチャ（CAR-01規格との整合）

C基板はCAR-01規格のストレージアーキテクチャに完全準拠する。

```
CAR-01 SPIバス（CH32V003 SPI1 @24MHz）
  SCK  = PC5
  MOSI = PC6
  MISO = PC7
  CS_FLASH = PA1  ──→ C基板 U1 /CS（W25Q128JVSIM）  【変更: 旧PD3】
  CS_FRAM   = PA2  ──→ C基板 U2 /CS（MB85RS4MTYPF）  【変更: 旧PD4】
```

**変更理由：** PD3/PD4はUIAPduino（CH32V003）のUSB D+/D−と競合するため使用不可。CS_FLASHをPA1へ、CS_FRAMをPA2へ移動した。

**U1（SPI Flash 16MB）の役割：** 読み出し専用マスターデータストレージ。全アプリのプログラム・画像・音声・シナリオ・恵梨沙フォント（0x008000固定）・SPI Flashカタログ（0x000000固定）を格納。

**U2（SPI FRAM 512KB）の役割：** 読み書き必須データの高速不揮発ストレージ。管理領域・セーブデータ・共通プログラムバックアップ・SPI Flash管理処理・SRAM相当作業領域（プレミアム版448KB）を格納。

### 回路設計のコアアーキテクチャ

#### 1. 受動部品の極限削減と1608サイズ選定

/WP（ライトプロテクト）および/HOLD（ホールド）ピンは、外付けプルアップ抵抗を一切配置せず、基板内銅箔パターンのみでVCC_3.3Vネットへ直接接続して無効化する。これにより実装部品数を最小の4点（U1・U2・C1・C2）に抑える。

#### 2. 堅牢なコモンバス配線とCS独立配線

電源（VCC_3.3V・GND）およびSPI共有信号線（SCK・MOSI・MISO）はU1・U2間で完全並列（共通バス）として引き回す。各チップセレクト（/CS）のみを独立させてエッジパッドへ引き出し、CAR-01規格のCS_FLASH・CS_FRAMに対応させる。

#### 3. 手はんだ最適化フットプリント

U1（SOIC-8 208mil）・U2（SOP-8 150mil）のフットプリントは、パッド長を標準より0.3mm延長した手はんだ最適化版を使用する。

-----

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# Claude Code Instruction: KiCad Schematic Netlist Generation
# for CAR-01 Storage Board C (Rev.1.1 Confirmed Version)

## 1. Context and Constraint
- Target Board     : C-Board, Storage Sub-Module (CAR-01 Standard)
- Dimensions       : 9.5mm x 27.8mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only (single-sided assembly).
                     Bottom side = solid GND copper pour, no routing.
- Manufacturing    : Hybrid assembly.
                     C1 and C2 (passive caps): JLCPCB PCBA (Basic Parts, no extra fee).
                     U1 and U2 (ICs): Hand-soldered by user (user stock from DigiKey).
- Footprint U1     : SOIC-8 208mil, hand-solder optimized
                     (extend pad length +0.3mm outward from standard).
- Footprint U2     : SOP-8 150mil, hand-solder optimized
                     (extend pad length +0.3mm outward from standard).
- Footprint C1/C2  : 1608 (0603 inch) metric standard footprint (PCBA compatible).
- Edge Pads        : 7x flat edge pads (no castellations) for external connection.

## 2. Component Assignments and RefDes

### Hand-Solder Items (ICs only)
- U1 : W25Q128JVSIM (SOIC-8, 208mil)
        128Mbit (16MB) SPI NOR Flash Memory
- U2 : MB85RS4MTYPF (SOP-8, 150mil)
        4Mbit (512KB) SPI FRAM

### JLCPCB PCBA Items (Basic Parts)
- C1 : 0.1uF / 50V / X7R Ceramic Capacitor (1608 metric) LCSC: C14663
- C2 : 0.1uF / 50V / X7R Ceramic Capacitor (1608 metric) LCSC: C14663

## 3. Netlist and Wiring Specifications

### Power Infrastructure
- Connect external VCC_3.3V net to U1 VCC (Pin 8) and U2 VCC (Pin 8).
- Connect unified GND net to U1 GND (Pin 4) and U2 GND (Pin 4).
- Place C1 (0.1uF) immediately adjacent to U1 Pin 8 (VCC) and Pin 4 (GND).
- Place C2 (0.1uF) immediately adjacent to U2 Pin 8 (VCC) and Pin 4 (GND).

### Shared SPI Bus Connections (Parallel)
- Connect external net SPI_SCK  to U1 CLK (Pin 6) and U2 SCK (Pin 6).
- Connect external net SPI_MOSI to U1 DIO (Pin 5) and U2 SI  (Pin 5).
- Connect external net SPI_MISO to U1 DO  (Pin 2) and U2 SO  (Pin 2).

### Dedicated Chip Select Lines  [CHANGED]
- Route U1 /CS (Pin 1) independently to external pad net CS_FLASH.
  (Connects to CH32V003 PA1 on main board)  [CHANGED: was PD3]
- Route U2 /CS (Pin 1) independently to external pad net CS_FRAM.
  (Connects to CH32V003 PA2 on main board)  [CHANGED: was PD4]

### Hardware Disabling of Unused Pins
- Connect U1 /WP   (Pin 3) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U1 /HOLD (Pin 7) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U2 /WP   (Pin 3) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U2 /HOLD (Pin 7) directly to VCC_3.3V net via copper trace on Top layer.

## 4. Edge Pad Assignment (7 pads, flat, no castellations)
- Pad 1 : VCC_3.3V
- Pad 2 : GND
- Pad 3 : SPI_SCK
- Pad 4 : SPI_MOSI
- Pad 5 : SPI_MISO
- Pad 6 : CS_FLASH  (connects to CH32V003 PA1)  [CHANGED: was PD3]
- Pad 7 : CS_FRAM   (connects to CH32V003 PA2)  [CHANGED: was PD4]

## 5. PCB Layout Rules
- ALL components on Top layer only.
- Bottom layer: solid GND copper pour, no signal routing.
- U1 (SOIC-8): place upper half of board, centered horizontally.
- U2 (SOP-8): place lower half of board, centered horizontally.
- C1 placed immediately adjacent to U1 VCC (Pin8) and GND (Pin4).
- C2 placed immediately adjacent to U2 VCC (Pin8) and GND (Pin4).
- /WP and /HOLD: connect to VCC_3.3V via short copper trace. No external exposure.
- Edge pads: distribute along one 27.8mm edge.
- Trace width: minimum 0.2mm for signal, 0.4mm for power.
- Clearance: minimum 0.15mm.
- Silkscreen: mark Pin 1 dot on U1 and U2. Label edge pads.

Generate complete KiCad schematic netlist, PCB layout file,
and Python build script based on this finalized architecture.
```

-----

## 3. 部品一覧表（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計2点

|RefDes|部品名・仕様                        |パッケージ      |LCSC番号|区分       |数量|役割                     |
|------|------------------------------|-----------|------|---------|--|-----------------------|
|C1    |積層セラミックコンデンサ 0.1µF / 50V / X7R|1608 (0603)|C14663|**Basic**|1 |U1 VCCバイパスコンデンサ（電源ピン直近）|
|C2    |積層セラミックコンデンサ 0.1µF / 50V / X7R|1608 (0603)|C14663|**Basic**|1 |U2 VCCバイパスコンデンサ（電源ピン直近）|

### 🔴 ユーザー手はんだ実装　計2点

|RefDes|部品名・仕様                                        |パッケージ          |参考LCSC番号|実装方法                        |数量|
|------|----------------------------------------------|---------------|--------|----------------------------|--|
|U1    |SPI Flash Memory 128Mbit (16MB) / W25Q128JVSIM|SOIC-8 (208mil)|C2613930|**手はんだ** (DigiKey調達済 891円)  |1 |
|U2    |SPI FRAM 4Mbit (512KB) / MB85RS4MTYPF         |SOP-8 (150mil) |C1020305|**手はんだ** (DigiKey調達済 2,888円)|1 |

-----

## 4. 設計根拠メモ

### CS_FLASH・CS_FRAMピン変更根拠【変更】

旧仕様（Rev.1.0）ではCS_FLASH=PD3・CS_FRAM=PD4としていた。UIAPduino（CH32V003）においてPD3=USB D+・PD4=USB D−であり、USB D+/D−と競合するため使用不可。2026年6月のピンアサイン見直しにより、CS_FLASHをPA1・CS_FRAMをPA2へ変更した。

### /WP・/HOLDピンの処理根拠

W25Q128JVSIM・MB85RS4MTYPFともに/WP・/HOLDピンは未使用。VCC_3.3Vへ直結して無効化。プルアップ抵抗不要な理由は常時固定電位で十分なため。

-----

## 5. CAR-01規格との整合確認【変更】

|信号      |C基板パッド|CH32V003ピン|確認|
|--------|------|----------|--|
|VCC_3.3V|Pad 1 |3.3V系     |✅ |
|GND     |Pad 2 |GND       |✅ |
|SPI_SCK |Pad 3 |PC5       |✅ |
|SPI_MOSI|Pad 4 |PC6       |✅ |
|SPI_MISO|Pad 5 |PC7       |✅ |
|CS_FLASH|Pad 6 |**PA1**   |✅ 【変更: 旧PD3】|
|CS_FRAM |Pad 7 |**PA2**   |✅ 【変更: 旧PD4】|

-----

## 変更履歴

|日付        |バージョン  |内容                                                    |
|----------|-------|------------------------------------------------------|
|2026-05-27|Rev.1.0|初版作成・確定版                                             |
|2026-06-10|Rev.1.1|CS_FLASHのCH32V003接続先をPD3→PA1に変更（USB D+競合回避）【変更】      |
|2026-06-10|Rev.1.1|CS_FRAMのCH32V003接続先をPD4→PA2に変更（USB D−競合回避）【変更】       |
|2026-06-10|Rev.1.1|基板サイズを10×28mm→9.5×27.8mmに修正（A/B基板との統一）【変更】           |
|2026-06-10|Rev.1.1|§2 Netlist・§4 エッジパッド・§5 整合確認表をPA1/PA2に全面更新【変更】       |

-----

*文書バージョン：Rev.1.1確定版 / 本ファイルをそのままClaude Codeへ投入してください*
