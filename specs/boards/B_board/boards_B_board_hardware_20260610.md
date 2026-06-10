# CAR-01規格 B基板（FM音源モジュール）設計仕様書

**Rev. 2.8 — 2026年6月9日**

---

## 1. 本書の位置づけ

本書はラテカピュータ（CAR-01規格）のB基板（FM音源モジュール）の設計仕様を定める。A基板・C基板・D基板・カートリッジ規格マスターと整合性を保ちながら、B基板固有の設計要件を記述する。

**参照ドキュメント：**

| ドキュメント | 参照事項 |
|---|---|
| cartridge_spec_master_20260522_1.md | CAR-01全体規格・電源方針・SPIバス設計 |
| A_board_spec_20260601.md | MT3608 5V出力・電源シーケンス |
| C基板設計仕様 | SPIバス共有・CS信号割り当て |
| d_board_wnn_prospectus_20260601.md | I2Cバストポロジー |

---

## 2. 概要・目的

### 2.1 役割

B基板はYMF825-EZ（4オペレータ・16チャンネルFM音源LSI）を中心とするFM音源モジュール基板である。CAR-01の左半分最下層に位置し、UIAPduino（CH32V003）からのSPI制御を受けてFM音声を出力する。

**越えるべき目標：** uda.la/fm/ YMF825Board（ウダデンシ製）の音質・回路品質同等以上。

### 2.2 出力構成

YMF825は16bitモノラルDACを1つ持つ。Pin14（SPOUT1/LINEOUT1）とPin16（SPOUT2/LINEOUT2）は同一モノラル信号の差動（BTL）ペアである。**ステレオ出力は存在しない。内蔵BTLスピーカーアンプは使用しない。**

SPOUT1/2をLINEOUTとして使用し、MCP6001で差動→SE変換後にPAM8302Aへ送る構成とする。AUDIO_IN（アナログ側エッジパッド）からの外部音声入力はR_PWM経由でAUDIO_MONO_MIXへ混合する。UIAPduino PC3（PWM出力）を接続する場合はJP1をオープンにしてRC-LPF経由とし、mp3デコーダ等のアナログ音声を直接接続する場合はJP1をショート（はんだブリッジまたは0Ω抵抗実装）してRC-LPFをバイパスする。

| 出力先 | 構成 | 備考 |
|---|---|---|
| スピーカー | LINEOUT→MCP6001（差動→SE変換）→抵抗ミックス（YMF825＋AUDIO_IN）→PAM8302A→BTL出力 | 内蔵BTLアンプ不使用 |
| ヘッドフォンジャック | AUDIO_MONO_MIX（YMF825＋AUDIO_IN混合後）→R10/R11→L/R両チャンネル | モノラル信号・両耳から同音 |

---

## 3. 物理仕様

### 3.1 基板寸法

| 項目 | 仕様 |
|---|---|
| 基板サイズ | 19mm × 27.8mm |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| パネライズグリッド | 19mm幅（A基板9.5mm＋C基板9.5mm＝19mmと統一） |
| パネル構成 | 84×84mm V-CUTパネル（A基板・C基板・B基板・D基板共通） |

### 3.2 スタック配置

フリスクカートリッジを横長に置き、蓋がスライドして開く方向を「右」と定義する。

```
【左半分】                        【右半分】
3層：TFTディスプレイ              3層：D基板
2層：バッテリー                   2層：A基板 ＋ C基板（横並び）
1層：B基板（本書対象）            1層：UIAPduino
```

左右の積層は高さ方向で完全に独立しており、各層の厚みは互いに連動しない。

**左半分の高さ設計：**

```
ジャック部分              パーツ部分
┌──────────┐         ┌──────────┐
│TFTディスプレイ│         │TFTディスプレイ│
├──────────┤ ← 同高 →├──────────┤
│              │         │  バッテリー  │
│  （ジャック） │         ├──────────┤
│              │         │  YMF825等  │
├──────────┤         ├──────────┤
│    B基板     │         │    B基板    │
└──────────┘         └──────────┘
```

- ジャック部分（MJ-495A）は高さが大きい
- パーツ部分（YMF825等）はジャックより低い
- バッテリーの厚みは「ジャック高さ − パーツ高さ」の差分相当を選定し、パーツ部分に載せることで左半分全体のトップ面をジャック高さと概ね揃える
- TFTディスプレイはその上に水平に乗る
- MJ-495Aの挿入口はケース側面穴から外部へ突出する配置とし、基板上の高さ制約を回避する

### 3.3 GNDプレーン設計方針（音質対策）

YMF825Boardを超える音質を目指すため、以下を全点実施する。

