# CAR-01規格 A基板（インテリジェント電源管理モジュール）設計仕様書

## Rev.2.0 — 2026年6月1日
## ※Rev.1.0からの変更箇所は【変更】タグで明示する。

---

## 1. 実装目論見書

### プロジェクト概要

本ボード（A基板）は、ラテカピュータ（CAR-01規格）に準拠した自律型電源管理モジュール基板である。ATtiny202をコプロセッサとして搭載し、メインMCU（UIAPduino等）のSRAM・フラッシュ・CPU時間を一切消費せずに、バッテリー残量監視・電源シーケンス制御・接続通知BEEPを現地完結させる。10mm × 28mmの片面実装スティック形状に全回路を収める。

### 物理仕様・製造要件

| 項目 | 仕様 |
|---|---|
| 基板サイズ | 10mm × 28mm（縦28mmはB基板・C基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | Top面（表面）のみへの片面実装。裏面は全面ベタGNDプレーン |
| 基板色 | Green または Black（最安・最短納期を優先） |
| 発注枚数 | 5枚（JLCPCBの最小ロット） |
| 外部接続端子 | 基板端エッジ寄りベタパッド × 7個（カスタレーションなし） |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** ATtiny202・IRLML6402・Buck DCDC IC・MT3608・受動部品（1608サイズ）の全品
- **手はんだ：** なし（全点工場実装）

### システムブロックおよび電気的トポロジー

#### 電源アーキテクチャ

本基板はリポ1セル（3.0V〜4.2V）からの生電圧（VBAT_IN）を主入力とする二系統の電源ラインを持つ。

**常時通電ライン（VBAT_IN）：** リポからATtiny202の1番ピン（VCC）へ直結。電源OFF時はATtiny202がパワーダウンスリープ（数µA）で待機し、バッテリー消費を最小化する。

**スイッチド下流ライン（VCC_3V3_OUT / VCC_5V_OUT）：** ATtiny202の7番ピン（PA3）がLOWを出力するとP-ch MOSFET（IRLML6402）が導通し、下流のDCDC回路（3.3V降圧BuckおよびMT3608による5V昇圧）へ電流が流れ、UIAPduinoおよびYMF-825を同時駆動する。

#### I2Cバストポロジー【変更】

ATtiny202を**I2Cスレーブ**、UIAPduino（CH32V003）を**I2Cマスター**として定義する。ATtiny202はUIAPduinoからのReadリクエストに対してバッテリー残量値（1バイト、0〜100%）を返答するのみである。OLEDへの描画・UIAPduinoへの自律送信は一切行わない。

OLEDはA基板から撤去し、D基板（ATtiny1604搭載）に移管した。OLEDへの描画はATtiny1604が単独で担当する（詳細はD基板目論見書参照）。

**変更理由：**
- ATtiny202のFlash（2KB）にOLED描画ライブラリを収容することが困難であった
- ATtiny202の全ピンが既割り当てであり、追加の通信線を引き出せない
- OLEDはUIAPduinoと同一電源（VCC_3V3_OUT）のため、ATtiny202が単独でOLEDを使用できる場面が存在しない
- UIAPduinoをマスターとするシングルマスター構成の方が設計がシンプルで堅牢である

### ATtiny202 ピンアサイン（厳格定義）

```
              ATtiny202 (SOIC-8)
           ┌───────┴───────┐
  VBAT_IN ─┤ 1. VCC   GND 8 ├─── GND_IN
LtcBus_SYS─┤ 2. PA1   PA3 7 ├─── MOSFETゲート制御（10kΩでVBATへPU）【基板内完結】
 _INT/SPK  │               │
  INT_SDA ─┤ 3. PA2   PA0 6 ├─── LOCAL_UPDI（自基板内UIAPduino専用）【基板内完結】
  INT_SCL ─┤ 4. PA4   PA5 5 ├─── タクトSW接続パッド ＆ バッテリー分圧監視回路【基板内完結】
           └───────────────┘
```

### 回路設計コアアーキテクチャ

#### 1. PA1：時系列マルチプレクス（1ピン2役）

