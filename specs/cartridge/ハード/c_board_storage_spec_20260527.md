# CAR-01規格 C基板（ストレージモジュール）設計仕様書

## 完全確定版 — Claude Code投入用マスタードキュメント

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（C基板）は、ラテカピュータ（CAR-01規格）のストレージ機能を担う極小ドーターボードである。メインアセット格納用の大容量SPI Flash（128Mbit / 16MB）と、超高速・高寿命なSRAM代替作業領域およびセーブ領域用のSPI FRAM（4Mbit / 512KB）を1枚の基板に統合する。

10mm × 28mmのスティック形状は、A基板（電源管理：10×28mm）・B基板（FM音源：18×28mm）と縦幅28mmを完全に統一しており、3基板をV-CUTのみで切り離せる84×84mmの格子状パネルを構成する。

### 物理仕様・製造要件

| 項目 | 仕様 |
|------|------|
| 基板サイズ | 10mm × 28mm（縦28mmはA基板・B基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | Top面のみへの片面実装。裏面は全面ベタGNDプレーン |
| 基板色 | Green または Black（最安・最短納期を優先） |
| 発注枚数 | パネル5枚発注により15枚取得（パネリング目論見書参照） |
| 外部接続端子 | 基板端エッジ寄りベタパッド × 7個（カスタレーションなし） |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** パスコン C1・C2（1608 Basic Parts）をJLCPCBの自動実装に委託する。Basic Parts扱いのため追加手数料なし。微細なチップコンデンサの手はんだ作業を完全に排除できる。
- **手はんだ（ユーザー後付け）：** U1（W25Q128JVSIM）・U2（MB85RS4MTYPF）はDigiKeyにて実調達済み（U1: 891円、U2: 2,888円）の現物を手はんだで実装する。
- **理由：** U1・U2はExtended Parts扱いでPCBA委託すると段取り費・治具費が高額になるため手はんだが経済的に最適。一方C1・C2はBasic Partsのため追加費用ゼロでPCBA委託できる。ICのみ手はんだ・受動部品はPCBAという最適な分担とする。

### システムブロックおよび電気的トポロジー

#### ストレージアーキテクチャ（CAR-01規格との整合）

C基板はCAR-01規格のストレージアーキテクチャに完全準拠する。

```
CAR-01 SPIバス（CH32V003 SPI1 @24MHz）
  SCK  = PC5
  MOSI = PC6
  MISO = PC7
  CS_FLASH = PD3  ──→ C基板 U1 /CS（W25Q128JVSIM）
  CS_FRAM   = PD4  ──→ C基板 U2 /CS（MB85RS4MTYPF）
```

**U1（SPI Flash 16MB）の役割：** 読み出し専用マスターデータストレージ。全アプリのプログラム・画像・音声・シナリオ・恵梨沙フォント（0x008000固定）・SPI Flashカタログ（0x000000固定）を格納。

**U2（SPI FRAM 512KB）の役割：** 読み書き必須データの高速不揮発ストレージ。管理領域・セーブデータ・共通プログラムバックアップ・SPI Flash管理処理・SRAM相当作業領域（プレミアム版448KB）を格納。

### 回路設計のコアアーキテクチャ

#### 1. 受動部品の極限削減と1608サイズ選定

/WP（ライトプロテクト）および/HOLD（ホールド）ピンは、外付けプルアップ抵抗を一切配置せず、基板内銅箔パターンのみでVCC_3.3Vネットへ直接接続して無効化する。これにより実装部品数を最小の4点（U1・U2・C1・C2）に抑える。

パスコン（C1・C2）は1608（0603インチ）サイズを採用する。JLCPCBのPCBA Basic Partsとして追加手数料なしで自動実装できるサイズであり、ICの手はんだ作業に集中できる環境を確保するための選定。

#### 2. 堅牢なコモンバス配線とCS独立配線

電源（VCC_3.3V・GND）およびSPI共有信号線（SCK・MOSI・MISO）はU1・U2間で完全並列（共通バス）として引き回す。各チップセレクト（/CS）のみを独立させてエッジパッドへ引き出し、CAR-01規格のCS_FLASH・CS_FRAMに対応させる。

