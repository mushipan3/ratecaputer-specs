# CAR-01規格 A基板（インテリジェント電源管理モジュール）設計仕様書

## 完全確定版 Rev.2 — Claude Code投入用マスタードキュメント
## 更新内容：SW1を外付け2ピンパッドに変更・TBD部品全件確定

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（A基板）は、ラテカピュータ（CAR-01規格）に準拠した自律型電源管理モジュール基板である。ATtiny202をコプロセッサとして搭載し、メインMCU（UIAPduino等）のSRAM・フラッシュ・CPU時間を一切消費せずに、バッテリー残量監視・電源シーケンス制御・極小OLED描画・接続通知BEEPを現地完結させる。10mm × 28mmの片面実装スティック形状に全回路を収める。

### 物理仕様・製造要件

| 項目 | 仕様 |
|------|------|
| 基板サイズ | 10mm × 28mm（縦28mmはB基板・C基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | Top面（表面）のみへの片面実装。裏面は全面ベタGNDプレーン |
| 基板色 | Green または Black（最安・最短納期を優先） |
| 発注枚数 | 5パネル（パネル内7枚取り → 計35枚） |
| 外部接続端子 | 基板端エッジ寄りベタパッド × 7個（カスタレーションなし） |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** ATtiny202・IRLML6402・Buck DCDC IC・MT3608・受動部品（1608サイズ）の全品
- **手はんだ（ユーザー後付け）：** BZ1（圧電スピーカー）・SW1外付けスイッチ本体
- **SW1について：** タクトスイッチ本体は基板に実装せず、ケースへの収まり具合を確認しながら外付けする。基板上には2ピンのスルーホールパッドのみ設ける。スイッチ本体からリード線で接続する。

### システムブロックおよび電気的トポロジー

#### 電源アーキテクチャ

本基板はリポ1セル（3.0V〜4.2V）からの生電圧（VBAT_IN）を主入力とする二系統の電源ラインを持つ。

**常時通電ライン（VBAT_IN）：** リポからATtiny202の1番ピン（VCC）へ直結。電源OFF時はATtiny202がパワーダウンスリープ（数µA）で待機し、バッテリー消費を最小化する。

**スイッチド下流ライン（VCC_3V3_OUT / VCC_5V_OUT）：** ATtiny202の7番ピン（PA3）がLOWを出力するとP-ch MOSFET（IRLML6402）が導通し、下流のDCDC回路（3.3V降圧BuckおよびMT3608による5V昇圧）へ電流が流れ、UIAPduinoおよびYMF-825を同時駆動する。

#### I2Cバストポロジー

ATtiny202を**常時I2Cマスター**、UIAPduinoを**I2Cスレーブ**として定義する。ATtiny202が主導権を握り、SSD1306 OLED（アドレス0x3C）への画面ストリーミングと、UIAPduino（スレーブ）へのバッテリー残量データ送信をすべて自律実行する。

### ATtiny202 ピンアサイン（厳格定義）

```
              ATtiny202 (SOIC-8)
           ┌───────┴───────┐
  VBAT_IN ─┤ 1. VCC   GND 8 ├─── GND_IN
LtcBus_SYS─┤ 2. PA1   PA3 7 ├─── MOSFETゲート制御（10kΩでVBATへPU）【基板内完結】
 _INT/SPK  │               │
  INT_SDA ─┤ 3. PA2   PA0 6 ├─── LOCAL_UPDI（自基板内UIAPduino専用）【基板内完結】
  INT_SCL ─┤ 4. PA4   PA5 5 ├─── タクトSW外付け2ピンパッド & バッテリー分圧監視回路【基板内完結】
           └───────────────┘
```

### 回路設計コアアーキテクチャ

#### 1. PA1：時系列マルチプレクス（1ピン2役）
基板外の拡張コネクタ線LtcBus_SYS_INTおよび圧電スピーカーのプラス側を並列に接続する。