基板外の拡張コネクタ線LtcBus_SYS_INTおよび圧電スピーカーのプラス側を並列に接続する。

- **初期動作（物理合体時）：** GPIO出力LOWを数ミリ秒維持し、ホスト側へ挿入を物理通知する
- **ネゴシエーション完了後：** TCAタイマー駆動のPWM出力モードへ動的切り替え。SYS_INT配線をスピーカー駆動線として流用し、接続成功のアンサーバックBEEPを吹鳴する。ホスト側はこのピンの割り込みをマスク済みのため安全

ホスト側は初期ネゴシエーション完了の次の瞬間にSYS_INT割り込みレジスタを完全無効化することを規格として義務付ける。ホストが予期せぬリセットを起こした場合は再ネゴシエーションシーケンスへ移行し、完了フラグが立つまでSYS_INTピンを見ない構造とする。

#### 2. PA5：アナログ・マルチプレクス（1ピン2役・基板内完結）【変更】

100kΩ抵抗2本（精度±1%）によるバッテリー電圧1/2分圧回路（ノイズ除去用0.1µFパスコン並列）の電位点に接続。同ピンに電源タクトスイッチ接続用パッド（SW_PAD）の一方を直結（もう一方はGND）する。

タクトスイッチ本体は基板上に実装しない。カートリッジ筐体の最適な場所に外付けし、基板上のSW_PADへ配線で接続する。基板上にははんだ付けパッド2点（SW_PAD_A・SW_PAD_B）のみを設ける。

- **ボタン非押下時：** バッテリー電圧の半分（約1.5V〜2.1V）をADCで測定し、残量を定周期監視（バッテリー範囲：3.0V〜4.2V、分圧後：1.5V〜2.1V）
- **ボタン押下時：** ピンが強制的にGNDへ落ちるため（0V）、閾値判定で電源ON／長押しシャットダウンイベントを検出
- **トレードオフ（仕様）：** 長押し中（最大3秒）はバッテリー監視停止。シャットダウン実行中のため問題なしとして許容

#### 3. PA3：MOSFETゲート制御（基板内完結・ハードウェアフェイルセーフ）

P-ch MOSFET（IRLML6402）のゲートへ直結。10kΩのプルアップ抵抗でVBAT_INへ常時ハードウェアプルアップする。UPDI書き換え中にPA3がハイインピーダンスになっても、プルアップ抵抗がゲートをVBATへ引き寄せ、MOSFETを遮断状態（安全側）に固定する。

#### 4. PA2 / PA4：I2Cバス【変更】

INT_SDA / INT_SCLとして、対外14ピン拡張コネクタ（UIAPduino・ATtiny1604・CH1115 OLEDと共通バス）に接続する。ATtiny202は**I2Cスレーブ**として動作し、UIAPduinoからのReadリクエストに対してバッテリー残量値（1バイト）を返答するのみである。

A基板上にOLEDは搭載しない（Rev.1.0から変更）。旧Rev.1.0で記載されていたSSD1306（0.42インチ・72×40ドット）はD基板のCH1115（0.5インチ・48×88ドット）に置き換えられた。

#### 5. PA0：LOCAL_UPDI（基板内完結）

自基板内のUIAPduino専用。他デバイスへは一切露出させない。UIAPduinoからATtiny202ファームウェアの自律書き換え（セルフアップデート）を可能にする。

#### 6. 3.3V降圧Buck DCDC

Buck DCDC IC（RT8008-33GB同等品、SOT-23-5、Vin 2.5V〜5.5V、Vout 3.3V固定、600mA）を採用。リポ放電末期（3.0V入力）でも3.3V出力を維持できるスイッチング方式を選定。LDOは入力電圧が出力電圧を下回るドロップアウトの懸念があるため不採用。

#### 7. 5V昇圧DCDC（MT3608）

MT3608（SOT-23-6、Vin 2V〜24V、Iout最大2A）によるYMF-825音源モジュール駆動用5V昇圧回路。出力電流目安300mA。試作時は10mm×28mm基板の5V出力パッドから空中配線で外付けモジュールへ接続する割り切りスタイルを採用し、量産時はSMDチップ単体設計を優先する。