| 対策 | 内容 |
|---|---|
| 裏面全体GNDベタ | 裏面（Bottom）は信号配線なし・全面GND銅箔 |
| 表面アナログGNDゾーン | SPOUT1/2周辺にアナログGND独立ゾーン |
| ガードビア列 | SPOUT1/2トレース両側に0.3mmビア・1mm間隔で配置 |
| デジタル/アナログ分離 | YMF825直下1点のみでデジ/アナGND接続（スター接地） |
| 信号ゾーン分離 | SPI信号線とSPOUT1/2を基板上で対角ゾーンに配置 |

---

## 4. 回路設計

### 4.1 電源構成

A基板から供給される電源を使用する。B基板は独自の電源生成を行わない。

| 電源ライン | 供給元 | 電圧 | 主な用途 |
|---|---|---|---|
| VCC_3.3V | A基板 Buck DCDC（RT8008-33GB） | 3.3V | YMF825 VDD・IOVDD・VREF、MCP6001 VDD |
| AMP_5.0V | A基板 MT3608 | 5.0V | PAM8302A VDD、YMF825 SPVDD |

**SPVDD電源方針：**
YMF825のSPVDDはスピーカーアンプ段のみに供給され、DACコア段はVDD（3.3V）で動作する。LINEOUT使用時にSPVDDリプルがDAC出力に直接乗る経路は限定的であり、後段のカップリングコンデンサ（C2/C5）で低周波リプルは遮断される。

以上の理由からSPVDD専用LDOは不採用とし、代わりにSPVDD直近に**47µF/1210**の大容量コンデンサを配置して高周波リプルを吸収する。

### 4.2 信号フロー

```
YMF825-EZ
  Pin14 (SPOUT1/LINEOUT1) --R1(1kΩ)--> AUDIO_DIFF_P
                                C3(1000pF) to GND
  Pin16 (SPOUT2/LINEOUT2) --R2(1kΩ)--> AUDIO_DIFF_N
                                C4(1000pF) to GND
                                        │
                               U3 MCP6001（差動アンプ・ユニティゲイン）
                               R6,R7,R8,R9（各10kΩ）
                                        │
                                  AUDIO_MONO
                                        │
                               R3(10kΩ)・R4(10kΩ)
                                        │
AUDIO_IN（アナログ側エッジパッド）      │
  │                                     │
  ├─ JP1オープン（PWM入力時）            │
  │   → R_LPF(1kΩ) → C_LPF(15nF→GND) │
  │                  → R_PWM(10kΩ) ────┤
  │                                     ├─ AUDIO_MONO_MIX
  └─ JP1ショート（アナログ直入力時）     │
      → R_PWM(10kΩ) ───────────────────┘
                                        │
                          ┌─────────────┴─────────────┐
                          │                           │
                     C2(1.0µF)                  R10(100Ω)・R11(100Ω)
                          │                           │
                   U1 PAM8302A IN+           J1 Tip(L) / Ring(R)
                          │                    [ヘッドフォンL/R]
  J1 NC Switch --> C5(1.0µF) --> U1 IN-
  J1 Tip = GND
                          │
                  U1 VO+/VO- --> SPK+/SPK-
```

**JP1切り替え方法：**

| JP1状態 | 実装方法 | 用途 |
|---|---|---|
| オープン（デフォルト） | JP1未実装 | UIAPduino PWM出力（ADPCM声等）を接続 |
| ショート | JP1にはんだブリッジまたは0Ω抵抗を実装 | mp3デコーダ等のアナログ音声を直接接続 |

### 4.3 YMF825-EZ 接続仕様

| ピン | 信号名 | 接続先 | 備考 |
|---|---|---|---|
| Pin1 | TESTOUT | 未接続 | テスト端子・接続禁止 |
| Pin2 | IRQ_N | CH32V003 GPIO（TBD） | 割り込み出力・オプション |
| Pin3 | SO | SPI_MISO | |
| Pin4 | SI | SPI_MOSI | |
| Pin5 | SCK | SPI_SCK | |
| Pin6 | SS_N | CS_YMF（CH32V003 GPIO） | 専用CS信号 |
| Pin7 | RST_N | CH32V003 GPIO（TBD）・10kΩプルアップ→IOVDD | リセット信号 |
| Pin8 | REG_C/VDD | 3.3V＋100nFデカップリング | 内部レギュレータ出力 |
| Pin9 | IOVDD | VCC_3.3V | I/O電源 |
| Pin10 | VREF | 100nF → GND | 基準電圧デカップリング |
| Pin11 | SPVDD | AMP_5.0V＋47µFデカップリング | スピーカーアンプ電源 |
| Pin12 | SPVSS | GND（アナログ） | スピーカーアンプGND |
| Pin13 | （接続禁止） | 未接続 | 電気的完全絶縁 |
| Pin14 | SPOUT1/LINEOUT1 | R1→AUDIO_DIFF_P | 差動出力+側 |
| Pin15 | （接続禁止） | 未接続 | 電気的完全絶縁 |
| Pin16 | SPOUT2/LINEOUT2 | R2→AUDIO_DIFF_N | 差動出力−側 |
| Pin17 | A_TEST | 未接続 | |
| Pin18 | VSS | GND | デジタルGND |
| Pin19 | TESTMODE0 | GND | GND固定必須 |
| Pin20 | TESTMODE1 | GND | GND固定必須 |
| Pin21 | TESTINA | GND | GND固定必須 |
| Pin22 | TESTINB | GND | GND固定必須 |
| Pin23 | XI | 水晶接続 | 12.288MHz |
| Pin24 | XO | 水晶接続 | 12.288MHz |

