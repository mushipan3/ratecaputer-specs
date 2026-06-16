# CAR-01規格 A基板（インテリジェント電源管理モジュール）設計仕様書

**Rev. 2.4 — 2026年6月12日**

---

## 目次

1. [本書の位置づけ](#1-本書の位置づけ)
2. [概要・目的](#2-概要目的)
3. [物理仕様](#3-物理仕様)
4. [回路設計](#4-回路設計)
   - 4.1 電源アーキテクチャ
   - 4.2 I2Cバストポロジー
   - 4.3 ATtiny202 ピンアサイン
   - 4.4 回路設計コアアーキテクチャ
5. [部品一覧（BOM）](#5-部品一覧bom)
6. [外部接続・インターフェース](#6-外部接続インターフェース)
7. [Claude Code用 回路図自動生成指示書](#7-claude-code用-回路図自動生成指示書)
8. [設計根拠メモ](#8-設計根拠メモ)
9. [未決事項（PENDING）](#9-未決事項pending)
10. [変更履歴](#10-変更履歴)

---

## 1. 本書の位置づけ

本書はラテカピュータ（CAR-01規格）のA基板（インテリジェント電源管理モジュール）の設計仕様を定める。

**参照ドキュメント：**

| ドキュメント | 参照事項 |
|---|---|
| cartridge_master_20260612.md | CAR-01全体規格・電源方針・拡張コネクタ仕様 |
| boards_B_board_hardware_20260612.md | AMP_5.0V供給先・電源シーケンス連携 |
| boards_D_board_firmware_20260612.md | ATtiny1604 LOCAL_UPDI_D・I2Cバス共有 |
| features_system_menu_20260612.md | CART_READY通知シーケンス |

---

## 2. 概要・目的

本ボード（A基板）は、ラテカピュータ（CAR-01規格）に準拠した自律型電源管理モジュール基板である。ATtiny202をコプロセッサとして搭載し、メインMCU（UIAPduino等）のSRAM・フラッシュ・CPU時間を一切消費せずに、バッテリー残量監視・電源シーケンス制御・接続通知BEEPを現地完結させる。9.5mm × 27.8mmの片面実装スティック形状に全回路を収める。

---

## 3. 物理仕様

| 項目 | 仕様 |
|---|---|
| 基板サイズ | 9.5mm × 27.8mm（縦27.8mmはB基板・C基板とパネライズグリッドを統一） |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 実装方式 | Top面（表面）のみへの片面実装。裏面は全面ベタGNDプレーン |
| 基板色 | Green または Black（最安・最短納期を優先） |
| 発注枚数 | 5枚（JLCPCBの最小ロット） |
| 外部接続端子 | 基板端エッジ寄りベタパッド × 8個（カスタレーションなし） |

#### 製造委託範囲

- **JLCPCB PCBA 自動実装：** ATtiny202・IRLML6402・Buck DCDC IC・MT3608・受動部品（1608サイズ）の全品
- **手はんだ：** なし（全点工場実装）

---

## 4. 回路設計

### 4.1 電源アーキテクチャ

本基板はリポ1セル（3.0V〜4.2V）からの生電圧（VBAT_IN）を主入力とする二系統の電源ラインを持つ。

**常時通電ライン（VBAT_IN）：** リポからATtiny202の1番ピン（VCC）へ直結。電源OFF時はATtiny202がパワーダウンスリープ（数µA）で待機し、バッテリー消費を最小化する。

**スイッチド下流ライン（VCC_3V3_OUT / VCC_5V_OUT）：** ATtiny202の7番ピン（PA3）がLOWを出力するとP-ch MOSFET（IRLML6402）が導通し、下流のDCDC回路（3.3V降圧BuckおよびMT3608による5V昇圧）へ電流が流れ、UIAPduinoおよびYMF-825を同時駆動する。

### 4.2 I2Cバストポロジー

ATtiny202を**I2Cスレーブ**、UIAPduino（CH32V003）を**I2Cマスター**として定義する。ATtiny202はUIAPduinoからのReadリクエストに対してバッテリー残量値（1バイト、0〜100%）を返答するのみである。OLEDへの描画・UIAPduinoへの自律送信は一切行わない。

OLEDはA基板から撤去し、D基板（ATtiny1604搭載）に移管した。OLEDへの描画はATtiny1604が単独で担当する（詳細はD基板目論見書参照）。

### 4.3 ATtiny202 ピンアサイン

```
              ATtiny202 (SOIC-8)
           ┌───────┴───────┐
  VBAT_IN ─┤ 1. VCC   GND 8 ├─── GND_IN
CART_READY─┤ 2. PA1   PA3 7 ├─── MOSFETゲート制御（10kΩでVBATへPU）【基板内完結】
  /BEEP    │               │
  INT_SDA ─┤ 3. PA2   PA0 6 ├─── LOCAL_UPDI_A（エッジパッド経由UIAPduino PD6へ）
  INT_SCL ─┤ 4. PA4   PA5 5 ├─── タクトSW接続パッド ＆ バッテリー分圧監視回路【基板内完結】
           └───────────────┘
```

### 4.4 回路設計コアアーキテクチャ

#### PA1：CART_READY兼アンサーバックBEEP（1ピン2役）

拡張コネクタ上列1（CART_READY）に直結し、圧電スピーカーのプラス側を並列に接続する。

PA1はCAR-01規格の**CART_READY信号**（拡張コネクタ上列1）と同一の信号線である。CART_READY出力のトリガーはATtiny202の自律判断ではなく、UIAPduinoからのI2C通知である。

**正しい動作シーケンス：**

1. UIAPduinoが共通プログラム初期化を完了する
2. UIAPduinoがI2C経由でATtiny202に「初期化完了」を通知する
3. ATtiny202がPA1をHIGH出力する（CART_READY）
4. キーボードがCART_READY HIGHをポーリングまたはHIGHエッジで検出し、ハンドシェイクを開始する
5. ハンドシェイク完了後、ATtiny202がPA1をTCA0（Type A Timer）PWM出力に切り替えてアンサーバックBEEPを吹鳴する（おまけ）

#### PA5：アナログ・マルチプレクス（1ピン2役・基板内完結）

100kΩ抵抗2本（精度±1%）によるバッテリー電圧1/2分圧回路（ノイズ除去用0.1µFパスコン並列）の電位点に接続。同ピンに電源タクトスイッチ接続用パッド（SW_PAD）の一方を直結（もう一方はGND）する。

タクトスイッチ本体は基板上に実装しない。カートリッジ筐体の最適な場所に外付けし、基板上のSW_PADへ配線で接続する。基板上にははんだ付けパッド2点（SW_PAD_A・SW_PAD_B）のみを設ける。

- **ボタン非押下時：** バッテリー電圧の半分（約1.5V〜2.1V）をADCで測定し、残量を定周期監視（バッテリー範囲：3.0V〜4.2V、分圧後：1.5V〜2.1V）
- **ボタン押下時：** ピンが強制的にGNDへ落ちるため（0V）、閾値判定で電源ON／長押しシャットダウンイベントを検出
- **トレードオフ（仕様）：** 長押し中（最大3秒）はバッテリー監視停止。シャットダウン実行中のため問題なしとして許容

#### PA3：MOSFETゲート制御（基板内完結・ハードウェアフェイルセーフ）

P-ch MOSFET（IRLML6402）のゲートへ直結。10kΩのプルアップ抵抗でVBAT_INへ常時ハードウェアプルアップする。UPDI書き換え中にPA3がハイインピーダンスになっても、プルアップ抵抗がゲートをVBATへ引き寄せ、MOSFETを遮断状態（安全側）に固定する。

#### PA2 / PA4：I2Cバス

INT_SDA / INT_SCLとして、対外14ピン拡張コネクタ（UIAPduino・ATtiny1604・CH1115 OLEDと共通バス）に接続する。ATtiny202は**I2Cスレーブ**として動作し、UIAPduinoからのReadリクエストに対してバッテリー残量値（1バイト）を返答するのみである。

A基板上にOLEDは搭載しない。OLEDはD基板（CH1115・0.5インチ・48×88ドット）に移管済み。

#### PA0：LOCAL_UPDI_A（エッジパッド経由・UIAPduino PD6へ基板間接続）

A基板のATtiny202のUPDIピン。エッジパッド（Pad 8）経由でUIAPduino PD6と基板間接続する。他デバイスへは露出させない。UIAPduinoからATtiny202ファームウェアの自律書き換え（セルフアップデート）を可能にする。

**兼用制約：** UIAPduino PD6はYMF825のIRQ_Nとしても使用する。ATtiny202アップデート中はFM音源を停止すること。TFT版では拡張デバイスのCSをネゲートすること。

なお、UIAPduino PD0はATtiny1604（D基板）のLOCAL_UPDI_Dとして使用する。PD6（LOCAL_UPDI_A）とPD0（LOCAL_UPDI_D）は独立した別線であり、UPDIは1対1接続のため同時アップデートはできない。

#### 3.3V降圧Buck DCDC

Buck DCDC IC（RT8008-33GB同等品、SOT-23-5、Vin 2.5V〜5.5V、Vout 3.3V固定、600mA）を採用。リポ放電末期（3.0V入力）でも3.3V出力を維持できるスイッチング方式を選定。LDOは入力電圧が出力電圧を下回るドロップアウトの懸念があるため不採用。

#### 5V昇圧DCDC（MT3608）

MT3608（SOT-23-6、Vin 2V〜24V、Iout最大2A）によるYMF-825音源モジュール駆動用5V昇圧回路。出力電流目安300mA。試作時は9.5mm×27.8mm基板の5V出力パッドから空中配線で外付けモジュールへ接続する割り切りスタイルを採用し、量産時はSMDチップ単体設計を優先する。

---

## 5. 部品一覧（BOM）

### 🟢 自動実装（JLCPCB PCBA委託）対象パーツ　計20点

| RefDes | 部品名・仕様 | パッケージ | LCSC番号 | 区分 | 数量 |
|---|---|---|---|---|---|
| U1 | ATtiny202マイコン | SOIC-8 | C2052970 | Extended | 1 |
| U2 | IRLML6402 P-ch MOSFET | SOT-23 | C5148470 | Extended | 1 |
| U3 | Buck DCDC 3.3V固定（RT8008-33GB同等） | SOT-23-5 | 要確認 | Extended | 1 |
| U4 | MT3608 昇圧DCDC | SOT-23-6 | C84817 | Extended | 1 |
| R1 | プルアップ抵抗（10kΩ/±1%） | 1608 | C25804 | Basic | 1 |
| R2 | 分圧抵抗・上（100kΩ/±1%） | 1608 | C25803 | Basic | 1 |
| R3 | 分圧抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 |
| R_upper | MT3608 FB抵抗・上（750kΩ/±1%） | 1608 | 要確認 | Basic | 1 |
| R_lower | MT3608 FB抵抗・下（100kΩ/±1%） | 1608 | C25803 | Basic | 1 |
| C1 | 分圧ノードパスコン（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 |
| C2 | Buck入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 |
| C3 | Buck出力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 |
| C4 | Buck出力バイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 |
| C5 | MT3608入力バルクコンデンサ（10µF/10V/X5R） | 1608 | C19702 | Basic | 1 |
| C6 | MT3608出力バルクコンデンサ（22µF/10V/X5R） | 1608 | 要確認 | Basic | 1 |
| C7 | ATtiny202 VCCバイパスコンデンサ（0.1µF/10V/X7R） | 1608 | C14663 | Basic | 1 |
| L1 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 |
| L2 | パワーインダクタ（4.7µH/Isat≥1A） | 1608〜2012 | 要確認 | Basic候補 | 1 |
| D1 | ショットキーダイオード（B5819W同等） | SOD-123 | C8598 | Basic | 1 |
| BZ1 | 圧電スピーカー接続パッド（パッシブ） | — | — | 別途手配 | 1 |

### 🔴 ユーザー手元での後付け（手はんだ）対象パーツ

| RefDes | 部品名 | 備考 |
|---|---|---|
| SW1 | タクトスイッチ（外付け） | 基板上には実装しない。カートリッジ筐体の最適な場所に取り付け、基板のSW_PAD_A・SW_PAD_Bへ配線する |

---

## 6. 外部接続・インターフェース

### エッジパッド一覧（8パッド・フラット・カスタレーションなし）

| パッド番号 | 信号名 | 方向 | 備考 |
|---|---|---|---|
| Pad 1 | VBAT_IN | 受信 | リポバッテリー入力（3.0V〜4.2V） |
| Pad 2 | GND_IN | 共通 | グラウンド |
| Pad 3 | VCC_3V3_OUT | 送信 | 3.3V降圧Buck出力 |
| Pad 4 | VCC_5V_OUT | 送信 | MT3608昇圧出力（5V） |
| Pad 5 | INT_SDA | 双方向 | 共通I2Cバス SDA（UIAPduino・ATtiny1604・CH1115 OLED共有） |
| Pad 6 | INT_SCL | 双方向 | 共通I2Cバス SCL（同上） |
| Pad 7 | CART_READY | 送信 | CAR-01拡張コネクタ上列1と同一信号。ATtiny202 PA1から出力 |
| Pad 8 | LOCAL_UPDI_A | 双方向 | ATtiny202 PA0。UIAPduino PD6へ基板間配線で接続 |

---

## 7. Claude Code用 回路図自動生成指示書

```text
# 📝 Claude Code Instruction: KiCad Schematic Netlist Generation
#    for CAR-01 Power Management Board A (Rev.2.4 Final Confirmed Version)

## 1. Context & Constraint
- Target Board     : A-Board, Intelligent Power Management Module (CAR-01 Standard)
- Dimensions       : 9.5mm x 27.8mm, PCB Thickness 0.8mm, 2-Layer FR-4
- Layer Rule       : ALL components on Top side only (single-sided assembly).
                     Bottom side = solid GND copper pour, no routing.
- Manufacturing    : ALL components JLCPCB PCBA (full factory assembly).
                     No hand-solder items.
- Footprint Size   : 1608 (0603 inch) metric for all passive components.
- Edge Pads        : 8x flat edge pads (no castellations) for external connection.

## 2. Component Assignments & RefDes

### JLCPCB PCBA Target (All components)
- U1  : ATtiny202 (SOIC-8)             LCSC: C2052970  Extended +$3
- U2  : IRLML6402 P-ch MOSFET (SOT-23) LCSC: C5148470  Extended +$3
- U3  : Buck DC-DC 3.3V (SOT-23-5)     LCSC: RT8008-33GB equivalent
         Vin 2.5V~5.5V, Vout 3.3V fixed, 600mA  Extended +$3
- U4  : MT3608 Boost DC-DC (SOT-23-6)  LCSC: C84817    Extended +$3
- R1  : 10kOhm / +-1% (1608)           MOSFET gate pull-up to VBAT_IN
- R2  : 100kOhm / +-1% (1608)          Battery voltage divider upper
- R3  : 100kOhm / +-1% (1608)          Battery voltage divider lower
- R_upper : 750kOhm / +-1% (1608)      MT3608 Feedback upper resistor (Vout=5.0V)
- R_lower : 100kOhm / +-1% (1608)      MT3608 Feedback lower resistor
- C1  : 0.1uF / 10V / X7R (1608)       Divider node noise bypass
- C2  : 10uF / 10V / X5R (1608)        Buck DCDC input bulk cap
- C3  : 10uF / 10V / X5R (1608)        Buck DCDC output bulk cap
- C4  : 0.1uF / 10V / X7R (1608)       Buck DCDC output bypass
- C5  : 10uF / 10V / X5R (1608)        MT3608 input bulk cap
- C6  : 22uF / 10V / X5R (1608)        MT3608 output bulk cap
- C7  : 0.1uF / 10V / X7R (1608)       ATtiny202 VCC bypass
- L1  : 4.7uH power inductor (1608~2012, Isat >= 1A)  Buck DCDC inductor
- L2  : 4.7uH power inductor (1608~2012, Isat >= 1A)  MT3608 boost inductor
- D1  : Schottky diode SOD-123 (e.g. B5819W, LCSC C8598 Basic)  MT3608 rectifier
- BZ1 : Piezo buzzer passive terminal points (connected to PA1 / CART_READY line)

### SW1 (Tactile Switch):
- Do NOT place a tactile switch component on the PCB.
- Instead, place 2x through-hole solder pads (SW_PAD_A, SW_PAD_B) on the board edge.
  SW_PAD_A connects to Net DIV_NODE (same as PA5).
  SW_PAD_B connects to GND.

## 3. Netlist & Wiring Specifications

### PA1: CART_READY / Answerback BEEP
- U1 PA1 (Pin2) -> Net CART_READY
- BZ1 positive terminal -> CART_READY (passive buzzer in parallel)
- BZ1 negative terminal -> GND
- CART_READY exposed on Edge Pad 7.
  (Sequence:
   1. UIAPduino completes common program initialization
   2. UIAPduino sends "init complete" notification to ATtiny202 via I2C
   3. ATtiny202 drives PA1 HIGH -> CART_READY asserted
   4. Keyboard detects CART_READY HIGH and starts handshake
   5. After handshake: ATtiny202 switches PA1 to TCA0 PWM for answerback BEEP)

### PA0: LOCAL_UPDI_A (Edge Pad 8 -> UIAPduino PD6)
- U1 PA0 (Pin6) -> Edge Pad 8 (LOCAL_UPDI_A)
- Edge Pad 8 connects to UIAPduino PD6 via board-to-board wiring.
- NOT exposed to any other external connection.
- Constraint: During ATtiny202 update, UIAPduino must stop FM audio (YMF825)
  as PD6 is shared with IRQ_N.
- NOTE: UIAPduino PD0 is separately used for LOCAL_UPDI_D (ATtiny1604 / D-board).
  UPDI is point-to-point. Simultaneous update of both devices is not possible.

## 4. Edge Pad Assignment
- Pad 1 : VBAT_IN
- Pad 2 : GND_IN
- Pad 3 : VCC_3V3_OUT
- Pad 4 : VCC_5V_OUT
- Pad 5 : INT_SDA  (common I2C bus SDA shared with UIAPduino, ATtiny1604, CH1115 OLED)
- Pad 6 : INT_SCL  (common I2C bus SCL shared with UIAPduino, ATtiny1604, CH1115 OLED)
- Pad 7 : CART_READY  (= CAR-01 expansion connector upper row pin 1)
- Pad 8 : LOCAL_UPDI_A  (= ATtiny202 PA0, connects to UIAPduino PD6)
```

---

## 8. 設計根拠メモ

### LOCAL_UPDI_Aの接続方式

PA0（LOCAL_UPDI_A）はエッジパッド（Pad 8）として基板端に露出し、UIAPduino PD6へ基板間配線で接続する。Rev.2.2ではPD0に割り当てていたが、PD0/PD6のUPDI割り当て入れ替えによりPD6に変更した。

UPDIはシングルワイヤ・1対1接続専用プロトコルであり、複数デバイスのバス接続はできない。ATtiny1604（D基板）はUIAPduino PD0（LOCAL_UPDI_D）で別途独立して接続する。同時アップデートは不可能であり、A基板とD基板は順番にアップデートする。

### ATtiny202ファームウェアの役割（確定版）

ATtiny202のファームウェアは以下の機能のみを担う。OLEDへの描画処理は一切含まない。

| 機能 | 内容 |
|---|---|
| 電源シーケンス制御 | 電源ボタン押下検出・PA3によるMOSFETオン/オフ |
| ディープスリープ管理 | 電源オフ時のパワーダウンスリープ（数µA）・ボタン押下で覚醒 |
| バッテリー残量監視 | PA5 ADCで定周期測定・内部変数に0〜100%で保持 |
| I2Cスレーブ応答 | UIAPduinoからのReadリクエストに残量値1バイトを返答 |
| CART_READY通知 | UIAPduinoからI2C経由で「初期化完了」通知を受信後、PA1をHIGH出力 |
| アンサーバックBEEP | CART_READY HIGH出力後、PA1をTCA0 PWMに切り替えてBEEP（おまけ） |

2KBのFlashは上記機能のみで十分に収まる。

### 3.3V Buck DCDC選定根拠

RT8008-33GB同等品を採用。リポ放電末期（3.0V入力）でも3.3Vを安定出力できるスイッチング方式を選定。LDOはドロップアウト電圧の懸念があるため不採用。

### MOSFET選定根拠（IRLML6402）

P-ch MOSFETとして、Vgsth（ゲート閾値電圧）が低く（最大-1.35V）、VBAT_INの3.0〜4.2VレンジでATtiny202のPA3（3.3V動作）から確実にオン・オフできる品種を選定。SOT-23の小型パッケージは9.5mm幅の基板制約に適合する。

---

## 9. 未決事項（PENDING）

| # | 項目 | 優先度 | 備考 |
|---|---|---|---|
| 1 | U3（Buck DCDC）LCSC番号確定 | 高 | RT8008-33GB同等品のLCSC在庫確認・選定 |
| 2 | R_upper（750kΩ）LCSC番号確定 | 中 | JLCPCB Basic Parts在庫確認 |
| 3 | C6（22µF/10V/X5R）LCSC番号確定 | 中 | JLCPCB Basic Parts在庫確認 |
| 4 | L1・L2（4.7µH/Isat≥1A）LCSC番号確定 | 中 | 1608〜2012サイズ・Basic候補から選定 |
| 5 | ATtiny202のI2Cスレーブアドレス確定 | 高 | UIAPduino・ATtiny1604と競合しないアドレスを確定 |

---

## 10. 変更履歴

| 日付 | バージョン | 内容 |
|---|---|---|
| — | Rev.1.0 | 初版作成（ATtiny202 I2Cマスター・SSD1306 OLED搭載・タクトスイッチ基板実装構成） |
| 2026-06-01 | Rev.2.0 | ATtiny202のI2C役割をマスター→スレーブに変更。UIAPduino（CH32V003）をI2Cマスターに変更。A基板上のSSD1306 OLEDを撤去・D基板へ移管。タクトスイッチを外付けパッド方式に変更。MT3608 FB抵抗をR_upper 470kΩ→750kΩに修正 |
| 2026-06-10 | Rev.2.1 | 基板サイズを10mm×28mm→9.5mm×27.8mmに修正。PA1 CART_READY動作シーケンス修正（GPIO LOW出力による挿入通知を削除・UIAPduinoからI2C通知を受けてHIGH出力する正しいシーケンスに変更）。PA0 LOCAL_UPDIをエッジパッド経由UIAPduino基板間接続に変更。エッジパッドを7→8個に変更 |
| 2026-06-10 | Rev.2.2 | PA0をLOCAL_UPDI_Aに改名。LOCAL_UPDI_A接続先をUIAPduino PD0に確定。Pad 8をLOCAL_UPDI_A（UIAPduino PD0接続）に更新。Pad5/Pad6に共通I2Cバス共有を明記 |
| 2026-06-11 | Rev.2.3 | LOCAL_UPDI_A接続先をPD0→PD6に変更（PD0/PD6のLOCAL_UPDI割り当て入れ替えに伴う） |
| 2026-06-12 | Rev.2.4 | パターンZ章立てに全面再編。§1「本書の位置づけ」新設。§2「概要・目的」に独立。§3「物理仕様」に独立。§4「回路設計」に電源アーキテクチャ・I2Cバストポロジー・ピンアサイン・コアアーキテクチャを統合。§6「外部接続・インターフェース」をClaude Code指示書から分離・独立。§8「設計根拠メモ」にATtiny202ファームウェア役割表と選定根拠を追加。§9「未決事項（PENDING）」新設（BOM内「要確認」5項目を整理）。目次追加。Claude Code投入用表記を削除。 |
