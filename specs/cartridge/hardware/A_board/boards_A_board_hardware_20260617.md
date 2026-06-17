# CAR-01規格 A基板（インテリジェント電源管理モジュール）設計仕様書

## Rev.2.2 — 2026年6月17日

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（A基板）は、ラテカピュータ（CAR-01規格）に準拠した自律型電源管理モジュール基板である。ATtiny202をコプロセッサとして搭載し、メインMCU（UIAPduino等）のSRAM・フラッシュ・CPU時間を一切消費せずに、バッテリー残量監視・充電状態監視・電源シーケンス制御・接続通知BEEPを現地完結させる。9.5mm × 27.8mmの片面実装スティック形状に全回路を収める。

### 物理仕様・製造要件

| 項目 | 仕様 |
|---|---|
| 基板サイズ | 9.5mm × 27.8mm（縦27.8mmはB基板・C基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | Top面（表面）のみへの片面実装。裏面は全面ベタGNDプレーン |
| 基板色 | Green または Black（最安・最短納期を優先） |
| 発注枚数 | 5枚（JLCPCBの最小ロット） |
| 外部接続端子 | 基板端エッジ寄りベタパッド × 9個（カスタレーションなし） |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** ATtiny202・IRLML6402・Buck DCDC IC・MT3608・MCP73831・受動部品（1608サイズ）の全品
- **手はんだ：** なし（全点工場実装）

### システムブロックおよび電気的トポロジー

#### 電源アーキテクチャ

本基板はリポ1セル（3.0V〜4.2V）からの生電圧（VBAT_IN）を主入力とする二系統の電源ラインを持つ。

**常時通電ライン（VBAT_IN）：** リポからATtiny202の1番ピン（VCC）へ直結。電源OFF時はATtiny202がパワーダウンスリープ（数µA）で待機し、バッテリー消費を最小化する。

**スイッチド下流ライン（VCC_3V3_OUT / VCC_5V_OUT）：** ATtiny202の7番ピン（PA3）がLOWを出力するとP-ch MOSFET（IRLML6402）が導通し、下流のDCDC回路（3.3V降圧BuckおよびMT3608による5V昇圧）へ電流が流れ、UIAPduinoおよびYMF-825を同時駆動する。

**充電ライン（USB_5V_IN → VBAT_IN）：** UIAPduinoの5VピンはUSB接続時にUSB VBUSの5Vを出力する。この5VをPad 9（USB_5V_IN）経由でMCP73831（充電IC）のVDDに入力し、LiPoバッテリーへCC/CV充電を行う。USB未接続時はVDD = 0Vとなり充電ICは自動停止する。拡張コネクタ下列1（5V_IO）経由の受電時も同様にMCP73831 VDDへ接続することで、上位デバイスからの給電でもバッテリー充電が可能となる（今後の拡張コネクタ見直しと合わせて設計を反映する）。

#### I2Cバストポロジー

ATtiny202を**I2Cスレーブ**、UIAPduino（CH32V003）を**I2Cマスター**として定義する。ATtiny202はUIAPduinoからのReadリクエストに対してバッテリー残量値・充電状態を返答する。OLEDへの描画・UIAPduinoへの自律送信は一切行わない。

OLEDはA基板から撤去し、D基板（ATtiny1604搭載）に移管した。

### ATtiny202 ピンアサイン（厳格定義）

```
              ATtiny202 (SOIC-8)
           ┌───────┴───────┐
  VBAT_IN ─┤ 1. VCC   GND 8 ├─── GND_IN
CART_READY─┤ 2. PA1   PA3 7 ├─── MOSFETゲート制御（10kΩでVBATへPU）【基板内完結】
 /BEEP     │               │
 /CHG_STAT │               │
  INT_SDA ─┤ 3. PA2   PA0 6 ├─── LOCAL_UPDI_A（エッジパッド経由UIAPduino PD6へ）
  INT_SCL ─┤ 4. PA4   PA5 5 ├─── タクトSW接続パッド ＆ バッテリー分圧監視回路【基板内完結】
           └───────────────┘
```