**クロック：** 12.288MHz固定。SMD3225-4Pパッケージ水晶をYMF825直近に配置。

### 4.4 差動→SE変換回路（U3: MCP6001）

ユニティゲイン差動アンプ構成。全抵抗10kΩ±1%（JLCPCB Basic）。

- AUDIO_DIFF_P → R6 → U3 VIN+（Pin3）
- AUDIO_DIFF_N → R7 → U3 VIN−（Pin4）
- R8（フィードバック）：U3 VIN−（Pin4）← U3 VOUT（Pin1）
- R9（基準）：U3 VIN+（Pin3）→ GND
- U3 VOUT（Pin1）→ AUDIO_MONO
- U3 VDD（Pin5）：VCC_3.3V＋C8（100nF）デカップリング

**抵抗精度について：** ±1%抵抗のCMRRは約40dB。YMF825自体の残留ノイズ（−85dBV）がボトルネックであるため±0.1%精度は不要。

### 4.5 スピーカーアンプ・ミュート回路（U1: PAM8302A）

**ハードウェア完結ミュート機構（GPIO不要）：**

| 状態 | ジャック内部 | IN−の状態 | スピーカー |
|---|---|---|---|
| プラグ**未挿入** | NCスイッチ**閉**（Tip端子と短絡） | C5→Tip（GND）へ接地 | **鳴る** |
| プラグ**挿入** | NCスイッチ**開** | 完全フローティング | **消音** |

- U1 IN+（Pin4）：AUDIO_MONO_MIX → C2（1.0µF）経由
- U1 IN−（Pin3）：J1 Left NCスイッチ端子 → C5（1.0µF）経由
- J1 Left Tip端子：GND固定
- U1 SD（Pin7）：R5（10kΩ）プルアップ → AMP_5.0V（常時アクティブ）
- U1 VDD（Pin6）：AMP_5.0V＋C1（10µF）デカップリング
- U1 VO+（Pin5）→ SPK+、U1 VO−（Pin8）→ SPK−

### 4.6 ヘッドフォンジャック（J1: MJ-495A）

| 項目 | 仕様 |
|---|---|
| 型番 | MJ-495A（秋月電子 112478） |
| 形式 | 3極（TRS）・NCスイッチ付き |
| 配置 | B基板長辺（27.8mm側）端・プラグ挿入口をケース外へ突出 |
| フットプリント | 14.3×12mm（スルーホール固定脚のみ基板内） |
| スイッチ構造 | パターンA：プラグ未挿入時TipとNCスイッチ端子が短絡、挿入で開放 |

### 4.7 SPIバス接続

| 信号 | CH32V003ピン | 備考 |
|---|---|---|
| SPI_SCK | PC5 | 共通バス |
| SPI_MOSI | PC6 | 共通バス |
| SPI_MISO | PC7 | 共通バス |
| CS_YMF | PD2 | 専用CS・確定済み |
| RST_YMF | PD7（NRST共通） | MCUリセットと共通接続・プルアップにより未接続時も自動解除 |
| IRQ_N | PD6（オプション） | 演奏完了・FIFO状態通知・ポーリングで代替可 |

---

## 5. 部品一覧（BOM）

