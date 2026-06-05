# CAR-01規格 B基板（FM音源オーディオモジュール）設計仕様書
## 完全確定版 — Claude Code投入用マスタードキュメント

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（B基板）は、ラテカピュータ（CAR-01規格）に準拠した汎用FM音源モジュール基板である。ヤマハ製FM音源LSI「YMF825-EZ」を核とし、フリスクケース内の極小スペース（18mm × 28mm）に収まるよう高密度に最適化する。

### 物理仕様・製造要件

| 項目 | 仕様 |
|---|---|
| 基板サイズ | 18mm × 28mm（縦28mmはA基板・C基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | 表面実装（SMD）部品を中心とした片面高密度実装 |

#### 製造委託範囲

- **手はんだ（ユーザー後付け）：** 音源チップ（YMF825-EZ）、3.5mmステレオミニジャック（MJ-352W-C想定）、12.288MHz水晶の3点
- **JLCPCB PCBA（自動実装委託）：** 周辺の受動部品（1608サイズ抵抗・コンデンサ）およびアンプIC（PAM8302A）のすべて

### 回路・音響設計のコアアーキテクチャ

#### 1. ステレオ・モノラル自動可変ミックス回路

YMF-825から出力されるLch/Rchのアナログ音声信号は、3.5mmジャックへはダイレクトにステレオのまま直行する。同時に、10kΩのミックス抵抗（R3, R4）を介して合流し、モノラル信号としてアンプの入力側へと導かれる。

#### 2. 完全差動入力を活かしたジャック物理スイッチ連動消音機構

モノラルアンプ（U1: PAM8302A）の差動入力端子のうち、IN+ にミックスされたモノラル音声信号をカップリングコンデンサ（C2）経由で入力する。IN- 側は、J1（MJ-352W-C想定）の左チャンネル用ノーマリークローズ（NC）スイッチ端子をC5経由でアンプ IN- へ結線し、J1のLeft Tip端子はシステムGNDへ直接固定する。

| 状態 | ジャック内部 | IN- の状態 | スピーカー |
|---|---|---|---|
| イヤホン**未挿入** | NCスイッチ**閉**（Tip端子と短絡） | C5 → Tip端子（GND）へ接地 | **鳴る** |
| イヤホン**挿入** | NCスイッチ**開** | 完全フローティング | **消音** |

> **動作原理：** プラグ挿入時にNCスイッチが物理的に跳ね上がり、IN- へのGNDパスが遮断される。アンプは差動基準を失い、スピーカー出力を自動カットする。GPIO制御不要のピュアハードウェア消音機構。

#### 3. アンプの起動制御

U1のSD（シャットダウン）ピンは、10kΩのプルアップ抵抗（R5）を介してAMP_5.0Vに直結し常時アクティブとする。モジュール全体のON/OFFは、前段の電源管理モジュール（A基板：ATtiny202 + IRLML6402 MOSFET）による5V給電自体の遮断によって制御する。

#### 4. YMF825-EZ電源安定化

YMF825-EZの電源ピン（VCC_3.3V）直近に0.1µFのバイパスコンデンサ（C6, C7）をJLCPCB PCBAで自動実装する。これによりユーザーの手はんだ作業をLSI本体（SSOP-24）と水晶の2点のみに集約し、製造安定性を確保する。

#### 5. 高周波キャリアノイズカット（RC低域通過フィルタ）

音源チップ出力に含まれる高周波ノイズを減衰させるため、アンプ前段にカットオフ周波数約160kHzの1kΩ（R1, R2）＋1000pF（C3, C4）によるRCフィルタを左右独立して配置する。

---

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# Claude Code Instruction: KiCad Schematic Netlist Generation
# for CAR-01 Audio Board (Final Confirmed Version)

## 1. Context and Constraint
- Target Board: YMF-825board Compatible Audio Module (CAR-01 Standard)
- Dimensions: 18mm x 28mm, PCB Thickness 0.8mm
- Manufacturing: All resistors, capacitors, and Amplifier IC (U1) MUST
  use JLCPCB PCBA (SMD).
- Footprint Size for passives: 1608 (0603 inch) metric package size.
- Footprint for U1: Package_SO:MSOP-8_3x3mm_P0.65mm
  (Strictly verified for PAM8302AAYCR / LCSC: C11117)

## 2. Component Assignments and RefDes
- U1 : PAM8302AAYCR (MSOP-8)
        Footprint: Package_SO:MSOP-8_3x3mm_P0.65mm
        LCSC: C11117                        -> JLCPCB PCBA Target