#### 2. PA5：アナログ・マルチプレクス（1ピン2役・基板内完結）
100kΩ抵抗2本（±1%）によるバッテリー電圧1/2分圧回路の電位点に接続。同ピンにSW1の2ピンパッド（Pin1）を接続する。スイッチ本体は外付けとし、基板上はパッドのみ。

- **ボタン非押下時：** バッテリー電圧の半分（約1.5V〜2.1V）をADCで測定
- **ボタン押下時：** ピンが強制的にGNDへ落ちるため（0V）、電源ON／長押しシャットダウンイベントを検出

#### 3. PA3：MOSFETゲート制御（基板内完結・ハードウェアフェイルセーフ）
P-ch MOSFET（IRLML6402）のゲートへ直結。10kΩのプルアップ抵抗でVBAT_INへ常時ハードウェアプルアップする。

#### 4. PA2 / PA4：I2Cバス（3者兼務）
INT_SDA / INT_SCLとして、自基板内蔵のUIAPduino・SSD1306 OLED・対外エッジパッドの3者が同一バスに乗る。

#### 5. PA0：LOCAL_UPDI（基板内完結）
自基板内のUIAPduino専用。他デバイスへは一切露出させない。

#### 6. 3.3V降圧Buck DCDC（U3: RT8008-33GB / C130801）
Vin 2.5V〜5.5V、Vout 3.3V固定、600mA。リポ放電末期（3.0V入力）でも3.3V出力を維持。

#### 7. 5V昇圧DCDC（U4: MT3608 / C84817）
Vin 2V〜24V、Iout最大2A。YMF-825音源モジュール駆動用5V昇圧回路。

#### 8. MT3608 フィードバック抵抗（Vout = 5V）
Vref = 0.6V。Vout = 0.6×(1 + R_upper/R_lower) = 5.0V
→ R_lower = 100kΩ、R_upper = 750kΩ（C863934）

---

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# Claude Code Instruction: KiCad Schematic Netlist Generation
# for CAR-01 Power Management Board A (Rev.2 Final)
# DO NOT read any existing files. Generate from this instruction only.

## 1. Context & Constraint
- Target Board     : A-Board, Intelligent Power Management Module (CAR-01)
- Dimensions       : 10mm x 28mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only.
                     Bottom side = solid GND copper pour, no routing.
- Manufacturing    : ALL IC/passive components JLCPCB PCBA (full factory assembly).
                     SW1 = 2-pin through-hole pad only (NO component, NO PCBA).
                     BZ1 = hand-solder by user after delivery.
- Footprint Size   : 1608 (0603 inch) metric for all passive components.
- Edge Pads        : 7x flat edge pads (no castellations) for external connection.

## 2. Component Assignments & RefDes (ALL TBD RESOLVED)

### JLCPCB PCBA Target
- U1  : ATtiny202 (SOIC-8)              LCSC: C2052970   Extended
- U2  : IRLML6402 P-ch MOSFET (SOT-23) LCSC: C5148470   Extended
- U3  : RT8008-33GB Buck DC-DC (SOT-23-5) LCSC: C130801  Extended ※治具必要
- U4  : MT3608 Boost DC-DC (SOT-23-6)  LCSC: C84817     Extended
- R1  : 10kΩ ±1% (1608)               LCSC: C25804     Basic   MOSFETゲートPU
- R2  : 100kΩ ±1% (1608)              LCSC: C25803     Basic   分圧上側
- R3  : 100kΩ ±1% (1608)              LCSC: C25803     Basic   分圧下側
- R4  : 750kΩ ±1% (1608)              LCSC: C863934    Extended MT3608 FB上側
- R5  : 100kΩ ±1% (1608)              LCSC: C25803     Basic   MT3608 FB下側
- C1  : 0.1µF / 10V / X7R (1608)      LCSC: C14663     Basic   分圧ノードパスコン
- C2  : 10µF / 10V / X5R (1608)       LCSC: C19702     Basic   Buck入力バルク
- C3  : 10µF / 10V / X5R (1608)       LCSC: C19702     Basic   Buck出力バルク
- C4  : 0.1µF / 10V / X7R (1608)      LCSC: C14663     Basic   Buck出力バイパス
- C5  : 10µF / 10V / X5R (1608)       LCSC: C19702     Basic   MT3608入力バルク
- C6  : 22µF / 6.3V / X5R (1608)      LCSC: C59461     Basic   MT3608出力バルク
         ※電圧6.3V品。MT3608 5V出力に対し定格比1.26倍で許容範囲内。