### 回路設計コアアーキテクチャ

#### 1. PA1：CART_READY / BEEP / 充電状態監視（1ピン3役・時系列多重）

PA1はフェーズによって役割が切り替わる時系列マルチプレクスピンである。

**PA1の時系列役割：**

| フェーズ | PA1のモード | 役割 |
|---|---|---|
| 初期化中 | GPIO出力 | CART_READY HIGH出力（CAR-01規格拡張コネクタ上列1） |
| BEEP中 | TCA0 PWM出力 | アンサーバックBEEP吹鳴（おまけ） |
| 通常動作中・拡張コネクタ未接続時 | GPIO入力（内部プルアップ） | MCP73831 STATピン読み取り（充電中/完了判定） |

**CART_READYシーケンス：**
1. UIAPduinoが共通プログラム初期化を完了する
2. UIAPduinoがI2C経由でATtiny202に「初期化完了」を通知する
3. ATtiny202がPA1をHIGH出力する（CART_READY）
4. キーボードがCART_READY HIGHをポーリングまたはHIGHエッジで検出し、ハンドシェイクを開始する
5. ハンドシェイク完了後、ATtiny202がPA1をTCA0（Type A Timer）PWM出力に切り替えてアンサーバックBEEPを吹鳴する（おまけ）

**充電状態監視：**
ハンドシェイク完了後・通常動作中および拡張コネクタ未接続時は、PA1をGPIO入力（内部プルアップ有効）に切り替え、MCP73831のSTATピンを定周期ポーリングで読み取る。CART_READY通知とBEEPはいずれも拡張コネクタ接続時の初期化処理における一時的な役割であり、接続が成立して以降はPA1が空くため、この期間を充電状態監視に活用する。

ATtiny202はPA5 ADCによるバッテリー電圧（残量%）とPA1のSTAT読み取り（充電中/完了）を組み合わせ、UIAPduinoからのReadリクエストに対して充電状態を含む情報を返答する。

**MCP73831 STATピンの状態：**

| STAT電圧 | 意味 |
|---|---|
| LOW | 充電中（CC/CVフェーズ） |
| ハイインピーダンス | 充電完了（内部プルアップでHIGHとして読み取られる） |

#### 2. PA5：アナログ・マルチプレクス（1ピン2役・基板内完結）

100kΩ抵抗2本（精度±1%）によるバッテリー電圧1/2分圧回路（ノイズ除去用0.1µFパスコン並列）の電位点に接続。同ピンに電源タクトスイッチ接続用パッド（SW_PAD）の一方を直結（もう一方はGND）する。

タクトスイッチ本体は基板上に実装しない。カートリッジ筐体の最適な場所に外付けし、基板上のSW_PADへ配線で接続する。基板上にははんだ付けパッド2点（SW_PAD_A・SW_PAD_B）のみを設ける。

- **ボタン非押下時：** バッテリー電圧の半分（約1.5V〜2.1V）をADCで測定し、残量を定周期監視（バッテリー範囲：3.0V〜4.2V、分圧後：1.5V〜2.1V）
- **ボタン押下時：** ピンが強制的にGNDへ落ちるため（0V）、閾値判定で電源ON／長押しシャットダウンイベントを検出
- **トレードオフ（仕様）：** 長押し中（最大3秒）はバッテリー監視停止。シャットダウン実行中のため問題なしとして許容

**バッテリー電圧のみでは充電完了を確実に判定できない理由：**
充電完了時のバッテリー電圧（4.2V）は高残量時の放電中電圧と区別がつかない。MCP73831のSTATピンによる明示的な充電完了通知と組み合わせることで、初めて確実な充電状態判定が可能となる。

#### 3. PA3：MOSFETゲート制御（基板内完結・ハードウェアフェイルセーフ）

P-ch MOSFET（IRLML6402）のゲートへ直結。10kΩのプルアップ抵抗でVBAT_INへ常時ハードウェアプルアップする。UPDI書き換え中にPA3がハイインピーダンスになっても、プルアップ抵抗がゲートをVBATへ引き寄せ、MOSFETを遮断状態（安全側）に固定する。