- U2 : YMF825-EZ (SSOP-24)                 -> Hand-solder footprint only
- X1 : 12.288MHz Crystal (SMD 3225-4P)     -> Hand-solder footprint only
- J1 : 3.5mm Stereo Mini Jack
        MJ-352W-C style, 5-Pin Thru-hole
        Switch type: Pattern-A NC
        (Tip/Ring pins short to Switch pins when plug is absent;
         opens upon plug insertion)         -> Hand-solder footprint only
- C1 : 10uF / 10V / X5R (1608)             -> JLCPCB PCBA Target
- C2 : 1.0uF / 16V / X7R (1608)            -> JLCPCB PCBA Target
- C5 : 1.0uF / 16V / X7R (1608)            -> JLCPCB PCBA Target
- C3 : 1000pF / 50V / NPO (1608)           -> JLCPCB PCBA Target
- C4 : 1000pF / 50V / NPO (1608)           -> JLCPCB PCBA Target
- C6 : 100nF / 50V / X7R (1608)            -> JLCPCB PCBA Target
- C7 : 100nF / 50V / X7R (1608)            -> JLCPCB PCBA Target
- R1 : 1kOhm / +/-1% (1608)               -> JLCPCB PCBA Target
- R2 : 1kOhm / +/-1% (1608)               -> JLCPCB PCBA Target
- R3 : 10kOhm / +/-1% (1608)              -> JLCPCB PCBA Target
- R4 : 10kOhm / +/-1% (1608)              -> JLCPCB PCBA Target
- R5 : 10kOhm / +/-1% (1608)              -> JLCPCB PCBA Target

## 3. Netlist and Wiring Specifications

### Power and Ground Lines
- Connect AMP_5.0V to U1 VDD (Pin 6).
  Place C1 (10uF) as close as possible between U1 VDD and U1 GND (Pin 2).
- Connect VCC_3.3V to U2 VDD pins (per YMF825 datasheet).
  Place C6 and C7 (100nF each) directly between VCC_3.3V and GND,
  as close as possible to U2 power pins.
- Connect all GND pins (including J1 Ground Pin and U1 Pin 2)
  to unified net GND.

### YMF-825 (U2) Analog Output and RC Filter Section
- U2 LOUT (Pin 19) -> Net AUDIO_L_RAW -> R1 (1kOhm).
- R1 opposite side -> Net AUDIO_L_FILTERED.
  Connect C3 (1000pF) from AUDIO_L_FILTERED to GND.
- U2 ROUT (Pin 18) -> Net AUDIO_R_RAW -> R2 (1kOhm).
- R2 opposite side -> Net AUDIO_R_FILTERED.
  Connect C4 (1000pF) from AUDIO_R_FILTERED to GND.

### Stereo Jack (J1) and Mixing Routing
- Connect AUDIO_L_FILTERED to J1 Left Tip Pin.
- Connect AUDIO_R_FILTERED to J1 Right Ring Pin.
- Connect J1 Left Tip Pin firmly to system GND net.
  (Tip is grounded to serve as NC switch reference for mute logic.)
- Connect AUDIO_L_FILTERED to R3 (10kOhm).
- Connect AUDIO_R_FILTERED to R4 (10kOhm).
- Combine opposite sides of R3 and R4 -> Net AUDIO_MONO_MIX.

### Amplifier SD Pin Treatment
- Connect U1 SD (Pin 7) to R5 (10kOhm).
- Connect opposite side of R5 to AMP_5.0V.
  (Amplifier kept permanently enabled; power-cycle control is handled
   upstream by A-board MOSFET.)

### Hardware Insertion Mute and Amplifier Input
- Connect AUDIO_MONO_MIX to C2 (1.0uF).
- Connect C2 opposite side to U1 IN+ (Pin 4).
- Connect J1 Left Switch Pin (NC pin that shorts to Left Tip Pin
  when plug is absent) to C5 (1.0uF).
- Connect C5 opposite side to U1 IN- (Pin 3).
- [Mute Logic]
    Jack empty  : NC switch CLOSED -> Left Switch Pin shorted to Tip Pin
                  (GND) -> IN- tied to GND via C5.
                  Amplifier operates normally. Speaker active.
    Jack inserted: NC switch OPENS -> IN- floats completely.
                  Amplifier loses differential reference.
                  Speaker output muted automatically.

### Speaker Output
- U1 VO+ (Pin 5) -> External pad SPK+
- U1 VO- (Pin 8) -> External pad SPK-

### Digital Control Interface (Edge Pads to U2)
- Expose from U2: SPI_SCK, SPI_MOSI, SPI_MISO, CS_YMF, RST_YMF