#### 3. 手はんだ最適化フットプリント

U1（SOIC-8 208mil）・U2（SOP-8 150mil）のフットプリントは、パッド長を標準より0.3mm延長した手はんだ最適化版を使用する。フラックス塗布エリアの確保とはんだブリッジ防止を両立させる。

---

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# Claude Code Instruction: KiCad Schematic Netlist Generation
# for CAR-01 Storage Board C (Final Confirmed Version)

## 1. Context and Constraint
- Target Board     : C-Board, Storage Sub-Module (CAR-01 Standard)
- Dimensions       : 10mm x 28mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only (single-sided assembly).
                     Bottom side = solid GND copper pour, no routing.
- Panel            : Panelized with A-Board (10x28mm) and B-Board (18x28mm).
                     All boards share 28mm height. V-CUT singulation only.
                     Panel total: 84mm x 84mm.
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
        Winbond / LCSC: C2613930 (ref) / Hand-solder (user stock from DigiKey)
- U2 : MB85RS4MTYPF (SOP-8, 150mil)
        4Mbit (512KB) SPI FRAM
        Fujitsu/RAMXEED / LCSC: C1020305 (ref) / Hand-solder (user stock from DigiKey)

### JLCPCB PCBA Items (Basic Parts)
- C1 : 0.1uF / 50V / X7R Ceramic Capacitor (1608 metric)
        Bypass Capacitor for U1 VCC. JLCPCB PCBA Target.
        LCSC: C14663 (CC0603KRX7R9BB104 YAGEO) / Basic Part / No extra fee.
- C2 : 0.1uF / 50V / X7R Ceramic Capacitor (1608 metric)
        Bypass Capacitor for U2 VCC. JLCPCB PCBA Target.
        LCSC: C14663 (CC0603KRX7R9BB104 YAGEO) / Basic Part / No extra fee.

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

### Dedicated Chip Select Lines
- Route U1 /CS (Pin 1) independently to external pad net CS_FLASH.
  (Connects to CH32V003 PD3 on main board)
- Route U2 /CS (Pin 1) independently to external pad net CS_FRAM.
  (Connects to CH32V003 PD4 on main board)

### Hardware Disabling of Unused Pins
- Connect U1 /WP   (Pin 3) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U1 /HOLD (Pin 7) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U2 /WP   (Pin 3) directly to VCC_3.3V net via copper trace on Top layer.
- Connect U2 /HOLD (Pin 7) directly to VCC_3.3V net via copper trace on Top layer.
- Do NOT route above pins to external pads or pull-up resistors.

## 4. Edge Pad Assignment (7 pads, flat, no castellations)
- Pad 1 : VCC_3.3V
- Pad 2 : GND
- Pad 3 : SPI_SCK
- Pad 4 : SPI_MOSI
- Pad 5 : SPI_MISO
- Pad 6 : CS_FLASH  (connects to CS_FLASH on main board / CH32V003 PD3)
- Pad 7 : CS_FRAM   (connects to CS_FRAM on main board / CH32V003 PD4)

## 5. PCB Layout Rules
- ALL components on Top layer only.
- Bottom layer: solid GND copper pour, no signal routing.
- U1 (SOIC-8): place upper half of board, centered horizontally.
- U2 (SOP-8): place lower half of board, centered horizontally.
- C1 placed immediately adjacent to U1 VCC (Pin8) and GND (Pin4).
- C2 placed immediately adjacent to U2 VCC (Pin8) and GND (Pin4).
- /WP and /HOLD of both U1 and U2: connect to VCC_3.3V via short
  copper trace on Top layer. No external exposure.
- Edge pads: distribute along one 28mm edge (consistent with
  A-Board and B-Board interface edge orientation in panel).
- Trace width: minimum 0.2mm for signal, 0.4mm for power.
- Clearance: minimum 0.15mm.
- Silkscreen: mark Pin 1 dot on U1 and U2. Label edge pads.
- U1 Pin 1 orientation: confirm with W25Q128JVSIM datasheet
  (/CS pin at top-left when viewed from component side).