#### 4. PA2 / PA4：I2Cバス

INT_SDA / INT_SCLとして、対外14ピン拡張コネクタ（UIAPduino・ATtiny1604・CH1115 OLEDと共通バス）に接続する。ATtiny202は**I2Cスレーブ**として動作し、UIAPduinoからのReadリクエストに対してバッテリー残量値・充電状態を返答する。

#### 5. PA0：LOCAL_UPDI_A（エッジパッド経由・UIAPduino PD6へ基板間接続）

A基板のATtiny202のUPDIピン。エッジパッド（Pad 8）経由でUIAPduino PD6と基板間接続する。他デバイスへは露出させない。UIAPduinoからATtiny202ファームウェアの自律書き換え（セルフアップデート）を可能にする。

#### 6. 3.3V降圧Buck DCDC

Buck DCDC IC（RT8008-33GB同等品、SOT-23-5、Vin 2.5V〜5.5V、Vout 3.3V固定、600mA）を採用。リポ放電末期（3.0V入力）でも3.3V出力を維持できるスイッチング方式を選定。LDOは入力電圧が出力電圧を下回るドロップアウトの懸念があるため不採用。

#### 7. 5V昇圧DCDC（MT3608）

MT3608（SOT-23-6、Vin 2V〜24V、Iout最大2A）によるYMF-825音源モジュール駆動用5V昇圧回路。出力電流目安300mA。試作時は9.5mm×27.8mm基板の5V出力パッドから空中配線で外付けモジュールへ接続する割り切りスタイルを採用し、量産時はSMDチップ単体設計を優先する。

#### 8. LiPo充電回路（MCP73831）

UIAPduinoの5VピンはUSB接続時にUSB VBUSの5Vを出力する。この出力をPad 9（USB_5V_IN）経由でA基板上のMCP73831に供給し、LiPoバッテリーへの充電を行う。

**MCP73831を選定した理由：**
LiPo充電ICの主要候補としてTP4056・CN3058E・MCP73831を比較検討した。TP4056はJLCPCB Basicで段取り費$0の優位があるが、状態出力ピンが2本（CHRG・STDBY）であり、ATtiny202の空きピンが1本（PA1）しかないため採用不可。CN3058EもSOT-23-6で小型だが同様に状態ピン2本で不適。MCP73831はSTATピン1本で充電中・充電完了の両状態を区別でき（充電中LOW・完了ハイインピーダンス）、SOT-23-5で最小サイズ、最大500mAで今回の300mA設定に適合する唯一の候補であった。Extended Parts（+$3）となるが設計制約上代替なし。

**充電電流設定の根拠：**
対象バッテリー容量は400〜700mAhを想定。標準充電レートは0.5C（容量の半分の電流）が安全で寿命に優しい。300mAは400mAhに対して0.75C・700mAhに対して0.43Cとなり、両容量帯でほぼ0.5C前後に収まる。フリスクケース（密閉筐体）での発熱抑制の観点からも300mAが最適と判断した。500mAは400mAhバッテリーに対して1.25Cとなり過熱リスクがある。

| 項目 | 値 |
|---|---|
| IC型番 | MCP73831T-2ACI/OT（SOT-23-5・Vcharge=4.2V品） |
| 充電電流 | 300mA |
| PROG抵抗 | R_PROG = 3.3kΩ（1608・±1%）※Icharge = 1000/R_PROG(kΩ)×1000(mA) |
| STAT接続先 | ATtiny202 PA1（GPIO入力・内部プルアップ・通常動作時） |
| VDD接続先 | Pad 9（USB_5V_IN）← UIAPduinoの5Vピン |
| VBAT接続先 | VBAT_IN（LiPoバッテリーライン） |
| USB未接続時 | VDD = 0V → 充電IC自動停止・バッテリーからの逆流なし（内部保護） |

#### 9. ATtiny202ファームウェアの役割（確定版）

ATtiny202のファームウェアは以下の機能を担う。OLEDへの描画処理は一切含まない。