#### 8. ATtiny202ファームウェアの役割（Rev.2.0確定版）【変更】

ATtiny202のファームウェアは以下の機能のみを担う。OLEDへの描画処理は一切含まない。

| 機能 | 内容 |
|---|---|
| 電源シーケンス制御 | 電源ボタン押下検出・PA3によるMOSFETオン/オフ |
| ディープスリープ管理 | 電源オフ時のパワーダウンスリープ（数µA）・ボタン押下で覚醒 |
| バッテリー残量監視 | PA5 ADCで定周期測定・内部変数に0〜100%で保持 |
| I2Cスレーブ応答 | UIAPduinoからのReadリクエストに残量値1バイトを返答 |
| 接続通知BEEP | PA1 PWMによるアンサーバックBEEP吹鳴 |

2KBのFlashは上記機能のみで十分に収まる。

---

## 2. Claude Code用 回路図自動生成・設計指示書

```text
# 📝 Claude Code Instruction: KiCad Schematic Netlist Generation
#    for CAR-01 Power Management Board A (Rev.2.0 Final Confirmed Version)

## 1. Context & Constraint
- Target Board     : A-Board, Intelligent Power Management Module (CAR-01 Standard)
- Dimensions       : 10mm x 28mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only (single-sided assembly).
                     Bottom side = solid GND copper pour, no routing.
- Manufacturing    : ALL components JLCPCB PCBA (full factory assembly).
                     No hand-solder items.
- Footprint Size   : 1608 (0603 inch) metric for all passive components.
- Edge Pads        : 7x flat edge pads (no castellations) for external connection.

## 2. Component Assignments & RefDes

### JLCPCB PCBA Target (All components)
- U1  : ATtiny202 (SOIC-8)             LCSC: C2052970  Extended +$3
- U2  : IRLML6402 P-ch MOSFET (SOT-23) LCSC: C5148470  Extended +$3
- U3  : Buck DC-DC 3.3V (SOT-23-5)     LCSC: RT8008-33GB equivalent
         Vin 2.5V~5.5V, Vout 3.3V fixed, 600mA  Extended +$3
- U4  : MT3608 Boost DC-DC (SOT-23-6)  LCSC: C84817    Extended +$3
- R1  : 10kOhm / ±1% (1608)           MOSFET gate pull-up to VBAT_IN
- R2  : 100kOhm / ±1% (1608)          Battery voltage divider upper
- R3  : 100kOhm / ±1% (1608)          Battery voltage divider lower
- R_upper : 750kOhm / ±1% (1608)      MT3608 Feedback upper resistor (Vout=5.0V)
- R_lower : 100kOhm / ±1% (1608)      MT3608 Feedback lower resistor
- C1  : 0.1uF / 10V / X7R (1608)      Divider node noise bypass
- C2  : 10uF / 10V / X5R (1608)       Buck DCDC input bulk cap
- C3  : 10uF / 10V / X5R (1608)       Buck DCDC output bulk cap
- C4  : 0.1uF / 10V / X7R (1608)      Buck DCDC output bypass
- C5  : 10uF / 10V / X5R (1608)       MT3608 input bulk cap
- C6  : 22uF / 10V / X5R (1608)       MT3608 output bulk cap
- C7  : 0.1uF / 10V / X7R (1608)      ATtiny202 VCC bypass
- L1  : 4.7uH power inductor (1608~2012, Isat >= 1A)  Buck DCDC inductor
- L2  : 4.7uH power inductor (1608~2012, Isat >= 1A)  MT3608 boost inductor
- D1  : Schottky diode SOD-123 (e.g. B5819W, LCSC C8598 Basic)  MT3608 rectifier
- BZ1 : Piezo buzzer passive terminal points (connected to PA1 / LtcBus_SYS_INT line)

### [CHANGED Rev.2.0] SW1 (Tactile Switch):
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
  (SWITCHED_VBAT feeds both U3 input and U4 input)

### Buck DC-DC 3.3V (U3)
- SWITCHED_VBAT -> U3 VIN
- Place C2 (10uF) between U3 VIN and GND. (input bulk)
- U3 SW -> L1 -> VCC_3V3_OUT
- Place C3 (10uF) and C4 (100nF) between VCC_3V3_OUT and GND. (output)
- U3 EN -> SWITCHED_VBAT (enable always-on when MOSFET conducts)
- U3 GND -> GND
- VCC_3V3_OUT exposed on Edge Pad 3.

### MT3608 Boost 5V (U4)
- SWITCHED_VBAT -> U4 IN (Pin1)
- Place C5 (10uF) between U4 IN and GND. (input bulk)
- U4 SW (Pin2) -> L2 -> Anode of D1
- D1 Cathode -> VCC_5V_OUT
- Place C6 (22uF) between VCC_5V_OUT and GND. (output bulk)
- Feedback: R_upper (750kOhm) from VCC_5V_OUT to U4 FB pin.
            R_lower (100kOhm) from U4 FB pin to GND.
- U4 EN (Pin3) -> SWITCHED_VBAT (always enabled when MOSFET conducts)
- VCC_5V_OUT exposed on Edge Pad 4.

### PA5 Hybrid: Battery Monitor & External Power Button Pads [CHANGED Rev.2.0]
- VBAT_IN -> R2 (100kOhm) -> Net DIV_NODE -> R3 (100kOhm) -> GND
- Place C1 (100nF) between DIV_NODE and GND. (noise bypass)
- DIV_NODE -> U1 PA5 (Pin5)
- SW_PAD_A (through-hole pad) -> DIV_NODE
- SW_PAD_B (through-hole pad) -> GND
  (External tactile switch wired between SW_PAD_A and SW_PAD_B by user.
   Button press forces PA5 to 0V. ADC threshold:
   >1.5V = battery measurement mode,
   <0.5V = button press detected)

### PA1: SYS_INT / Speaker Multiplex
- U1 PA1 (Pin2) -> Net LtcBus_SYS_INT_SPK
- BZ1 positive terminal -> LtcBus_SYS_INT_SPK
- BZ1 negative terminal -> GND
- LtcBus_SYS_INT_SPK exposed on Edge Pad 7.
  (Initial: GPIO LOW output for host notification.
   Post-negotiation: TCA timer PWM output for answerback BEEP.)

### PA2 / PA4: I2C Bus (INT_SDA / INT_SCL) [CHANGED Rev.2.0]
- U1 PA2 (Pin3) -> Net INT_SDA
- U1 PA4 (Pin4) -> Net INT_SCL
- INT_SDA and INT_SCL exposed on Edge Pads 5 and 6.
- ATtiny202 operates as I2C SLAVE (address TBD).
- UIAPduino (CH32V003) operates as I2C Master.
- ATtiny202 responds to UIAPduino Read requests with battery level (1 byte, 0-100%).
- No on-board OLED. OLED relocated to D-board (CH1115, 48x88, 0x3C).

### PA0: LOCAL_UPDI (Internal only)
- U1 PA0 (Pin6) -> UPDI pad (internal, connected to UIAPduino UPDI pin only)
- NOT exposed to any external pad.

## 4. Edge Pad Assignment (7 pads, flat, no castellations)
- Pad 1 : VBAT_IN
- Pad 2 : GND_IN
- Pad 3 : VCC_3V3_OUT
- Pad 4 : VCC_5V_OUT
- Pad 5 : INT_SDA
- Pad 6 : INT_SCL
- Pad 7 : LtcBus_SYS_INT_SPK

## 5. PCB Layout Rules
- ALL components on Top layer only.
- Bottom layer: solid GND copper pour, no signal routing.
- All passive components: 1608 footprint.
- U1 (ATtiny202 SOIC-8): place near board center-top.
- U2 (IRLML6402 SOT-23): place near PA3 net, short gate trace.
- R1 pull-up: place between U2 gate and VBAT_IN, as short as possible.
- U3 Buck IC: place L1 and output caps as close as possible per datasheet layout.
- U4 MT3608: place L2, D1, C6 in tight switching loop per datasheet layout.
- SW_PAD_A / SW_PAD_B: place at board edge for easy external wiring access.
- Edge pads: distribute along the 28mm edge of the board.
- Silkscreen: mark Pin 1 dot on U1, polarity marks on C5/C6, label SW_PAD_A/B.

Generate complete KiCad schematic netlist and Python build script
based on this finalized architecture.
```