- C7  : 0.1µF / 10V / X7R (1608)      LCSC: C14663     Basic   ATtiny202 VCCパスコン
- L1  : 4.7µH / Isat≥1.15A (4.5×4mm) LCSC: C501773    Extended Buck DCDC用
- L2  : 4.7µH / Isat≥1.15A (4.5×4mm) LCSC: C501773    Extended MT3608昇圧用
- D1  : B5819W Schottky (SOD-123)      LCSC: C8598      Basic   MT3608整流

### Hand-solder / External (NOT PCBA)
- BZ1 : 圧電スピーカー（パッシブ）— ユーザー手はんだ。LCSC番号なし。
- SW1 : 2ピン スルーホールパッドのみ（部品なし）
        PA5（DIV_NODE）← Pin1、GND ← Pin2
        スイッチ本体はケース加工後にリード線で外付け。

## 3. Netlist & Wiring Specifications

### Power Nets
- VBAT_IN      : Raw LiPo input (3.0V〜4.2V)
- GND          : Common ground
- SWITCHED_VBAT: Post-MOSFET switched rail
- VCC_3V3_OUT  : Switched 3.3V output (post-Buck)
- VCC_5V_OUT   : Switched 5V output (post-MT3608)
- DIV_NODE     : Battery voltage divider midpoint / SW1 Pin1

### Always-On Supply (ATtiny202)
- VBAT_IN -> U1 Pin1 (VCC)
- C7 (0.1µF) between U1 Pin1 (VCC) and Pin8 (GND)
- GND -> U1 Pin8 (GND)

### MOSFET Gate Control (PA3 / U2)
- U1 PA3 (Pin7) -> U2 Gate (Pin1)
- R1 (10kΩ) between U2 Gate and VBAT_IN (hardware pull-up)
- VBAT_IN -> U2 Source (Pin3)
- U2 Drain (Pin2) -> SWITCHED_VBAT

### Buck DC-DC 3.3V (U3: RT8008-33GB)
- SWITCHED_VBAT -> U3 VIN
- C2 (10µF) between U3 VIN and GND (input bulk)
- U3 SW -> L1 -> VCC_3V3_OUT
- C3 (10µF) and C4 (0.1µF) between VCC_3V3_OUT and GND (output)
- U3 EN -> VCC_3V3_OUT (enable always-on)
- U3 GND -> GND
- VCC_3V3_OUT exposed on Edge Pad 3

### MT3608 Boost 5V (U4)
- SWITCHED_VBAT -> U4 IN (Pin1)
- C5 (10µF) between U4 IN and GND (input bulk)
- U4 SW (Pin2) -> L2 -> D1 Anode
- D1 Cathode -> VCC_5V_OUT
- C6 (22µF) between VCC_5V_OUT and GND (output bulk)
- MT3608 FB resistors (Vout=5V):
    R4 (750kΩ) from VCC_5V_OUT to FB pin
    R5 (100kΩ) from FB pin to GND
    Vout = 0.6V × (1 + 750k/100k) = 5.10V ✅
- U4 EN (Pin3) -> SWITCHED_VBAT (always enabled when MOSFET on)
- VCC_5V_OUT exposed on Edge Pad 4

### PA5 Hybrid: Battery Monitor & SW1 External Pad
- VBAT_IN -> R2 (100kΩ) -> DIV_NODE -> R3 (100kΩ) -> GND
- C1 (0.1µF) between DIV_NODE and GND (noise bypass)
- DIV_NODE -> U1 PA5 (Pin5)
- SW1 Pin1 -> DIV_NODE (same net)
- SW1 Pin2 -> GND
  ※SW1はスルーホール2ピンパッドのみ。部品なし。
  　外付けスイッチをリード線でPin1-Pin2間に接続する。