| 機能 | 内容 |
|---|---|
| 電源シーケンス制御 | 電源ボタン押下検出・PA3によるMOSFETオン/オフ |
| ディープスリープ管理 | 電源オフ時のパワーダウンスリープ（数µA）・ボタン押下で覚醒 |
| バッテリー残量監視 | PA5 ADCで定周期測定・内部変数に0〜100%で保持 |
| 充電状態監視 | PA1でMCP73831 STATピンを定周期ポーリング・充電中/完了を内部変数に保持 |
| I2Cスレーブ応答 | UIAPduinoからのReadリクエストにバッテリー残量・充電状態を返答 |
| CART_READY通知 | UIAPduinoからI2C経由で「初期化完了」通知を受信後、PA1をHIGH出力 |
| アンサーバックBEEP | CART_READY HIGH出力後、PA1をTCA0 PWMに切り替えてBEEP（おまけ） |

2KBのFlashは上記機能のみで十分に収まる。

---

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# 📝 Claude Code Instruction: KiCad Schematic Netlist Generation
#    for CAR-01 Power Management Board A (Rev.2.2 Final Confirmed Version)

## 1. Context & Constraint
- Target Board     : A-Board, Intelligent Power Management Module (CAR-01 Standard)
- Dimensions       : 9.5mm x 27.8mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only (single-sided assembly).
                     Bottom side = solid GND copper pour, no routing.
- Manufacturing    : ALL components JLCPCB PCBA (full factory assembly).
                     No hand-solder items.
- Footprint Size   : 1608 (0603 inch) metric for all passive components.
- Edge Pads        : 9x flat edge pads (no castellations) for external connection.

## 2. Component Assignments & RefDes

### JLCPCB PCBA Target (All components)
- U1  : ATtiny202 (SOIC-8)               LCSC: C2052970  Extended +$3
- U2  : IRLML6402 P-ch MOSFET (SOT-23)   LCSC: C5148470  Extended +$3
- U3  : Buck DC-DC 3.3V (SOT-23-5)       LCSC: RT8008-33GB equivalent
         Vin 2.5V~5.5V, Vout 3.3V fixed, 600mA  Extended +$3
- U4  : MT3608 Boost DC-DC (SOT-23-6)    LCSC: C84817    Extended +$3
- U5  : MCP73831T-2ACI/OT LiPo Charger (SOT-23-5)
         Vcharge=4.2V, Imax=500mA, STAT pin (open-drain)  Extended +$3
- R1  : 10kOhm / ±1% (1608)             MOSFET gate pull-up to VBAT_IN
- R2  : 100kOhm / ±1% (1608)            Battery voltage divider upper
- R3  : 100kOhm / ±1% (1608)            Battery voltage divider lower
- R_upper : 750kOhm / ±1% (1608)        MT3608 Feedback upper resistor (Vout=5.0V)
- R_lower : 100kOhm / ±1% (1608)        MT3608 Feedback lower resistor
- R_PROG  : 3.3kOhm / ±1% (1608)        MCP73831 charge current set (300mA)
- C1  : 0.1uF / 10V / X7R (1608)        Divider node noise bypass
- C2  : 10uF / 10V / X5R (1608)         Buck DCDC input bulk cap
- C3  : 10uF / 10V / X5R (1608)         Buck DCDC output bulk cap
- C4  : 0.1uF / 10V / X7R (1608)        Buck DCDC output bypass
- C5  : 10uF / 10V / X5R (1608)         MT3608 input bulk cap
- C6  : 22uF / 10V / X5R (1608)         MT3608 output bulk cap
- C7  : 0.1uF / 10V / X7R (1608)        ATtiny202 VCC bypass
- C8  : 4.7uF / 10V / X5R (1608)        MCP73831 VBAT bypass
- L1  : 4.7uH power inductor (1608~2012, Isat >= 1A)  Buck DCDC inductor
- L2  : 4.7uH power inductor (1608~2012, Isat >= 1A)  MT3608 boost inductor
- D1  : Schottky diode SOD-123 (e.g. B5819W, LCSC C8598 Basic)  MT3608 rectifier
- BZ1 : Piezo buzzer passive terminal points (connected to PA1 / CART_READY line)