---

## 3. 部品一覧表（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計20点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 | 役割 |
|---|---|---|---|---|---|---|
| U1 | ATtiny202マイコン | SOIC-8 | C2052970 | Extended | 1 | 電源管理コプロセッサ |
| U2 | IRLML6402 P-ch MOSFET | SOT-23 | C5148470 | Extended | 1 | スイッチド電源ライン制御 |
| U3 | Buck DCDC 3.3V固定（RT8008-33GB同等） | SOT-23-5 | 要確認 | Extended | 1 | 3.3V降圧・リポ放電末期も安定 |
| U4 | MT3608 昇圧DCDC | SOT-23-6 | C84817 | Extended | 1 | 5V昇圧・YMF-825駆動用 |
| R1 | プルアップ抵抗（10kΩ/±1%） | 1608 | C25804 | Basic | 1 | MOSFETゲートVBATプルアップ |
| R2 | 分圧抵抗・上（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・上側 |
| R3 | 分圧抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | バッテリー電圧1/2分圧・下側 |
| R_upper | MT3608 FB抵抗・上（750kΩ/±1%） | 1608 | 要確認 | Basic | 1 | 5V出力電圧設定 |
| R_lower | MT3608 FB抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 | 5V出力電圧設定 |
| C1 | 分圧ノードパスコン（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ADCノイズ除去 |
| C2 | Buck入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | Buck DCDC入力安定化 |
| C3 | Buck出力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | 3.3V出力平滑 |
| C4 | Buck出力バイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | 3.3V高周波バイパス |
| C5 | MT3608入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 | MT3608入力安定化 |
| C6 | MT3608出力バルクコンデンサ（22µF/10V/X5R） | 1608 | 要確認 | Basic | 1 | 5V出力平滑 |
| C7 | ATtiny202 VCCバイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 | ATtiny202電源デカップリング |
| L1 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 | Buck DCDC用インダクタ |
| L2 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 | MT3608昇圧用インダクタ |
| D1 | ショットキーダイオード（B5819W同等） | SOD-123 | C8598 | Basic | 1 | MT3608整流ダイオード |
| BZ1 | 圧電スピーカー接続パッド（パッシブ） | — | — | 別途手配 | 1 | アンサーバックBEEP用端子 |