Generate complete KiCad schematic netlist and Python build script
based on this finalized architecture.
```

---

## 3. 部品一覧表（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計13点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 | 役割 |
|---|---|---|---|---|---|---|
| U1 | Dクラスアンプ（PAM8302AAYCR） | MSOP-8 | C11117 | Basic | 1 | スピーカー駆動用アンプ。PCBAコスト最安 |
| C1 | 電源平滑コンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | U1 VCC直近パスコン |
| C2 | 入力カップリングコンデンサ（1.0µF/16V/X7R） | 1608 | C15849 | Basic | 1 | IN+直前・直流カット用 |
| C5 | 入力カップリングコンデンサ（1.0µF/16V/X7R） | 1608 | C15849 | Basic | 1 | IN-側・ジャックNCスイッチ（パターンA）経由GND接続用 |
| C3 | RCフィルタコンデンサ（1000pF/50V/NPO） | 1608 | C15984 | Basic | 1 | Lch高周波ノイズカット |
| C4 | RCフィルタコンデンサ（1000pF/50V/NPO） | 1608 | C15984 | Basic | 1 | Rch高周波ノイズカット |
| C6 | YMF825用パスコン（100nF/50V/X7R） | 1608 | CC0603KRX7R9BB104 | Basic | 1 | U2 VCC_3.3V直近デカップリング |
| C7 | YMF825用パスコン（100nF/50V/X7R） | 1608 | CC0603KRX7R9BB104 | Basic | 1 | U2 アナログ3.3V電源ピン直近デカップリング |
| R1 | RCフィルタ抵抗（1kΩ/1/10W/±1%） | 1608 | C21190 | Basic | 1 | Lch RCフィルタ抵抗 |
| R2 | RCフィルタ抵抗（1kΩ/1/10W/±1%） | 1608 | C21190 | Basic | 1 | Rch RCフィルタ抵抗 |
| R3 | ミックス抵抗（10kΩ/1/10W/±1%） | 1608 | C25804 | Basic | 1 | Lch→モノラル合流・逆流防止 |
| R4 | ミックス抵抗（10kΩ/1/10W/±1%） | 1608 | C25804 | Basic | 1 | Rch→モノラル合流・逆流防止 |
| R5 | SDピン用プルアップ抵抗（10kΩ/1/10W/±1%） | 1608 | C25804 | Basic | 1 | アンプ常時アクティブ化 |

### 🔴 ユーザー手元での後付け（手はんだ）対象パーツ　計3点

| RefDes | 部品名・仕様 | パッケージ | 数量 | 備考 |
|---|---|---|---|---|
| U2 | FM音源LSI（YMF825-EZ） | SSOP-24 | 1 | ユーザー保有在庫を実装 |
| X1 | 水晶振動子（12.288MHz） | SMD 3225-4P | 1 | YMF-825専用マスタークロック |
| J1 | 3.5mmステレオミニジャック（MJ-352W-C想定） | スルーホール 5ピン | 1 | パターンAスイッチ仕様・自動消音回路の要 |

---

## 4. 設計根拠メモ（Claude Code補足情報）

### U1フットプリントについて（重要）

PAM8302AAYCR（LCSC: C11117）は **MSOP-8パッケージ**（ピッチ0.65mm）である。
KiCadでフットプリントを選択する際は必ず以下を指定すること。

```
Package_SO:MSOP-8_3x3mm_P0.65mm
```

> ⚠️ SOIC-8（ピッチ1.27mm）と混同しないこと。ピッチが異なるため実装不可となる。

### J1スイッチ構造について（パターンA確定）

MJ-352W-C想定品は **パターンA構造** を持つ。プラグ挿入前はTip端子とLeft NCスイッチ端子が内部で短絡しており、プラグ挿入によって開放される。本設計ではTip端子をGNDに固定しているため：

- 未挿入時：NCスイッチ閉 → Tip（GND）経由でIN-がC5を通じてGNDに接続 → アンプ正常動作
- 挿入時：NCスイッチ開 → IN-が完全フローティング → アンプ消音

### SDピン常時プルアップについて

PAM8302AのSDピン（Pin 7、Active Low）をAMP_5.0VへR5（10kΩ）でプルアップしてアンプを常時Enable状態とする。電源ON/OFF制御はA基板のIRLML6402 MOSFETによるAMP_5.0V供給の物理遮断で行うため、追加のGPIO消費は発生しない。

### YMF825-EZピン番号（回路指示書記載値）

| 信号 | ピン番号 |
|---|---|
| LOUT（Left Analog Out） | Pin 19 |
| ROUT（Right Analog Out） | Pin 18 |

> ⚠️ SSOP-24パッケージのYMF825-EZデータシートにて実装前に必ず照合すること。

---

*文書バージョン：完全確定版 / 本ファイルをそのままClaude Codeへ投入してください*