### SW1 (Tactile Switch):
- Do NOT place a tactile switch component on the PCB.
- Instead, place 2x through-hole solder pads (SW_PAD_A, SW_PAD_B) on the board edge.
  SW_PAD_A connects to Net DIV_NODE (same as PA5).
  SW_PAD_B connects to GND.
- The actual tactile switch is mounted externally on the cartridge enclosure
  and wired to these pads by the user.

## 3. Netlist & Wiring Specifications

### Power Nets
- VBAT_IN      : Raw LiPo input (3.0V ~ 4.2V)
- GND          : Common ground
- VCC_3V3_OUT  : Switched 3.3V output (post-MOSFET, post-Buck)
- VCC_5V_OUT   : Switched 5V output (post-MOSFET, post-MT3608)
- USB_5V_IN    : USB 5V input from UIAPduino 5V pin (active only when USB connected)

### Always-On Supply (ATtiny202)
- VBAT_IN -> U1 Pin1 (VCC)
- Place C7 (100nF) directly between U1 VCC and GND.
- GND_IN  -> U1 Pin8 (GND)

### MOSFET Gate Control (PA3 / U2)
- U1 PA3 (Pin7) -> U2 Gate (SOT-23 Pin1)
- R1 (10kOhm) between U2 Gate and VBAT_IN.
  (Hardware pull-up: MOSFET OFF = safe state during UPDI programming)
- VBAT_IN -> U2 Source (Pin3)
- U2 Drain (Pin2) -> Net SWITCHED_VBAT

### Buck DC-DC 3.3V (U3)
- SWITCHED_VBAT -> U3 VIN
- Place C2 (10uF) between U3 VIN and GND.
- U3 SW -> L1 -> VCC_3V3_OUT
- Place C3 (10uF) and C4 (100nF) between VCC_3V3_OUT and GND.
- U3 EN -> SWITCHED_VBAT
- VCC_3V3_OUT exposed on Edge Pad 3.

### MT3608 Boost 5V (U4)
- SWITCHED_VBAT -> U4 IN (Pin1)
- Place C5 (10uF) between U4 IN and GND.
- U4 SW (Pin2) -> L2 -> Anode of D1
- D1 Cathode -> VCC_5V_OUT
- Place C6 (22uF) between VCC_5V_OUT and GND.
- Feedback: R_upper (750kOhm) from VCC_5V_OUT to U4 FB pin.
            R_lower (100kOhm) from U4 FB pin to GND.
- U4 EN (Pin3) -> SWITCHED_VBAT
- VCC_5V_OUT exposed on Edge Pad 4.

### MCP73831 LiPo Charger (U5)
- USB_5V_IN (Edge Pad 9) -> U5 VDD (Pin1)
- U5 GND (Pin2) -> GND
- U5 STAT (Pin3) -> U1 PA1 (Pin2)
  (Open-drain output. Internal pull-up on ATtiny202 PA1 active during normal operation.
   LOW = charging, Hi-Z = charge complete)
- U5 VBAT (Pin4) -> Net VBAT_IN (LiPo battery line)
  Place C8 (4.7uF) between VBAT_IN and GND, close to U5 VBAT pin.
- U5 PROG (Pin5) -> R_PROG (3.3kOhm) -> GND
  (Sets charge current: Icharge = 1000V / R_PROG(kOhm) = 303mA ≈ 300mA)

### PA5 Hybrid: Battery Monitor & External Power Button Pads
- VBAT_IN -> R2 (100kOhm) -> Net DIV_NODE -> R3 (100kOhm) -> GND
- Place C1 (100nF) between DIV_NODE and GND.
- DIV_NODE -> U1 PA5 (Pin5)
- SW_PAD_A (through-hole pad) -> DIV_NODE
- SW_PAD_B (through-hole pad) -> GND