### 🔴 ユーザー手元での後付け（手はんだ）対象パーツ【変更】

| RefDes | 部品名 | 備考 |
|---|---|---|
| SW1 | タクトスイッチ（外付け） | 基板上には実装しない。カートリッジ筐体の最適な場所に取り付け、基板のSW_PAD_A・SW_PAD_Bへ配線する |

---

## 4. 設計根拠メモ（Claude Code補足情報）

### Extended Parts 段取り費について

本基板はU1・U2・U3・U4の4品種がExtended Parts扱いとなる。段取り費は4種 × $3 = **合計約$12**。5枚製造の場合、1枚あたり$2.40の上乗せとなる。手はんだ作業コストとSOIC-8のブリッジリスクを排除できる工場実装品質の対価として許容済み。

### Buck DCDC選定根拠（LDOを不採用とした理由）

リポ1セルは放電末期に3.0Vまで低下する。LDOは入力電圧が出力電圧（3.3V）を上回れなくなった瞬間に出力が崩壊し、バッテリー残量が残っているにもかかわらずシステムが停止する。BuckはスイッチングによりVin > Voutのわずかな差でも動作継続できるため、リポを最後まで使い切れる。RT8008-33GBはVin最低2.5Vまで対応しており、リポ3.0V入力でも3.3V出力を維持する。