### PA1: SYS_INT / Speaker Multiplex
- U1 PA1 (Pin2) -> LtcBus_SYS_INT_SPK
- BZ1 positive -> LtcBus_SYS_INT_SPK (hand-solder pad)
- BZ1 negative -> GND (hand-solder pad)
- LtcBus_SYS_INT_SPK exposed on Edge Pad 7

### PA2 / PA4: I2C Bus (INT_SDA / INT_SCL)
- U1 PA2 (Pin3) -> INT_SDA -> Edge Pad 5
- U1 PA4 (Pin4) -> INT_SCL -> Edge Pad 6
- On-board SSD1306 OLED (0.42", 72x40, I2C 0x3C) on same bus

### PA0: LOCAL_UPDI (Internal only)
- U1 PA0 (Pin6) -> UPDI pad (internal, NOT exposed to any edge pad)

## 4. Edge Pad Assignment (7 pads, flat, no castellations)
- Pad 1 : VBAT_IN
- Pad 2 : GND_IN
- Pad 3 : VCC_3V3_OUT
- Pad 4 : VCC_5V_OUT
- Pad 5 : INT_SDA
- Pad 6 : INT_SCL
- Pad 7 : LtcBus_SYS_INT_SPK

## 5. PCB Layout Rules
- ALL active components on Top layer only.
- Bottom layer: solid GND copper pour, no signal routing.
- All passive SMD components: 1608 footprint.
- L1・L2: 4.5×4mm footprint (Me-TECH CD43 series).
- SW1: Through-hole 2-pin pad, 2.54mm pitch, place at board edge.
       Footprint: Connector_PinHeader_2.54mm:PinHeader_1x02_P2.54mm_Vertical
       Label pins: Pin1=DIV_NODE, Pin2=GND
- BZ1: SMD or through-hole pad for passive piezo buzzer, place near PA1.
- U3 Buck IC: place L1 and caps close per RT8008-33GB datasheet layout.
- U4 MT3608: place L2, D1, C6 in tight switching loop.
- Edge pads: along the 28mm edge.

Generate complete KiCad schematic netlist (.kicad_sch),
PCB layout (.kicad_pcb), and Python build script.
Use KiCad MCP server for direct file generation.
```

---

## 3. 部品一覧表（BOM）Rev.2

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計18点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 | 役割 |
|--------|------------|----------|---------|------|------|------|
| U1 | ATtiny202マイコン | SOIC-8 | C2052970 | Extended | 1 | 全機能を担うコプロセッサ |
| U2 | IRLML6402 P-ch MOSFET | SOT-23 | C5148470 | Extended | 1 | スイッチド電源ライン制御 |
| U3 | RT8008-33GB Buck DCDC 3.3V固定 | SOT-23-5 | **C130801** | Extended | 1 | 3.3V降圧・リポ放電末期も安定 ※治具必要 |
| U4 | MT3608 昇圧DCDC | SOT-23-6 | C84817 | Extended | 1 | 5V昇圧・YMF-825駆動用 |
| R1 | プルアップ抵抗（10kΩ/±1%） | 1608 | C25804 | Basic | 1 | MOSFETゲートVBATプルアップ |
| R2 | 分圧抵抗・上（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・上側 |
| R3 | 分圧抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・下側 |
| R4 | MT3608 FB抵抗・上（**750kΩ**/±1%） | 1608 | **C863934** | Extended | 1 | 5V出力設定（Vout=5.10V） |
| R5 | MT3608 FB抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | 5V出力設定（基準側） |
| C1 | 分圧ノードパスコン（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ADCノイズ除去 |
| C2 | Buck入力バルク（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | Buck DCDC入力安定化 |
| C3 | Buck出力バルク（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | 3.3V出力平滑 |
| C4 | Buck出力バイパス（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | 3.3V高周波バイパス |
| C5 | MT3608入力バルク（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | MT3608入力安定化 |
| C6 | MT3608出力バルク（**22µF/6.3V/X5R**） | **1608** | **C59461** | **Basic** | 1 | 5V出力平滑 ※6.3V品で許容範囲内 |
| C7 | ATtiny202 VCCバイパス（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ATtiny202電源デカップリング |
| L1 | パワーインダクタ（**4.7µH/Isat 1.15A/4.5×4mm**） | 2012相当 | **C501773** | Extended | 1 | Buck DCDC用 ※治具必要 |
| L2 | パワーインダクタ（**4.7µH/Isat 1.15A/4.5×4mm**） | 2012相当 | **C501773** | Extended | 1 | MT3608昇圧用 ※治具必要 |
| D1 | ショットキーダイオード B5819W | SOD-123 | C8598 | Basic | 1 | MT3608整流ダイオード |

### 🔴 手はんだ / 外付け対象パーツ

| RefDes | 部品名 | 実装方法 | 備考 |
|--------|--------|---------|------|
| BZ1 | 圧電スピーカー（パッシブ） | 手はんだ | LCSC番号なし・別途手配 |
| SW1 | タクトスイッチ（外付け） | **基板上はパッドのみ** | 2ピンTHパッド（2.54mmピッチ）。ケース加工後にリード線で接続 |

---

## 4. 設計根拠メモ（Rev.2更新点）

### SW1 外付け2ピンパッド方式の根拠

タクトスイッチはフリスクケースの物理的な制約（穴加工位置・ボタン突出量）に依存するため、基板設計時点では最終位置を確定できない。基板上に2ピンのスルーホールパッドのみを設け、ケース加工後にスイッチ本体をリード線で接続する方式を採用する。これにより基板面積の節約にもなり、10mm幅基板上のレイアウト自由度が向上する。

電気的には従来と同一：PA5（DIV_NODE）← SW1 Pin1、GND ← SW1 Pin2。

### C6（22µF/6.3V）の電圧定格について

MT3608の5V出力に対して6.3V品を使用する。定格比 = 6.3V / 5.0V = 1.26倍。MT3608データシートの推奨は「定格電圧 ≥ 出力電圧 × 1.25」であり、1.26倍でギリギリ満たしている。かつC59461（Samsung）はJLCPCB Basicの定番品（在庫豊富）であるため採用。長期信頼性を重視する場合は10V品（1206サイズ）への変更を検討すること。

### L1・L2（C501773）のパッケージについて

Me-TECH CD43-4R7M-S-Z（4.5×4mm）はA基板の1608統一仕様から外れるが、4.7µH・Isat 1.15A以上を1608サイズで実現できる製品が市場に存在しないため、4.5×4mmパッケージを例外的に採用する。KiCadのフットプリントは`Inductor_SMD:L_4.5x4.0mm`または同等品を使用すること。

### R4（750kΩ/C863934）の必要性

MT3608の出力電圧式：Vout = Vref × (1 + R_upper/R_lower) = 0.6 × (1 + R4/R5)
- R4=750kΩ → Vout = 5.10V ✅
- R4=680kΩ → Vout = 4.68V ❌（5Vを下回りYMF825が動作しない）

750kΩはE96標準値であり、C863934（YAGEO RT0603FRE07750KL）として在庫確認済み。

### Extended Parts 段取り費サマリー

Extended品：U1・U2・U3・U4・L1・L2・R4 = 7品種
段取り費：7種 × $3 = **$21/パネル**（5パネルで$105）
1枚あたり：$3.00の上乗せ。工場実装品質の対価として許容。

---

*文書バージョン：Rev.2 確定版 2026-05-27*
*SW1外付け変更・TBD全件（U3/L1/L2/C6/R4/SW1）確定を反映*
*本ファイルをそのままClaude Codeへ投入してください*