### 自動実装（JLCPCB PCBA委託）　23点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 |
|---|---|---|---|---|---|
| U1 | PAM8302AAYCR Dクラスアンプ | MSOP-8 | C11117 | Basic | 1 |
| U3 | MCP6001T-I/OT オペアンプ | SOT-23-5 | C116490 | **Extended** | 1 |
| C1 | 10µF/10V/X5R（U1 VCC平滑） | 1608 | C19702 | Basic | 1 |
| C2 | 1.0µF/16V/X7R（U1 IN+カップリング） | 1608 | C15849 | Basic | 1 |
| C5 | 1.0µF/16V/X7R（U1 IN−側カップリング） | 1608 | C15849 | Basic | 1 |
| C3 | 1000pF/50V/NPO（AUDIO_DIFF_P RCフィルタ） | 1608 | C15984 | Basic | 1 |
| C4 | 1000pF/50V/NPO（AUDIO_DIFF_N RCフィルタ） | 1608 | C15984 | Basic | 1 |
| C6 | 100nF/50V/X7R（YMF825 VCC直近） | 1608 | CC0603KRX7R9BB104 | Basic | 1 |
| C7 | 100nF/50V/X7R（YMF825 アナログ電源直近） | 1608 | CC0603KRX7R9BB104 | Basic | 1 |
| C8 | 100nF/50V/X7R（MCP6001 VDD直近） | 1608 | CC0603KRX7R9BB104 | Basic | 1 |
| C_SPVDD | 47µF/16V/X5R（SPVDD高周波リプル吸収・Murata GRM32ER61C476KE15L） | 1210 | C77101 | Basic | 1 |
| C_XTL1 | 12pF/50V/NPO（X1 XIピン側負荷容量） | 1608 | C1547 | Basic | 1 |
| C_XTL2 | 12pF/50V/NPO（X1 XOピン側負荷容量） | 1608 | C1547 | Basic | 1 |
| C_LPF | 15nF/50V/NPO（AUDIO_IN RC-LPF・fc≈10.6kHz） | 1608 | C1644 | Basic | 1 |
| R1 | 1kΩ/±1%（AUDIO_DIFF_P RCフィルタ） | 1608 | C21190 | Basic | 1 |
| R2 | 1kΩ/±1%（AUDIO_DIFF_N RCフィルタ） | 1608 | C21190 | Basic | 1 |
| R_LPF | 1kΩ/±1%（AUDIO_IN RC-LPF） | 1608 | C21190 | Basic | 1 |
| R3 | 10kΩ/±1%（ミックス抵抗） | 1608 | C25804 | Basic | 1 |
| R4 | 10kΩ/±1%（ミックス抵抗） | 1608 | C25804 | Basic | 1 |
| R_PWM | 10kΩ/±1%（AUDIO_IN混合抵抗） | 1608 | C25804 | Basic | 1 |
| R5 | 10kΩ/±1%（U1 SD プルアップ） | 1608 | C25804 | Basic | 1 |
| R6〜R9 | 10kΩ/±1%（U3 差動アンプ構成抵抗） | 1608 | C25804 | Basic | 4 |
| R10 | 100Ω/±1%（ジャックL出力保護） | 1608 | C22775 | Basic | 1 |
| R11 | 100Ω/±1%（ジャックR出力保護） | 1608 | C22775 | Basic | 1 |

### 手はんだ（ユーザー後付け）　4点

| RefDes | 部品名・仕様 | パッケージ | 備考 |
|---|---|---|---|
| U2 | YMF825-EZ FM音源LSI | SSOP-24 | ユーザー保有在庫を実装 |
| X1 | 水晶振動子 ABM8G-12.288MHZ-B4Y-T | SMD 3225-4P | CL=10pF・DigiKey調達・手はんだ |
| J1 | MJ-495A 3.5mmステレオミニジャック | スルーホール5ピン | NCスイッチ付き・ハードウェアミュートの要 |
| JP1 | ジャンパ（RC-LPFバイパス） | 1608 | デフォルト未実装（オープン）。アナログ直入力時にはんだブリッジまたは0Ω抵抗を実装してショート |

---

## 6. Claude Code用 回路図自動生成指示書

本セクションはKiCadスケマ生成時にClaude Codeへそのまま投入するための英語指示書である。本書をマスターとして管理し、回路変更時は本セクションも合わせて更新すること。