### MOSFETプルアップによるUPDI書き換え安全保証

UPDI書き換え中（ATtiny202のメモリ消去・再書き込み中）はPA3がハイインピーダンスになる。しかしR1（10kΩ）がゲートをVBAT_INへ物理的に引き寄せ続けるため、P-ch MOSFETはゲート＝ソース＝VBATとなりOFF状態を維持する。書き換え中に下流の電源ラインが乱れることはなく、ATtiny202のファームウェアのみを安全にセルフアップデートできる。

### PA5ハイブリッド判定の閾値設計

バッテリー電圧範囲3.0V〜4.2Vを1/2分圧した場合、PA5電位は1.5V〜2.1Vとなる。閾値を以下のように設定する。
- **1.5V以上：** バッテリー測定モード（通常動作）
- **0.5V以下：** ボタン押下検出（GNDへ強制短絡）
- **0.5V〜1.5V：** 不定帯域（リポ過放電カットオフ領域のため実使用では発生しない）

### PA1 PWM切り替えシーケンス

合体通知（GPIO LOW）からBEEP吹鳴（TCA PWM）への切り替えは、ホスト側からのI2C経由ネゴシエーション完了通知を受信した直後に実行する。切り替えタイミングでホスト側はSYS_INT割り込みをすでにマスクしているため、PWMパルスが誤割り込みとして扱われることはない。

### ATtiny202 I2Cスレーブ動作について【変更】

ATtiny202はI2Cスレーブとして動作し、UIAPduinoからのReadリクエストに対してバッテリー残量値（1バイト）を返答する。マスター送信処理は不要となり、128B SRAMおよび2KB Flashの消費をさらに抑制できる。旧Rev.1.0で採用していたI2Cマスター構成（ATtiny202がOLEDへ自律描画・UIAPduinoへ自律送信）は廃止した。

### MT3608 フィードバック抵抗計算（Vout = 5V）

MT3608のフィードバック基準電圧Vref = 0.6V。
Vout = Vref × (1 + R_upper / R_lower)
5.0 = 0.6 × (1 + R_upper / R_lower)
R_upper / R_lower = 7.33
→ R_lower = 100kΩ、R_upper = 750kΩ（標準値: 680kΩまたは750kΩ）
※Claude Codeは上記計算に基づき最寄りのE96標準値を選定すること。

### タクトスイッチの外付け方針【変更】

タクトスイッチ本体は基板上に実装しない。カートリッジ筐体の設計に合わせてユーザーが最適な場所に取り付け、基板上のSW_PAD_A・SW_PAD_Bへ配線する。基板上にははんだ付け用スルーホールパッド2点のみを設ける。これによりカートリッジ筐体の設計自由度を確保し、基板高さへの影響もない。

---

## 変更履歴

| 日付 | バージョン | 内容 |
|---|---|---|
| — | Rev.1.0 | 初版作成（ATtiny202 I2Cマスター・SSD1306 OLED搭載・タクトスイッチ基板実装構成） |
| 2026-06-01 | Rev.2.0 | ATtiny202のI2C役割をマスター→スレーブに変更 |
| 2026-06-01 | Rev.2.0 | UIAPduino（CH32V003）をI2Cマスターに変更 |
| 2026-06-01 | Rev.2.0 | A基板上のSSD1306 OLEDを撤去。OLEDはD基板（CH1115・0.5インチ・48×88）へ移管 |
| 2026-06-01 | Rev.2.0 | ATtiny202ファームウェア役割をOLED描画なし・バッテリー残量スレーブ応答のみに変更 |
| 2026-06-01 | Rev.2.0 | タクトスイッチ（SW1）を基板実装から外付けパッド方式に変更 |
| 2026-06-01 | Rev.2.0 | MT3608 FB抵抗をR_upper 470kΩ→750kΩに修正 |
| 2026-06-01 | Rev.2.0 | Claude Code指示書のI2C・SW1・FB抵抗記述を全更新 |

---

*文書バージョン：Rev.2.0確定版 / 本ファイルをそのままClaude Codeへ投入してください*