### PA1: CART_READY / Answerback BEEP / Charge Status (time-multiplexed 3 roles)
- U1 PA1 (Pin2) -> Net CART_READY
- BZ1 positive terminal -> CART_READY (passive buzzer in parallel)
- BZ1 negative terminal -> GND
- U5 STAT (Pin3) -> CART_READY (MCP73831 STAT open-drain, shares PA1 net)
- CART_READY exposed on Edge Pad 7.
  (Phase 1 - initialization: GPIO OUTPUT HIGH -> CART_READY asserted to keyboard)
  (Phase 2 - post-handshake: TCA0 Type A Timer PWM output for answerback BEEP)
  (Phase 3 - normal operation / no connector: GPIO INPUT with internal pull-up,
             polling MCP73831 STAT: LOW=charging, HIGH=charge complete)

### PA2 / PA4: I2C Bus (INT_SDA / INT_SCL)
- U1 PA2 (Pin3) -> Net INT_SDA -> Edge Pad 5
- U1 PA4 (Pin4) -> Net INT_SCL -> Edge Pad 6
- ATtiny202 operates as I2C SLAVE.
- Responds to UIAPduino Read requests with battery level (0-100%) and charge status.

### PA0: LOCAL_UPDI_A (Edge Pad 8 -> UIAPduino PD6)
- U1 PA0 (Pin6) -> Edge Pad 8 (LOCAL_UPDI_A)
- Edge Pad 8 connects to UIAPduino PD6 via board-to-board wiring.

## 4. Edge Pad Assignment (9 pads, flat, no castellations)

### Grouping
- Power group  : Pad 1 (VBAT_IN), Pad 2 (GND_IN), Pad 3 (VCC_3V3_OUT),
                 Pad 4 (VCC_5V_OUT), Pad 9 (USB_5V_IN)
- I2C group    : Pad 5 (INT_SDA), Pad 6 (INT_SCL)
- Control group: Pad 7 (CART_READY), Pad 8 (LOCAL_UPDI_A)

### Assignment
- Pad 1 : VBAT_IN
- Pad 2 : GND_IN
- Pad 3 : VCC_3V3_OUT
- Pad 4 : VCC_5V_OUT
- Pad 5 : INT_SDA
- Pad 6 : INT_SCL
- Pad 7 : CART_READY  (= CAR-01 expansion connector upper row pin 1)
- Pad 8 : LOCAL_UPDI_A  (= ATtiny202 PA0, connects to UIAPduino PD6)
- Pad 9 : USB_5V_IN  (= UIAPduino 5V pin -> MCP73831 VDD, USB connected only)

## 5. PCB Layout Rules
- ALL components on Top layer only.
- Bottom layer: solid GND copper pour, no signal routing.
- All passive components: 1608 footprint.
- Board dimensions: 9.5mm x 27.8mm.
- U1 (ATtiny202 SOIC-8): place near board center.
- U2 (IRLML6402 SOT-23): place near PA3 net, short gate trace.
- R1 pull-up: place between U2 gate and VBAT_IN, as short as possible.
- U3 Buck IC: place L1 and output caps as close as possible per datasheet layout.
- U4 MT3608: place L2, D1, C6 in tight switching loop per datasheet layout.
- U5 MCP73831: place R_PROG and C8 close to U5. Keep VBAT trace short.
- SW_PAD_A / SW_PAD_B: place at board edge for easy external wiring access.
- Edge pads: group by function along the 27.8mm edge.
- Silkscreen: mark Pin 1 dot on U1/U5, polarity marks on C5/C6/C8,
              label SW_PAD_A/B, label all edge pads with signal names.