```text
# Claude Code Instruction: KiCad Schematic Netlist Generation
# for CAR-01 Audio Board (B-board) Rev.2 (Final Confirmed Version)

## 1. Context and Constraint
- Target Board: YMF-825board Compatible Audio Module (CAR-01 Standard)
- Dimensions: 19mm x 27.8mm, PCB Thickness 0.8mm
- Manufacturing: All resistors, capacitors, U1 and U3 MUST use
  JLCPCB PCBA (SMD).
- Footprint Size for passives: 1608 (0603 inch) metric.

## 2. Component Assignments and RefDes
- U1 : PAM8302AAYCR (MSOP-8)
        Footprint: Package_SO:MSOP-8_3x3mm_P0.65mm
        LCSC: C11117                        -> JLCPCB PCBA Target
- U2 : YMF825-EZ (SSOP-24)                 -> Hand-solder footprint only
- U3 : MCP6001T-I/OT (SOT-23-5)
        Footprint: Package_TO_SOT_SMD:SOT-23-5
        LCSC: C116490                       -> JLCPCB PCBA Target (Extended)
- X1 : ABM8G-12.288MHZ-B4Y-T (SMD 3225-4P, CL=10pF)
        -> Hand-solder footprint only (DigiKey procurement)
- J1 : MJ-495A, 3.5mm Stereo Mini Jack, 5-Pin Thru-hole
        Switch type: Pattern-A NC
        (Tip shorted to Switch pin when plug absent;
         opens upon plug insertion)         -> Hand-solder footprint only
- C1 : 10uF / 10V / X5R (1608)   LCSC: C19702    -> JLCPCB PCBA Target
- C2 : 1.0uF / 16V / X7R (1608)  LCSC: C15849    -> JLCPCB PCBA Target
- C3 : 1000pF / 50V / NPO (1608) LCSC: C15984    -> JLCPCB PCBA Target
- C4 : 1000pF / 50V / NPO (1608) LCSC: C15984    -> JLCPCB PCBA Target
- C5 : 1.0uF / 16V / X7R (1608)  LCSC: C15849    -> JLCPCB PCBA Target
- C6 : 100nF / 50V / X7R (1608)  LCSC: CC0603KRX7R9BB104 -> JLCPCB PCBA Target
- C7 : 100nF / 50V / X7R (1608)  LCSC: CC0603KRX7R9BB104 -> JLCPCB PCBA Target
- C8 : 100nF / 50V / X7R (1608)  LCSC: CC0603KRX7R9BB104 -> JLCPCB PCBA Target
        (U3 MCP6001 VDD bypass capacitor)
- C_XTL1: 12pF / 50V / NPO (1608) LCSC: C1547   -> JLCPCB PCBA Target
        (X1 crystal load capacitor, XI side)
- C_XTL2: 12pF / 50V / NPO (1608) LCSC: C1547   -> JLCPCB PCBA Target
        (X1 crystal load capacitor, XO side)
- C_SPVDD: 47uF / 16V / X5R (1210) LCSC: C77101 -> JLCPCB PCBA Target
        (Murata GRM32ER61C476KE15L, SPVDD ripple bypass)
- R1 : 1kOhm / +/-1% (1608)   LCSC: C21190  -> JLCPCB PCBA Target
- R2 : 1kOhm / +/-1% (1608)   LCSC: C21190  -> JLCPCB PCBA Target
- R3 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
- R4 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
- R5 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
- R6 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
        (U3 differential amp input resistor, IN+ side)
- R7 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
        (U3 differential amp input resistor, IN- side)
- R8 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
        (U3 differential amp feedback resistor)
- R9 : 10kOhm / +/-1% (1608)  LCSC: C25804  -> JLCPCB PCBA Target
        (U3 differential amp reference resistor to GND)
- R10: 100Ohm / +/-1% (1608)  LCSC: C22775  -> JLCPCB PCBA Target
        (Jack Left channel output protection resistor)
- R11: 100Ohm / +/-1% (1608)  LCSC: C22775  -> JLCPCB PCBA Target
        (Jack Right channel output protection resistor)

## 3. Netlist and Wiring Specifications

### Power and Ground Lines
- Connect AMP_5.0V to U1 VDD (Pin 6).
  Place C1 (10uF) directly between U1 VDD and U1 GND (Pin 2).
- Connect AMP_5.0V to C_SPVDD. Connect C_SPVDD opposite side to GND.
  Place C_SPVDD as close as possible to U2 SPVDD pin (Pin 11).
- Connect VCC_3.3V to U2 VDD pins (per YMF825 datasheet).
  Place C6 and C7 (100nF each) directly between VCC_3.3V and GND,
  as close as possible to U2 power pins.
- Connect VCC_3.3V to U3 VDD (Pin 5).
  Place C8 (100nF) directly between U3 VDD and GND.
- Connect all GND pins to unified net GND.

### YMF825 (U2) Crystal Oscillator
- Connect X1 between U2 Pin 23 (XI) and U2 Pin 24 (XO).
- Connect C_XTL1 (12pF) from U2 Pin 23 (XI) to GND.
- Connect C_XTL2 (12pF) from U2 Pin 24 (XO) to GND.

### YMF825 (U2) Analog Output and RC Filter
- IMPORTANT: YMF825 has a single monaural 16-bit DAC.
  Pin 14 (SPOUT1/LINEOUT1) and Pin 16 (SPOUT2/LINEOUT2) are
  a differential (BTL) output pair of the same mono signal.
  There is NO stereo output.
- U2 Pin 14 (SPOUT1/LINEOUT1) -> R1 (1kOhm) -> Net AUDIO_DIFF_P.
  Connect C3 (1000pF) from AUDIO_DIFF_P to GND.
- U2 Pin 16 (SPOUT2/LINEOUT2) -> R2 (1kOhm) -> Net AUDIO_DIFF_N.
  Connect C4 (1000pF) from AUDIO_DIFF_N to GND.
- U2 Pin 13: No Connection (must be electrically isolated).
- U2 Pin 15: No Connection (must be electrically isolated).
- U2 Pin 19 (TESTMODE0): Connect to GND.
- U2 Pin 20 (TESTMODE1): Connect to GND.
- U2 Pin 21 (TESTINA): Connect to GND.
- U2 Pin 22 (TESTINB): Connect to GND.
- U2 Pin 1 (TESTOUT): No Connection.
- U2 Pin 17 (A_TEST): No Connection.

### Differential to Single-Ended Converter (U3: MCP6001)
- Build a standard differential amplifier circuit with unity gain
  (all resistors = 10kOhm) using U3.
- Connect AUDIO_DIFF_P to R6 (10kOhm).
  Connect R6 opposite side to U3 VIN+ (Pin 3).
- Connect AUDIO_DIFF_N to R7 (10kOhm).
  Connect R7 opposite side to U3 VIN- (Pin 4).
- Connect R8 (10kOhm) from U3 VIN- (Pin 4) to U3 VOUT (Pin 1).
  (This is the feedback resistor.)
- Connect R9 (10kOhm) from U3 VIN+ (Pin 3) to GND.
  (This is the reference resistor for common-mode rejection.)
- U3 VOUT (Pin 1) -> Net AUDIO_MONO.

### Stereo Jack (J1) Output Routing
- Connect AUDIO_MONO_MIX to R10 (100Ohm).
  Connect R10 opposite side to J1 Left Tip Pin.
- Connect AUDIO_MONO_MIX to R11 (100Ohm).
  Connect R11 opposite side to J1 Right Ring Pin.
- Connect J1 Left Tip Pin also to GND.
  (Tip is grounded to serve as NC switch reference for mute logic.)
  NOTE: Headphone output is taken from AUDIO_MONO_MIX (after AUDIO_IN mix),
  so both FM audio and AUDIO_IN source are audible on headphones.

### Audio Mix to Amplifier
- Connect AUDIO_MONO to R3 (10kOhm).
- Connect AUDIO_MONO to R4 (10kOhm).
- Combine opposite sides of R3 and R4 -> Net AUDIO_MONO_MIX.

### AUDIO_IN RC-LPF, Bypass Jumper and Mix
- Add external analog-side edge pad AUDIO_IN as the entry point for
  external audio mixing (ADPCM PWM from UIAPduino PC3, or analog
  audio from mp3 decoder etc.).
- Connect AUDIO_IN to R_LPF (1kOhm).
  Connect R_LPF opposite side to Net PWM_FILTERED.
  Connect C_LPF (15nF) from PWM_FILTERED to GND.
  (RC low-pass filter: fc = 1/(2*pi*1k*15n) ≈ 10.6kHz)
- Place JP1 footprint (1608) as bypass jumper between AUDIO_IN and PWM_FILTERED.
  JP1 default: OPEN (not populated). RC-LPF active. Use for PWM input.
  JP1 shorted: populate with solder bridge or 0Ohm resistor.
  RC-LPF bypassed. Use for direct analog audio input.
- Connect PWM_FILTERED to R_PWM (10kOhm).
  Connect R_PWM opposite side to Net AUDIO_MONO_MIX.

### Amplifier SD Pin Treatment
- Connect U1 SD (Pin 7) to R5 (10kOhm).
- Connect opposite side of R5 to AMP_5.0V.
  (Amplifier permanently enabled; power control by A-board MOSFET.)

### Hardware Insertion Mute and Amplifier Input
- Connect AUDIO_MONO_MIX to C2 (1.0uF).
- Connect C2 opposite side to U1 IN+ (Pin 4).
- Connect J1 Left NC Switch Pin to C5 (1.0uF).
- Connect C5 opposite side to U1 IN- (Pin 3).
- [Mute Logic]
    Jack empty  : NC switch CLOSED -> Left Switch Pin shorted to
                  Tip Pin (GND) -> IN- tied to GND via C5.
                  Amplifier operates normally. Speaker active.
    Jack inserted: NC switch OPENS -> IN- floats completely.
                  Amplifier loses differential reference.
                  Speaker output muted automatically.

### Speaker Output
- U1 VO+ (Pin 5) -> External edge pad SPK+
- U1 VO- (Pin 8) -> External edge pad SPK-

### Digital Control Interface (Edge Pads)
- Expose from U2:
    Pin 2  (IRQ_N) -> edge pad IRQ_N (optional, for sequencer/FIFO notification)
    Pin 3  (SO)    -> edge pad SPI_MISO (optional, for read-back/debug)
    Pin 4  (SI)    -> edge pad SPI_MOSI
    Pin 5  (SCK)   -> edge pad SPI_SCK
    Pin 6  (SS_N)  -> edge pad CS_YMF
    Pin 7  (RST_N) -> edge pad RST_YMF (pull up to VCC_3.3V via 10kOhm)
- Add edge pad AUDIO_IN as analog-side edge pad for external audio mixing input.

Generate complete KiCad schematic netlist and Python build script
based on this finalized architecture.
```

