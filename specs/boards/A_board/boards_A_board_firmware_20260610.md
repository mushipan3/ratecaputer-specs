# A基板（ATtiny202）ファームウェア仕様書

**Rev. 1.0 — 2026年6月10日**

---

## 1. 概要

本書はA基板搭載ATtiny202のファームウェア仕様を定める。ATtiny202はUIAPduino（CH32V003）のI2Cスレーブとして動作し、電源管理・バッテリー残量監視・CART_READY通知・アンサーバックBEEPを担う。OLEDへの描画処理は一切含まない。

ハードウェア仕様は `specs/boards/A_board/hardware.md` を参照。

---

## 2. 機能一覧

| 機能 | 内容 |
|---|---|
| 電源シーケンス制御 | 電源ボタン押下検出・PA3によるMOSFETオン/オフ |
| ディープスリープ管理 | 電源オフ時のパワーダウンスリープ（数µA）・ボタン押下で覚醒 |
| バッテリー残量監視 | PA5 ADCで定周期測定・内部変数に0〜100%で保持 |
| I2Cスレーブ応答 | UIAPduinoからのReadリクエストに残量値1バイトを返答 |
| CART_READY通知 | UIAPduinoからI2C経由で「初期化完了」通知を受信後、PA1をHIGH出力 |
| アンサーバックBEEP | CART_READY HIGH出力後、PA1をTCA0 PWMに切り替えてBEEP（おまけ） |

2KBのFlashは上記機能のみで十分に収まる。

---

## 3. 電源シーケンス制御

### 3.1 電源ONシーケンス

```
ボタン押下（PA5が0V）を検出
    ↓
PA3をLOW出力 → P-ch MOSFET導通
    ↓
VCC_3V3_OUT / VCC_5V_OUT が立ち上がる
    ↓
UIAPduino・B基板が起動
    ↓
UIAPduino共通プログラム初期化完了後、I2C経由でATtiny202に通知
    ↓
ATtiny202がPA1をHIGH出力（CART_READY）
    ↓
ハンドシェイク完了後、PA1をTCA0 PWMに切り替えてBEEP
```

### 3.2 電源OFFシーケンス（長押しシャットダウン）

```
ボタン長押し（約3秒）を検出
    ↓
PA3をHIGH（またはハイインピーダンス）→ MOSFET遮断
    ↓
VCC_3V3_OUT / VCC_5V_OUT が落ちる
    ↓
ATtiny202はパワーダウンスリープへ移行（VBAT_INで常時通電・数µA）
```

### 3.3 PA5判定閾値

| 電圧 | 判定 |
|---|---|
| 1.5V以上 | バッテリー測定モード（通常動作） |
| 0.5V以下 | ボタン押下検出（GNDへ強制短絡） |
| 0.5V〜1.5V | 不定帯域（リポ過放電カットオフ領域・実使用では発生しない） |

**トレードオフ（仕様）：** 長押し中（最大3秒）はバッテリー監視停止。シャットダウン実行中のため問題なしとして許容。

---

## 4. バッテリー残量監視

PA5 ADCで定周期測定。バッテリー電圧範囲3.0V〜4.2Vを1/2分圧（1.5V〜2.1V）してADC入力とする。

```c
// 残量計算（線形近似）
uint8_t battery_percent(uint16_t adc_val) {
    // adc_val: 0〜1023（1.5V=0% / 2.1V=100%）
    if (adc_val <= ADC_MIN) return 0;
    if (adc_val >= ADC_MAX) return 100;
    return (uint8_t)((adc_val - ADC_MIN) * 100 / (ADC_MAX - ADC_MIN));
}
```

測定値はI2Cスレーブ応答用の内部変数に保持し、UIAPduinoからのReadリクエストに即答する。

---

## 5. I2Cスレーブ動作

UIAPduino（CH32V003）をマスターとする共通I2Cバス上でスレーブとして動作する。

| コマンド種別 | 方向 | 内容 |
|---|---|---|
| Read（バッテリー残量） | ATtiny202→UIAPduino | 残量値1バイト（0〜100%） |
| Write（CART_READY通知） | UIAPduino→ATtiny202 | 初期化完了通知・PA1 HIGH出力を指示 |

I2Cスレーブアドレス：TBD

---

## 6. CART_READY通知シーケンス

```
UIAPduino共通プログラム初期化完了
    ↓
UIAPduino → I2C Write → ATtiny202（「初期化完了」通知）
    ↓
ATtiny202: PA1をHIGH出力（CART_READY = HIGH）
    ↓
キーボードがCART_READY HIGHをポーリングまたはHIGHエッジで検出
    ↓
ハンドシェイク開始・完了
    ↓
ATtiny202: PA1をTCA0（Type A Timer）PWM出力に切り替え
    ↓
アンサーバックBEEP吹鳴（おまけ）
```

**注意：** GPIO LOWによる挿入通知は行わない。CART_READYはアクティブHIGH。

---

## 7. アンサーバックBEEP

TCA0（Type A Timer）によるPWM出力でPA1（CART_READY線に並列接続された圧電スピーカー）を駆動する。キーボード側はハンドシェイク完了直後にCART_READY割り込みをマスクするため、PWMパルスが誤割り込みとして扱われることはない。

---

## 変更履歴

| 日付 | 内容 |
|---|---|
| 2026-06-10 | Rev.1.0 新規作成。A_board_spec_20260610.md（Rev.2.2）の§8ファームウェア役割・設計根拠を独立ファイル化・詳細化 |