Generate complete KiCad schematic netlist and Python build script
based on this finalized architecture.
```

---

## 3. 部品一覧表（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計23点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 | 役割 |
|---|---|---|---|---|---|---|
| U1 | ATtiny202マイコン | SOIC-8 | C2052970 | Extended | 1 | 電源管理コプロセッサ |
| U2 | IRLML6402 P-ch MOSFET | SOT-23 | C5148470 | Extended | 1 | スイッチド電源ライン制御 |
| U3 | Buck DCDC 3.3V固定（RT8008-33GB同等） | SOT-23-5 | 要確認 | Extended | 1 | 3.3V降圧・リポ放電末期も安定 |
| U4 | MT3608 昇圧DCDC | SOT-23-6 | C84817 | Extended | 1 | 5V昇圧・YMF-825駆動用 |
| U5 | MCP73831T-2ACI/OT LiPo充電IC | SOT-23-5 | 要確認 | Extended | 1 | LiPoバッテリー充電・300mA |
| R1 | プルアップ抵抗（10kΩ/±1%） | 1608 | C25804 | Basic | 1 | MOSFETゲートVBATプルアップ |
| R2 | 分圧抵抗・上（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・上側 |
| R3 | 分圧抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・下側 |
| R_upper | MT3608 FB抵抗・上（750kΩ/±1%） | 1608 | 要確認 | Basic | 1 | 5V出力電圧設定 |
| R_lower | MT3608 FB抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | 5V出力電圧設定 |
| R_PROG | MCP73831充電電流設定抵抗（3.3kΩ/±1%） | 1608 | C22978 | Basic | 1 | 充電電流300mA設定 |
| C1 | 分圧ノードパスコン（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ADCノイズ除去 |
| C2 | Buck入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | Buck DCDC入力安定化 |
| C3 | Buck出力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | 3.3V出力平滑 |
| C4 | Buck出力バイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | 3.3V高周波バイパス |
| C5 | MT3608入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | MT3608入力安定化 |
| C6 | MT3608出力バルクコンデンサ（22µF/10V/X5R） | 1608 | 要確認 | Basic | 1 | 5V出力平滑 |
| C7 | ATtiny202 VCCバイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ATtiny202電源デカップリング |
| C8 | MCP73831 VBATバイパスコンデンサ（4.7µF/10V/X5R） | 1608 | 要確認 | Basic | 1 | 充電IC出力安定化 |
| L1 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 | Buck DCDC用インダクタ |
| L2 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 | MT3608昇圧用インダクタ |
| D1 | ショットキーダイオード（B5819W同等） | SOD-123 | C8598 | Basic | 1 | MT3608整流ダイオード |
| BZ1 | 圧電スピーカー接続パッド（パッシブ） | — | — | 別途手配 | 1 | アンサーバックBEEP用端子 |

### 🔴 ユーザー手元での後付け（手はんだ）対象パーツ

| RefDes | 部品名 | 備考 |
|---|---|---|
| SW1 | タクトスイッチ（外付け） | 基板上には実装しない。カートリッジ筐体の最適な場所に取り付け、基板のSW_PAD_A・SW_PAD_Bへ配線する |

---

## 4. 設計根拠メモ

### Extended Parts 段取り費について

本基板はU1・U2・U3・U4・U5の5品種がExtended Parts扱いとなる。段取り費は5種 × $3 = **合計約$15**。5枚製造の場合、1枚あたり$3.00の上乗せとなる。

### Buck DCDC選定根拠（LDOを不採用とした理由）

リポ1セルは放電末期に3.0Vまで低下する。LDOは入力電圧が出力電圧（3.3V）を上回れなくなった瞬間に出力が崩壊し、バッテリー残量が残っているにもかかわらずシステムが停止する。BuckはスイッチングによりVin > Voutのわずかな差でも動作継続できるため、リポを最後まで使い切れる。RT8008-33GBはVin最低2.5Vまで対応しており、リポ3.0V入力でも3.3V出力を維持する。

### MOSFETプルアップによるUPDI書き換え安全保証

UPDI書き換え中（ATtiny202のメモリ消去・再書き込み中）はPA3がハイインピーダンスになる。しかしR1（10kΩ）がゲートをVBAT_INへ物理的に引き寄せ続けるため、P-ch MOSFETはゲート＝ソース＝VBATとなりOFF状態を維持する。書き換え中に下流の電源ラインが乱れることはなく、ATtiny202のファームウェアのみを安全にセルフアップデートできる。

### PA5ハイブリッド判定の閾値設計

バッテリー電圧範囲3.0V〜4.2Vを1/2分圧した場合、PA5電位は1.5V〜2.1Vとなる。閾値を以下のように設定する。
- **1.5V以上：** バッテリー測定モード（通常動作）
- **0.5V以下：** ボタン押下検出（GNDへ強制短絡）
- **0.5V〜1.5V：** 不定帯域（リポ過放電カットオフ領域のため実使用では発生しない）

### PA1時系列多重の設計意図

CART_READY通知とBEEPはいずれも拡張コネクタ接続時の初期化処理における一時的な役割であり、接続が成立した以降・および拡張コネクタに何も接続されていない時間帯にPA1は空く。この空き時間を充電状態監視（MCP73831 STATポーリング）に活用することで、ATtiny202の8ピンを全て使い切りながら充電状態監視を実現した。ATtiny202はピンのモードをソフトウェアで動的に切り替えられるため、フェーズ移行時に出力→入力の切り替えを行う。

### MCP73831充電IC選定根拠

ATtiny202の全8ピンが割り当て済みであり、充電IC状態通知に使えるピンはPA1の1本のみであった。主要候補（TP4056・CN3058E・MCP73831）を比較した結果、STATピン1本で充電中・完了を区別できるMCP73831が唯一の適合品であった。TP4056はJLCPCB Basic（段取り費$0）の優位があるが状態ピン2本必要で不採用。充電電流300mAはバッテリー容量400〜700mAhに対して0.43〜0.75Cとなり、密閉筐体での発熱を抑えつつ安全に充電できる最適値として選定した。

### MT3608 フィードバック抵抗計算（Vout = 5V）

MT3608のフィードバック基準電圧Vref = 0.6V。
Vout = Vref × (1 + R_upper / R_lower)
5.0 = 0.6 × (1 + R_upper / R_lower)
→ R_lower = 100kΩ、R_upper = 750kΩ

### USB充電経路の設計根拠

UIAPduinoの5VピンはUSB接続時にUSB VBUSの5Vを出力するピンであり、外部からの入力は受け付けない出力専用ピンである。USB未接続時は0Vになるため、MCP73831のVDDが0Vとなり充電ICが自動停止し、バッテリーからの逆流も内部保護により防止される。この経路により追加の切り替え回路なしにUSB充電を安全に実現できる。

---

## 変更履歴

| 日付 | バージョン | 内容 |
|---|---|---|
| — | Rev.1.0 | 初版作成（ATtiny202 I2Cマスター・SSD1306 OLED搭載・タクトスイッチ基板実装構成） |
| 2026-06-01 | Rev.2.0 | ATtiny202のI2C役割をマスター→スレーブに変更。UIAPduino（CH32V003）をI2Cマスターに変更。A基板上のSSD1306 OLEDを撤去・D基板へ移管。タクトスイッチを外付けパッド方式に変更。MT3608 FB抵抗をR_upper 470kΩ→750kΩに修正 |
| 2026-06-10 | Rev.2.1 | 基板サイズを10mm×28mm→9.5mm×27.8mmに修正。PA1信号名をCART_READYに修正・CAR-01規格拡張コネクタ上列1と同一であることを明記。PA1動作シーケンスをGPIO LOW出力削除・UIAPduino I2C通知→HIGH出力に修正。PA0をエッジパッド経由UIAPduino基板間接続に変更。エッジパッド7→8個に変更 |
| 2026-06-17 | Rev.2.2 | LiPo充電回路（MCP73831T-2ACI/OT）をA基板に追加。充電電流300mA（R_PROG=3.3kΩ）。PA1に充電状態監視（MCP73831 STATポーリング）を追加（通常動作時の3役目）。エッジパッド8→9個に変更（Pad 9: USB_5V_IN追加）。BOM・Claude Code指示書・設計根拠メモを全更新。充電IC選定比較（TP4056・CN3058E・MCP73831）を設計根拠に追記 |

---

*文書バージョン：Rev.2.2確定版 / 本ファイルをそのままClaude Codeへ投入してください*