---

## 7. 外部接続・インターフェース

### 7.1 外部接続インターフェース

B基板の信号はすべてCAR-01規格エッジパッドから引き出される。CAR-01スタック内のUIAPduino経由でも、Xiao・Waveshare ESP32・PB-1000等の外部デバイスから直接でも、同一エッジパッドを介して接続する。

**デジタル側エッジパッド**

| 信号 | 方向 | 備考 |
|---|---|---|
| VCC_3.3V | 受信 | YMF825・MCP6001電源 |
| GND | 共通 | |
| SPI_SCK | 受信 | |
| SPI_MOSI | 受信 | |
| SPI_MISO | 送信 | 通信確立確認（I_ADR#80）等に使用。ピン制約時はオプション |
| CS_YMF | 受信 | 専用CS |
| RST_YMF | 受信 | システムリセットと兼用可。未接続時はプルアップにより自動解除 |
| IRQ_N | 送信 | 演奏完了・FIFO状態通知。ピン制約時はオプション |

**アナログ側エッジパッド**

| 信号 | 方向 | 備考 |
|---|---|---|
| AMP_5.0V | 受信 | PAM8302A・SPVDD電源 |
| GND | 共通 | |
| SPK+ | 送信 | スピーカー出力 |
| SPK− | 送信 | スピーカー出力 |
| AUDIO_IN | 受信 | 外部音声混合入力。JP1オープン時はRC-LPF経由（PWM入力用）・JP1ショート時はRC-LPFバイパス（アナログ直入力用） |