Generate complete KiCad schematic netlist (.kicad_sch),
PCB layout file (.kicad_pcb), and Python build script
based on this finalized architecture.
Use KiCad MCP server for direct file generation.
```

---

## 3. 部品一覧表（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計2点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 | 役割 |
|--------|------------|----------|---------|------|------|------|
| C1 | 積層セラミックコンデンサ 0.1µF / 50V / X7R | 1608 (0603) | C14663 | **Basic** | 1 | U1 VCCバイパスコンデンサ（電源ピン直近） |
| C2 | 積層セラミックコンデンサ 0.1µF / 50V / X7R | 1608 (0603) | C14663 | **Basic** | 1 | U2 VCCバイパスコンデンサ（電源ピン直近） |

### 🔴 ユーザー手はんだ実装　計2点

| RefDes | 部品名・仕様 | パッケージ | 参考LCSC番号 | 実装方法 | 数量 | 役割 |
|--------|------------|----------|------------|---------|------|------|
| U1 | SPI Flash Memory 128Mbit (16MB) / W25Q128JVSIM | SOIC-8 (208mil) | C2613930 | **手はんだ** (DigiKey調達済 891円) | 1 | メインアセット用大容量ストレージ |
| U2 | SPI FRAM 4Mbit (512KB) / MB85RS4MTYPF | SOP-8 (150mil) | C1020305 | **手はんだ** (DigiKey調達済 2,888円) | 1 | SRAM代替作業領域・セーブ領域（プレミアム版512KB） |

### JLCPCB発注情報

| 項目 | 仕様 |
|------|------|
| 発注形態 | パネル内C基板として発注（パネリング目論見書参照） |
| パネル内取数 | 3個/パネル |
| 発注パネル数 | 5パネル |
| 取得枚数 | 15枚 |

---

## 4. 設計根拠メモ（Claude Code補足情報）

### 実装方針の根拠（ICは手はんだ・パスコンはPCBA）

U1・U2はDigiKeyにて実調達済み（合計3,779円＋送料）。JLCPCBでPCBA委託した場合のExtended部品段取り費（各+$3）・治具費用と比較して、手持ち在庫を活用する手はんだが経済的に最適。一方、C1・C2（パスコン）はLCSC C14663（Basic Parts）のためPCBA追加手数料ゼロで自動実装できる。0603サイズの微細なチップコンデンサを手はんだする手間を省き、ICの手はんだ作業に集中できる最適な分担。

### /WP・/HOLDピンの処理根拠

W25Q128JVSIM・MB85RS4MTYPFともに/WP（ライトプロテクト）・/HOLD（ホールド）ピンは未使用。電気的に確実に無効化するためVCC_3.3Vへ直結する。プルアップ抵抗不要な理由は、これらピンが常時固定電位で十分（動的制御不要）なため。基板面積の節約にも寄与する。

### SPI FRAM（MB85RS4MTYPF）の仕様確認事項

- パッケージ：SOP-8 150mil（SOIC-8 208milではない）
- 最大クロック：50MHz（CH32V003の24MHzで十分）
- SPI MODE：0と3に対応
- 書き換え回数：100兆回（UIAPduino交換方針と整合）
- Extended部品のためJLCPCB在庫は変動する可能性あり（手持ち在庫で問題なし）

### エッジパッド設計根拠

7本のエッジパッドはA基板・B基板と同じ面（28mm辺の一端）に揃えて配置する。パネル内でのV-CUT方向との整合性を保ち、切り離し後のメイン基板への実装向きを統一するため。

### CAR-01規格との整合確認

| 信号 | C基板パッド | CH32V003ピン | 確認 |
|------|------------|------------|------|
| VCC_3.3V | Pad 1 | 3.3V系 | ✅ |
| GND | Pad 2 | GND | ✅ |
| SPI_SCK | Pad 3 | PC5 | ✅ |
| SPI_MOSI | Pad 4 | PC6 | ✅ |
| SPI_MISO | Pad 5 | PC7 | ✅ |
| CS_FLASH | Pad 6 | PD3 | ✅ |
| CS_FRAM | Pad 7 | PD4 | ✅ |

---

*文書バージョン：確定版 2026-05-27 / 本ファイルをそのままClaude Codeへ投入してください*