**MISOとIRQ_Nのピン制約時の選択指針**

両信号は用途が異なり電気的共有は不可。ピン数に制約がある場合は用途に応じて選択する。

| 状況 | 推奨 |
|---|---|
| 開発・デバッグ中 | MISO優先（通信確立確認が先決） |
| 完成品・ピン不足 | IRQ_N優先（演奏完了検出の実用価値が高い） |
| ピン十分 | 両方接続 |

**電圧注意：** YMF825-EZのIOVDDは3.3V。5V系デバイス（PB-1000等）から接続する場合はレベルシフタが必要。Xiao・Waveshare ESP32シリーズ等3.3V系はそのまま接続可。

**2枚同期（ステレオ化）について：** YMF825チップに複数チップ間の同期専用ピンは存在しない。2枚同期はSCK・CS・RST_YMFを2枚で共有しMOSIのみ分岐する方式で実現可能であり、追加エッジパッドは不要。

### 7.2 ケース外部インターフェース

| 端子 | 配置 | 備考 |
|---|---|---|
| ヘッドフォンジャック（MJ-495A） | B基板長辺端・ケース側面穴から突出 | 3.5mmステレオミニ |
| スピーカーコネクタ（SPK+/SPK−） | B基板上（ケース内） | 内蔵スピーカー接続 |

---

## 8. ファームウェア仕様（YMF825制御）

### 8.1 SPI通信パラメータ

| 項目 | 値 |
|---|---|
| SPIモード | Mode 0（CPOL=0・CPHA=0） |
| 最大クロック | 10MHz |
| ビット順 | MSB first |
| CS極性 | アクティブLOW |

### 8.2 初期化シーケンス

アプリ起動時にIAPコンテキストスイッチで実行される初期化ブロックでYMF825の初期化・音色設定を完了させる。アプリ本体からはレジスタ書き込みのみ行う（再初期化不要）。詳細はcartridge_spec_master第1部1.8節「オーディオ機能の位置づけ」に準拠する。

### 8.3 CPU使用率（参考）

| タスク | CPU使用率 |
|---|---|
| YMF825 SPI送信 | 0.124% |
| SPI FRAMアクセス（演奏中） | 0.36% |

---

## 9. 未決事項（PENDING）

| # | 項目 | 優先度 | 備考 |
|---|---|---|---|
| 1 | ~~CS_YMF・RST_YMF・IRQ_NのCH32V003ピン番号確定~~ | ~~高~~ | **解消済み**（CS_YMF=PD2・RST_YMF=PD7共通・IRQ_N=PD6） |
| 2 | ~~C_SPVDD LCSC番号確定~~ | ~~中~~ | **解消済み**（C77101・Murata GRM32ER61C476KE15L） |
| 3 | ~~水晶負荷容量確定（C_XTL追加要否）~~ | ~~中~~ | **解消済み**（C_XTL1/2・12pF・C1547・LCSC C1547） |
| 4 | KiCadスケマ生成（Claude Code実施） | — | 本書確定後に着手 |
| 5 | PCBレイアウト（SPOUT1/2ガードビア・GNDゾーン反映） | — | スケマ完成後 |
| 6 | YMF825音色設計（15種効果音） | — | cartridge_spec_master PENDING #6 |

---

## 10. 設計根拠メモ

### SPVDD LDO不採用の理由

YMF825のSPVDDはスピーカーアンプ段のみに供給される。DACコア段はVDD（3.3V）で動作するため、LINEOUT使用時にSPVDDリプルがDAC出力に直接乗る経路は限定的である。後段のカップリングコンデンサ（C2/C5）が低周波リプルを遮断し、YMF825自体の残留ノイズ（−85dBV）がすでにボトルネックとなっている。MT3608出力5.0V→LDO後の電圧降下でSPVDD下限（4.5V）にも近接するリスクもあり、LDO不採用が合理的と判断。SPVDD直近47µF（1210）で高周波リプルを吸収する。

### 水晶振動子（X1）選定根拠

ABM8G-12.288MHZ-B4Y-T（Abracon、CL=10pF、SMD 3225-4P）をDigiKeyで調達・手はんだ。秋月電子およびJLCPCBには12.288MHz・SMD 3225-4P品の取り扱いがないため。

外付け負荷容量コンデンサの計算：

```
C_ext = 2 × (CL − C_stray) = 2 × (10 − 4) = 12pF
```

C_XTL1・C_XTL2として各12pF（1608 NPO、LCSC: C1547）をPCBAで実装。



3極スイッチ付きジャックとして小型品（短辺6mm台）を探索したが、ハードウェアスイッチ機能を持つ製品はいずれも幅広（12mm級）であることが判明。プラグ挿入口をケース外に突出させる配置により物理制約を回避する。ソフトウェアミュート（GPIO制御）は排除し、ハードウェア完結ミュートを優先した。

### MCP6001採用理由

差動→SE変換にオペアンプを用いる理由はCMRRによるデジタルノイズキャンセルである。SOT-23-5（シングル）1個で差動アンプ回路を構成でき、B基板の面積制約に適合する。LCSC C116490（Extended、段取り費+$3）。

### YMF825接続禁止ピン（Pin13・Pin15）

データシートで「接続禁止（No Connection）」と明記。誤接続は音質劣化または破損の原因となる。

### 将来的なB基板単体利用について

VCC・GND・MOSI・RST_N・SCK・SS_Nの6本のみでYMF825を制御可能（PB-1000接続実験より確認）。B基板を外部機器から単体利用する際のインターフェースとして参考とする。

---

## 11. 変更履歴

| 日付 | バージョン | 内容 |
|---|---|---|
| 2026-06-06 | Rev.1.0 | 初版作成 |
| 2026-06-06 | Rev.1.1 | B基板の筐体内位置を修正（左半分最下層）・積層構成図を全面書き直し |
| 2026-06-06 | Rev.1.2 | 基板サイズを27.8×19mmに修正 |
| 2026-06-07 | Rev.2.0 | Rev.2（Claude Code投入用マスター）と統合。オペアンプをTLV9062→MCP6001（SOT-23-5）に変更。ミュート方式をSD_N制御→PAM8302A差動入力利用方式に変更。SPVDD LDO不採用・47µF/1210追加に変更。電源ライン名をAMP_5.0V/VCC_3.3Vに統一。信号フロー図追加。BOMをLCSC番号付きで全面更新。 |
| 2026-06-07 | Rev.2.1 | セクション6.1を全面改訂。デジタル/アナログ側エッジパッド分離。MISO・IRQ_Nをオプション明記・選択指針追加。外部デバイス接続・2枚同期・電圧注意事項を記載。 |
| 2026-06-08 | Rev.2.2 | 水晶X1型番をABM8G-12.288MHZ-B4Y-T（CL=10pF）に確定。C_XTL1・C_XTL2（12pF NPO C1547）追加。C_SPVDD LCSC番号をC77101に確定。PENDING #2・#3解消。PCBA合計20点。 |
| 2026-06-08 | Rev.2.3 | Rev.2.2との整合確認・設計根拠メモに水晶選定根拠・負荷容量計算を追加。 |
| 2026-06-08 | Rev.2.4 | Claude Code用回路図自動生成指示書（旧Rev.2.2セクション3）を統合・セクション6として挿入。本書をマスタードキュメントとして一本化。セクション番号を全面更新。 |
| 2026-06-08 | Rev.2.5 | セクション7見出し行欠落を修正。セクション8内部見出しを8.x に修正。CS_YMFをPD2に確定。PENDING #1解消。 |
| 2026-06-09 | Rev.2.6 | §4.7 RST_YMFをPD7（NRST）共通接続に確定。IRQ_NをPD6（オプション）に確定。PENDING #1完全解消。 |
| 2026-06-09 | Rev.2.7 | §2.2 出力構成にADPCM混合を追記。§4.2 信号フローにADPCM RC-LPF→AUDIO_MONO_MIX混合経路を追加。ヘッドフォン出力分岐点をAUDIO_MONOからAUDIO_MONO_MIX（ADPCM混合後）に変更。§5 BOMにC_LPF・R_LPF・R_PWMを追加（20点→23点）。§6 KiCad指示書にADPCM混合回路・PWM_AUDIOエッジパッド・ヘッドフォン分岐点変更を反映。§7.1 エッジパッドにPWM_AUDIO追加。 |
| 2026-06-09 | Rev.2.8 | パッド名PWM_AUDIO→AUDIO_INに変更・アナログ側エッジパッドに移動。JP1（RC-LPFバイパスジャンパ・1608フットプリント・デフォルト未実装）を追加。JP1オープン=PWM入力・JP1ショート（はんだブリッジまたは0Ω抵抗）=アナログ直入力。手はんだ部品にJP1を追加（3点→4点）。§2.2・§4.2・§6・§7.1をAUDIO_IN・JP1構成に全面更新。 |

