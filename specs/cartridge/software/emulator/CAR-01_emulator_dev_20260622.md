# ラテカピューターカートリッジ エミュレータ・開発環境目論見書

**初版：2026年6月15日　改訂：2026年6月19日**
**対象ハードウェア：CAR-01（UIAPduino本体 + A/B/C/D基板構成）**
**ステータス：設計確定・実装着手前（Step A-0-3まで完了）**

---

## 0. 本改訂について

初版（2026-06-15）はCH32V003コア単体（UIAPduino + C基板相当）のエミュレーションを想定していた。本改訂（2026-06-19）では以下の理由により対象を全面拡張する。

- **配布目標の明確化**：カートリッジ実機を持たないユーザーでもアプリストア経由のインストールだけで遊べる状態を最終目標とする
- **対象基板の拡大**：B基板（FM音源・ADPCMミックス）、D基板（ジョイスティック・日本語入力OLED）、A基板（電源管理・バッテリー）を含む全基板のエミュレーションが必要と判明
- **ユーザー層の三層構造化**：「遊ぶだけの一般ユーザー」「アプリ開発者」「mushipan3（プラットフォーム開発者）」で要求されるエミュレーション精度が異なることが判明し、設計を分離

本書は初版の構成を踏襲しつつ、新規調査・決定事項を追加し、開発ステップを全面更新したものである。

---

## 1. 目的と位置付け

### 1.1 エミュレータの位置付け

エミュレータはゲームエンジン開発のためだけのツールではない。**CAR-01プラットフォーム開発全体の基盤ツール**であり、カートリッジ開発工程の最初から使うものである。さらに改訂後の方針では、**最終的な配布物そのもの**（実機を持たないユーザー向けのスタンドアロン版）としての役割も担う。

```
【カートリッジ開発工程】← エミュレータが最初から必要
  共通プログラムのデバッグ
    → iap_call()・iap_return()の動作確認
    → FRAMアドレスマップの検証
    → RcApiテーブルの動作確認

  ブートローダの改変・検証
    → アプリ選択UIの動作確認
    → 恵梨沙フォント表示の確認
    → IAPシーケンスの検証

【ゲームエンジン開発】
  実機なしでバイナリレベルの動作確認
  バグがエンジン（環境）にあるかアプリにあるかの切り分け

【配布物としてのエミュレータ】← 本改訂で追加
  カートリッジ未所持ユーザーがアプリストアから遊べる
  アプリ開発者がカートリッジ到着前から開発・検証できる
```

### 1.2 想定ユーザー層（三層構造）

今回の議論で、要求されるエミュレーション精度がユーザー層によって異なることが明確になった。

| ユーザー層 | 目的 | 必要なエミュレーション精度 |
|---|---|---|
| 一般ユーザー | 遊ぶだけ | CH32V003 CPU・SPI Flash/FRAM・TFT・YMF825・ADPCM・ジョイスティック入力・OLED表示は必須。A基板（電源管理）はI2C応答スタブで十分（バッテリー常時満充電固定） |
| アプリ開発者 | 自作ゲーム・アプリの開発 | 一般ユーザーと同じ。加えてPlatform API・ツールチェーン（変換ツール群）が必要 |
| mushipan3 | プラットフォーム本体・ファームウェアの開発 | 上記全て＋ATtiny202/ATtiny1604のCPUレベル完全エミュレーション、MCP73831充電ICの状態遷移シミュレーション、UPDIデバッグインターフェース |

この三層構造を以後の全セクションで「一般/開発者向け」「mushipan3向け」と区別して記述する。

---

## 2. エミュレータ選定

### 2.1 選定結果

**Spike（riscv-isa-sim）を採用する。**

| 選択肢 | ライセンス | 採否 | 理由 |
|---|---|---|---|
| Spike（riscv-isa-sim） | BSD | ✅ 採用 | 配布制約なし・RV32EC対応・軽量・デバッガ内蔵 |
| QEMU | GPL v2 | ⚠️ 条件付き | 配布時ソース公開義務・周辺実装例が豊富だが制約あり |
| Wokwi | プロプライエタリ | ❌ 不採用 | 改変・組み込み不可 |
| Velxio | AGPL v3 | ❌ 不採用 | AGPLはGPLより厳しい（ネット配信でもソース公開義務） |

**ライセンス選定の根拠：** CAR-01のSDKをMITライセンスで配布する計画があるため、エミュレータもGPLなどの伝染性ライセンスを避けることが重要。SpikeはBSDライセンスで配布制約がない。この方針は本改訂でAVRエミュレータ・YMF825エミュレータの選定にも一貫して適用する（後述）。

### 2.2 Spikeの既存機能

```
CPUコア（RV32EC命令セット） ← これが最大の難所、Spikeが標準対応
デバッガ（-dオプション）：ステップ実行・レジスタダンプ・メモリダンプ・ブレークポイント
メモリマップ設定
```

---

## 3. ハードウェア構成全体像

本改訂で対象を全基板に拡大したため、まず全体のバス構成を整理する。

```
                    I2C共通バス（マスター: UIAPduino）
                         │
          ┌──────────────┼──────────────────┐
          │              │                  │
    [A基板]ATtiny202   [D基板]ATtiny1604   CH1115 OLED
    電源管理・BEEP       ジョイスティック・     48×88ドット
    バッテリー監視        日本語入力UI         日本語入力表示
    CART_READY          (SKRHAAE010)
          │
    UPDI(PD6)    UPDI(PD0)
          │              │
    [UIAPduino = CH32V003 / Spikeでエミュレート]
          │
    SPI共通バス
    ├── [C基板] SPI Flash(16MB) + FRAM×2(1MB)
    ├── [B基板] YMF825(FM音源) + PAM8302A + 3.5mmジャック
    └── TFT ST7789(240×240)

    CH32V003 PC3（TIM1 PWM出力）
    ├── RC-LPF（fc≈10.6kHz）
    └── B基板上でYMF825出力とミックス → AUDIO_MONO_MIX → PAM8302A
```

| 基板 | 搭載チップ | 役割 | エミュレーション方式 |
|---|---|---|---|
| UIAPduino（本体） | CH32V003（RV32EC） | プラットフォーム中核 | Spike（CPUコア完全エミュレーション） |
| C基板 | SPI Flash 16MB / FRAM 1MB | プログラム・データ格納 | Spikeプラグイン（ペリフェラルスタブ） |
| B基板 | YMF825 / PAM8302A | FM音源・音声出力 | Spikeプラグイン（YMF825エミュレータ） |
| D基板 | ATtiny1604 / CH1115 | 入力・日本語入力UI | I2Cスタブ（一般）／AVR完全エミュレーション（mushipan3） |
| A基板 | ATtiny202 / MCP73831 | 電源管理・バッテリー監視 | I2Cスタブ（一般）／AVR完全エミュレーション（mushipan3） |

---

## 4. 実装範囲

### 4.1 対象別実装方法・優先度

| 対象 | 実装方法 | 優先度 | ライセンス課題 |
|---|---|---|---|
| CPUコア（RV32EC） | Spikeがそのまま対応 | ✅不要 | なし |
| 内蔵Flash（16KB）/SRAM（2KB） | Spikeのメモリマップに追加 | 高 | なし |
| SPI Flash（16MB）/FRAM（1MB） | Spikeプラグイン（C++） | 高 | なし |
| TFT（ST7789） | フレームバッファ→PNG/WebSocket出力スタブ | 高 | なし |
| YMF825（FM音源） | 専用エミュレータ（fmfm.core移植、詳細8章） | 高（必須） | 要調査→解消済 |
| TIM1 PWM（ADPCM） | レジスタフック＋デジタルIIRフィルタ（詳細9章） | 高（必須） | なし |
| ATtiny202（A基板） | I2Cスタブ（一般）／AVRエミュレーション（mushipan3、詳細10章） | 高 | 要調査→解消済 |
| ATtiny1604（D基板） | I2Cスタブ（一般）／AVRエミュレーション（mushipan3、詳細10章） | 高 | 要調査→解消済 |
| CH1115 OLED | I2Cコマンド解釈スタブ | 高 | なし |
| MCP73831（充電IC） | mushipan3向けのみ状態遷移スタブ | 中 | なし |
| 拡張コネクタ | プレースホルダのみ（詳細12章） | 低（将来） | 未着手 |

---

## 5. CAR-01メモリマップ（Spike設定用）

```
CH32V003F4P6 内蔵Flash：
  0x00000000〜0x000003FF  ブートローダ（1KB）
  0x00000400〜0x000007FF  ブートローダ続き
  0x00000800〜0x00001BFF  共通プログラム（5.5KB）
  0x00001C00〜0x00001DFF  共通モジュール置き場（512B）
  0x00001E00〜0x00003FFF  App Area（8KB）

CH32V003F4P6 内蔵SRAM：
  0x20000000〜0x200007FF  SRAM（2KB）

SPI Flash（外部・スタブで模擬）：
  0x00000000〜0x000FFFFF  システム領域（1MB）
  0x00100000〜0x00FFFFFF  MOD領域（15MB）

SPI FRAM（外部・スタブで模擬）：
  0x00000〜0x01FFF  システム管理領域（8KB）
  0x02000〜0x05FFF  セーブデータ領域（16KB）
  0x06000〜0x07FFF  共通プログラムバックアップ（8KB）
  0x08000〜0x0FFFF  SPI Flash管理処理（32KB）
  0x10000〜0x13800  コンテキストスタック（〜14KB）
  0x13800〜0x21C00  恵梨沙フォント展開領域（55KB）
  0x21C00〜        アプリ自由作業領域
```

---

## 6. SPI周辺デバイスのスタブ実装方針

SpikeはCSR（制御ステータスレジスタ）へのアクセスをフックできる。SPI1のレジスタアクセスをフックして、疑似的なSPIデバイスとして応答させる。

```cpp
// Spikeプラグインの概念コード
class SpiFlashPlugin : public simif_t {
  uint8_t flash_data[16 * 1024 * 1024];  // 16MBバッファ

  void write(reg_t addr, size_t len, const uint8_t* bytes) override {
    if (addr == SPI1_DATA_REG) {
      process_spi_byte(*bytes);
    }
  }

  bool read(reg_t addr, size_t len, uint8_t* bytes) override {
    if (addr == SPI1_DATA_REG) {
      *bytes = spi_response();
      return true;
    }
    return false;
  }
};
```

SPI1バスにはTFT（CS=PC0）・SPI Flash（CS=PA2）・FRAM（CS=PA1）・YMF825（CS=PD2）が共有接続される。CSピンの状態に応じてルーティング先を切り替えるディスパッチャをプラグイン基盤として実装する（Phase 2 A-2-1）。

重要な制約として、SPIバスが1本のため同時アクセス不可。FRAMとTFTの同時DMAアクセスは不可能なため、FRAMフレームバッファ方式は採用しない設計判断が初版から継続している。

---

## 7. TFTスタブの実装方針

ST7789の描画コマンドをフックし、240×240のフレームバッファとして保持する。

```
フレームバッファ取得方式：
  set_window() / write_pixel() をフックしてRAM上のフレームバッファに書き込み
  フレーム完了タイミングでWebSocket経由でPNGをpush（詳細16章）

解像度：240×240（ST7789実機解像度に統一）
スケーリング：PC側で固定倍率は持たず、CSS側のレスポンシブ表示に委譲
  (image-rendering: pixelated + アスペクト比維持の自動リサイズ)
```

---

## 8. YMF825 FM音源エミュレーション

### 8.1 背景・課題

一般ユーザーが実機なしで遊べることを配布目標とした結果、FM音源（YMF825）のソフトウェアエミュレーションが**必須**となった。YMF825は4オペレータ・8アルゴリズムの独自FM LSIであり、汎用OPL系チップとはレジスタマップが異なるため、既存のOPL/OPM系エミュレータをそのまま流用することはできない。

### 8.2 既存実装調査

GitHub上の関連実装を調査した結果は以下の通り。

| 実装 | 言語 | ライセンス | 評価 |
|---|---|---|---|
| **but80/fmfm.core** | Go | **MIT** | YMF825/YMU765クローンのソフトウェアFMシンセサイザー。OPL3実装をベースにYMF825レジスタ仕様へ対応させた本格実装。**採用候補** |
| hasebems/YMF825_sample | C | **MIT（Yamaha公式）** | Yamaha Corporation自身が2017年に公開したIF仕様・レジスタ定義・音色データ。レジスタマップの一次資料として極めて有用 |
| ywabiko/ymf825, lipoyang/SimpleYMF825 等 | C/C++ | 各種 | いずれもYMF825チップを**ハードウェア越しに制御するドライバ**であり、ソフトウェアエミュレーション（音声合成そのもの）ではない |

**結論：fmfm.core（MIT）が唯一の本格的なソフトウェアエミュレーション実装であり、ライセンス上の制約もない。**

### 8.3 採用方針

fmfm.coreはGo言語実装のため、Spikeプラグイン（C++）への組み込みには移植が必要。

| 案 | 内容 | 採否 |
|---|---|---|
| A: C++移植 | fmfm.coreのFM合成コアをC++に書き直す | ✅採用 |
| B: Goサブプロセス＋WebSocket連携 | fmfm-wasm/CLIをサブプロセス化 | ❌不採用（配布が複雑化するため） |

A案を採用する理由：Spikeプラグインに直接組み込めるため配布構成がシンプルになり、かつYamaha公式MIT資産（レジスタ仕様・音色データ）を直接参照できるため実装精度を上げやすい。

### 8.4 YMF825仕様概要（実装参考）

- 16ボイスポリフォニックFMシンセサイザー
- 29種のオペレーター波形・8アルゴリズム
- 内蔵16bitモノラルDAC、サンプリング周波数48kHz
- 3バンドデジタルイコライザー内蔵
- 4線SPIインターフェース（SS_N・SCK・SI・SO）でホストCPUと接続

### 8.5 実装方針

CS_YMF（PD2）へのSPIアクセスをフックし、レジスタ書き込みをfmfm.core由来のC++ FM合成コアに渡す。出力は48kHzのPCMサンプルストリームとし、9章のADPCM出力とミキシングする。

---

## 9. ADPCM音声エミュレーション（TIM1 PWM）

### 9.1 信号経路

B基板仕様より、ADPCM再生はCH32V003のPWM出力をアナログLPFで平滑化する方式である。

```
CH32V003 PC3（TIM1 PWM出力）
    ↓
RC-LPF（R_LPF 1kΩ + C_LPF 15nF、fc≈10.6kHz）
    ↓
R_PWM（10kΩ混合抵抗）
    ↓
AUDIO_MONO_MIX ← YMF825出力（R3/R4各10kΩ経由）も合流
    ↓
PAM8302A → スピーカー／ヘッドフォン
```

ファームウェアはADPCMをソフトウェアデコードし、デコード済みサンプル値をTIM1のCCR（キャプチャ/コンペアレジスタ）に書き込んでPWMのデューティ比として出力する。

### 9.2 エミュレーション方式

**① TIM1レジスタフック**：TIM1_CCR1への書き込みをSpikeプラグインでフックし、デューティ比をPCMサンプル値として取り出す。スタブが必要な範囲はTIM1_PSC（プリスケーラ）・TIM1_ARR（オートリロード=PWM周期）・TIM1_CCR1の3レジスタのみで、割り込み生成等の完全実装は不要。

**② デジタルRCフィルタ（1次IIR）による等価実現**

```c
// fc=10.6kHz, fs=48kHz の場合
float rc = 1.0f / (2.0f * M_PI * 10600.0f);
float dt = 1.0f / 48000.0f;
float alpha = dt / (rc + dt);  // ≈ 0.741
// y[n] = alpha * x[n] + (1-alpha) * y[n-1]
```

**③ ミキシング**：YMF825サンプルストリームとADPCMサンプルストリームは共に48kHzで生成されるため、サンプル単位の加算で混合できる。実ハードウェアの抵抗ミックス比（R3/R4/R_PWM各10kΩ）も加算係数として再現する。

---

## 10. AVRエミュレーション（ATtiny202／ATtiny1604）

### 10.1 背景

D基板（ジョイスティック・日本語入力OLED）の音量調整コマンド処理、A基板（電源管理・バッテリー監視）の状態をエミュレータ上で再現する必要があるが、一般ユーザー・アプリ開発者にはCPUレベルの完全エミュレーションは不要で、I2C応答のスタブで十分という整理に至った。一方mushipan3にとってはATtiny202/1604ファームウェア自体の開発・検証が必須であり、CPUレベルのエミュレーションが必要。

### 10.2 既存AVRエミュレータのライセンス調査

| 候補 | ライセンス | 対応MCUシリーズ | 評価 |
|---|---|---|---|
| simavr | GPLv3 | tinyAVR 0/1-series（avrxmega3）は**非対応**、開発者から今後対応予定なしとの言及あり | ❌ライセンス以前にそもそも使えない |
| SimulAVR | GPLv2+ | 大容量アドレス空間非対応、世代も古い | ❌ |
| その他（IMAVR等） | ライセンス不明瞭 | - | ❌ |

ATtiny1604・ATtiny202はいずれも**tinyAVR 0-series（avrxmega3アーキテクチャ）**に分類される。調査の結果、唯一実用的とされるsimavrがそもそもこの世代に対応していないことが判明し、**ライセンス問題以前にツールとして使えない**ことが分かった。これにより「自作するしかない」という結論が、ライセンス回避の都合ではなく技術的必然として確定した。

### 10.3 設計方針：二層構造

```
一般ユーザー／アプリ開発者向け（必須・今回実装）
  ATtiny202・ATtiny1604のCPUはエミュレート不要
  I2Cバスのふるまいのみスタブで再現すれば機能上十分

mushipan3向け（ファームウェア開発・検証用）
  ATtiny202・ATtiny1604のCPUを完全エミュレート
  → クリーンルームでのMIT自作AVR8エミュレータ
```

### 10.4 一般・アプリ開発者向け：I2Cスタブ仕様

| デバイス | スタブ内容 |
|---|---|
| ATtiny202（バッテリー監視） | I2C Read要求に対し常に「満充電・USB未接続」相当の固定値を返す |
| ATtiny1604（ジョイスティック） | キーボード入力（WASD+Z+X+Enter）をJOYビットフィールドに変換してI2C経由で返答 |
| ATtiny1604→CH1115（OLED経由処理） | ATtiny1604 CPUをエミュレートせず、OLEDへの描画コマンドを直接panel.htmlの表示エリアにマッピング |
| 音量調整コマンド | ATtiny1604を経由せず、YMF825レジスタ0x19へ直接反映 |

### 10.5 mushipan3向け：自作MIT AVR8エミュレータ仕様

| 項目 | 内容 |
|---|---|
| 対象 | ATtiny202（2KB Flash/256B SRAM）、ATtiny1604（16KB Flash/1KB SRAM） |
| アーキテクチャ | tinyAVR 0-series（avrxmega3） |
| 命令セット | AVR8命令セット（Microchip公開データシートに基づくクリーンルーム実装） |
| 必須ペリフェラル（ATtiny202） | PA5 ADC（バッテリー電圧計算）、PA1 GPIO（MCP73831 STAT読み取り）、PA3 GPIO（MOSFET制御）、TCA0 PWM（BEEP）、TWI I2Cスレーブ |
| 必須ペリフェラル（ATtiny1604） | GPIO×5（JOY_UP/DOWN/LEFT/RIGHT/CENTER）、TWI I2Cスレーブ（対UIAPduino）、I2Cマスター（対CH1115） |
| ライセンス | MIT（自作のため制約なし） |
| 開発時期 | Phase 4〜5（一般ユーザー向け基盤完成後） |

### 10.6 UPDIデバッグインターフェースの扱い

UPDIはプログラミング兼デバッグ専用の単線プロトコルであり、基板間の実通信プロトコル（I2C/SPI/UART）とは層が異なる。エミュレータ文脈では「デバッガがATtinyエミュレータにアタッチする口」として扱う。

```
PC側デバッガ
    ↓ UPDI（仮想）
ATtinyエミュレータ ← ここに対するgdbサーバ相当のインターフェースを用意
    ↓ I2C/SPI/UART（仮想）
CH32V003（Spike）
```

---

## 11. バッテリー残量表示エミュレーション

### 11.1 信号経路

```
[A基板] MCP73831（充電IC）
    ↓ PA1 STATポーリング + PA5 ADC
[A基板] ATtiny202（I2Cスレーブ）
    ↓ I2C Read（UIAPduinoがマスター）
[UIAPduino] CH32V003
    ↓ I2C Write（ATtiny1604スレーブへ）
[D基板] ATtiny1604
    ↓ I2C（CH1115スレーブへ）
[D基板] CH1115 OLED → バッテリー残量バー表示
```

### 11.2 モード別実装

| 層 | 一般ユーザー／開発者 | mushipan3 |
|---|---|---|
| MCP73831（充電IC） | スタブ不要 | 充電中／充電完了／USB未接続の3状態をソフトウェアスイッチで切り替え |
| ATtiny202 I2C応答 | 常に100%固定返答 | ADCカーブ＋STAT込みの完全エミュレーション（10.5節のAVR8コア上で動作） |
| ATtiny1604→CH1115 | I2Cスタブ（直接描画） | ATtiny1604 CPUエミュレーション経由で描画 |
| panel.html表示 | 満充電バー固定表示 | 残量%リアルタイム反映＋電源シーケンスログ表示 |

---

## 12. 拡張コネクタ（将来対応）

14ピン拡張コネクタを介した拡張デバイス接続のエミュレーションは将来実装予定。本改訂時点ではアーキテクチャ上の拡張ポイントとして存在を記録するのみで、実装はスコープ外とする（Phase 7）。

---

## 13. PC版Platformレイヤー（SDL）

### 13.1 目的・位置付け

CAR-01完成を待たずにエンジン開発を先行するための抽象化レイヤー。Catacombsのオリジナルソースと同じ手法（`#ifdef _WIN32`）を採用。本改訂後の配布アーキテクチャ（16章、WebSocket+panel.html方式）とは独立した経路として併存させる。

**SDL2版の提供対象（改訂）**：当初はmushipan3のローカル開発・高速イテレーション専用と位置付けていたが、アプリ開発者にもSDL2版を提供することとした。理由は以下の通り。

- WebView版ではネイティブ↔JS間ブリッジのオーバーヘッド（毎フレーム数ms）が常に存在し、シューティング等の高速描画ジャンルを開発中に「入力の遅れがバグなのか環境の限界なのか」の切り分けが困難になる
- SDL2はzlib licenseであり、バイナリ配布時の表示義務もないため、SDKへの同梱に一切ライセンス上の障壁がない
- アプリ開発者はWindows上でWSL2を使って開発することが前提であり、mushipan3と同じ環境でSDL2版を動かすことが可能

整理すると以下の役割分担となる：SDL2版はmushipan3・アプリ開発者共通の開発時ツール、WebView版・ネイティブ版は配布アプリのエンドユーザー向け（16.5節参照）。

### 13.2 Platform API設計

```c
// platform.h（共通インターフェース）

// レンダリングバックエンド選択（16.5節参照）
typedef enum {
    RENDER_BACKEND_WEBVIEW,  // デフォルト：WebView+生ピクセル配列
    RENDER_BACKEND_NATIVE    // オプトイン：SurfaceView(Android)/Metal(iOS)直描画
} PlatformRenderBackend;

// 描画
void Platform_Init(int width, int height, PlatformRenderBackend backend);
void Platform_DrawPixel(int x, int y, uint16_t color);
void Platform_FillRect(int x, int y, int w, int h, uint16_t color);
void Platform_DrawBitmap(int x, int y, const uint8_t* bitmap, int w, int h);
void Platform_Present(void);

// 入力
uint8_t Platform_GetInput(void);

// SPI Flash / FRAM（PC版はファイルで模擬）
void Platform_FlashRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramWrite(uint32_t addr, const uint8_t* buf, size_t len);

// タイミング
void Platform_SetFrameRate(int fps);
bool Platform_NextFrame(void);
uint32_t Platform_GetTick(void);
```

### 13.3 入力マッピング

| CAR-01（ジョイスティック） | PC版（キーボード） |
|---|---|
| 十字キー上 | W / ↑ |
| 十字キー下 | S / ↓ |
| 十字キー左 | A / ← |
| 十字キー右 | D / → |
| Aボタン | Z / Space |
| Bボタン | X |

---

## 14. 書き込み・読み出しツール群

### 14.1 write_program.py

USB HID経由でCAR-01の内蔵FlashまたはSPI Flashにデータを書き込むPCツール。

```
64バイト固定フォーマット：
  [0]    cmd（コマンド種別）
  [1]    seq（シーケンス番号）
  [2-5]  addr（書き込み先アドレス）
  [6]    length（データ長）
  [7-63] data[56]（データ本体）
```

| cmd | 内容 |
|---|---|
| 0x01 | 内蔵Flash書き込み |
| 0x02 | SPI Flash書き込み |
| 0x03 | 内蔵Flash読み出し |
| 0x04 | SPI Flash読み出し |
| 0x05 | 書き込み完了確認 |

実機待ち事項：rv003usbのHIDディスクリプタがWindows環境でドライバ不要で動作するか（未決事項、22章）。

### 14.2 log_capture.py

FRAMの指定領域からデータを吸い出してPC上でログとして表示するデバッグツール。

### 14.3 dummy_data_gen.py

SPI Flashのテストデータを生成するツール。

---

## 15. 変換ツール群

### 15.1 char_convert.py（最優先）

UTF-8またはShift-JIS文字列を恵梨沙フォントインデックス列（uint16_t[]）に双方向変換するツール。

```
python char_convert.py encode "こんにちは"
# → [0x0034, 0x0068, 0x0012, ...]

python char_convert.py decode 0x0034 0x0068 0x0012
# → "こんにちは"
```

### 15.2 画像変換ツール

```
PNG → RGB565バイナリ（TFT直接書き込み用）
PNG → 1bitバイナリ（Arduboy互換・移植用）
PNG → インデックスカラー256色 + uzlib圧縮（ノベルゲーム背景用）
```

### 15.3 scenario_compile.py

```
scenario.json → scenario.bin
  パラメータ検証
  バランスチェック警告
  バイナリパッキング
  SPI Flashアドレス解決
```

---

## 16. 配布アーキテクチャ

### 16.1 検討経緯：Termux同梱案の棄却

配布初期検討では「エミュレータとTermuxをセットで配布できないか」という案が出たが、以下の理由で棄却した。

- Termuxは現在Google Playに存在せず（ポリシー上の問題で削除済み、配布元はF-Droidのみ）
- 「アプリAをインストールすると自動でアプリBも入る」という仕組みはAndroid/iOSのストア配布モデルに存在しない
- ユーザーに別アプリの事前インストールを案内する形は「アプリストアからのインストールだけで完結」という配布目標と相容れない

### 16.2 採用方針：各OSネイティブ組み込み

Termuxは**mushipan3自身の開発・クロスコンパイル用サンドボックス**としてのみ使用し、エンドユーザー向けには各OS用にSpike＋car01_pluginをネイティブビルドしてアプリバイナリに直接埋め込む。

| プラットフォーム | 方式 |
|---|---|
| Android | NDKでSpike+car01_pluginをARM64の.soとしてビルド→APKに同梱→JNI経由呼び出し |
| iOS | Spikeを静的ライブラリ(.a)としてXcodeビルド→Swiftアプリにリンク |
| Windows | WSL2ビルドのexeをそのまま配布、またはWindows Native(MSVC)ビルド |
| Android（開発用途のみ） | Termuxでのネイティブ実行も将来的な開発者向けオプションとして残す |

SpikeはJITを使わない素直なC++インタプリタであるため、iOSアプリバイナリへの静的リンクに技術的な障壁はない。

### 16.3 App Store政策の調査結果

2026年6月時点の調査で、Appleは2024年4月のガイドライン4.7改訂により、レトロゲームコンソールエミュレータアプリのストア掲載を許可する方針に転換していることを確認した。同年8月にはPCエミュレータについても対象が拡大されている。ラテカピュータは任天堂等の既存ハードウェアを模倣するものではなく独自のオリジナルプラットフォームであるため、著作権上の懸念も小さく、この「レトロコンソールエミュレータ」枠での申請が現実的と判断する。なお実際の申請にあたっては申請時点の最新ガイドライン本文（4.7.1〜4.7.5）を都度確認すること。

### 16.4 panel.htmlの再利用設計

各プラットフォームのネイティブアプリ内に組み込みWebView（Android: WebView、iOS: WKWebView）を配置し、アプリ内部でSpike（子プロセスまたはライブラリ呼び出し）→ローカルWebSocket（または同一プロセス内ブリッジ）→WebView内panel.html、という構成にすることで、Phase 2〜3で開発したUI資産（panel.html・serial_monitor.py）をそのまま全プラットフォームで再利用する。配布版ではLAN越えのネットワーク設定（mirrored networking等、開発時のみ必要）は不要になる。

音声出力はWebAudio API（ブラウザ標準）を利用するため、追加のネイティブオーディオ実装なしに全プラットフォームで動作する。OLED・メニュー・デバッグログ表示はジャンルによらず常にこのWebView経路を使う（16.5節参照）。

### 16.5 レンダリングバックエンド方式（WebView／ネイティブ切替）

**課題の発覚**：panel.htmlはもともと「TFT画面を時々確認できればよい」デバッグパネルとして設計されたものであり、フレームごとのPNGエンコード→WebSocket送信→ブラウザデコードという転送方式は60fps常時表示を想定していなかった。これをそのまま配布版のゲーム画面表示に流用すると、動きの速いジャンル（弾幕シューティング等）で反応速度が不足する懸念がある。

**ボトルネックの切り分け**：問題はWebViewそのものではなく、PNG圧縮往復のCPU負荷にある。240×240のRGB565生フレームは1枚115,200バイト（約112.5KB）であり、60fpsでも転送量は約6.75MB/s程度でローカル内通信としては軽量。PNGエンコード/デコードを廃し、TypedArrayによる生ピクセル配列を`canvas.putImageData()`で描画する方式に変更すれば、ブラウザ内ゲームエミュレータ（NES/GBA系JS実装等）が実例として60fps安定動作している水準まで引き上げられる。

**それでも残る制約**：ネイティブ↔JS間のブリッジ呼び出しには毎フレーム数ms程度のオーバーヘッドが残る。16.6msの予算（60fps）内では大抵許容範囲だが、フレーム精度がシビアなジャンルでは無視できない場合がある。

**採用方針：ジャンル別ハイブリッド方式**

OLED・メニュー・システムUI・WebAudio音声は常にWebView経路で統一する（実装コスト・保守性を優先）。TFTゲーム画面の描画方式のみ、アプリ単位で以下の3方式から選択できるようにする。

| バックエンド | 方式 | 対象 | 工数 |
|---|---|---|---|
| `RENDER_BACKEND_SDL2`（開発時） | SDL2ネイティブウィンドウ | mushipan3・アプリ開発者の開発中動作確認用。Windows配布版にはSDL2.dllを同梱 | SDKに同梱済み |
| `RENDER_BACKEND_WEBVIEW`（配布デフォルト） | canvas + 生ピクセル配列（putImageData） | Android/iOS配布用。ノベル・パズル・RPG等、大半のジャンル | 小（開発者は意識不要） |
| `RENDER_BACKEND_NATIVE`（配布オプトイン） | Android: SurfaceView／iOS: Metal | Android/iOS配布用。弾幕シューティング等、フレーム精度がシビアなジャンル | 中〜大（ホストアプリ側で1回実装すれば全アプリが利用可） |

ネイティブ描画パスはゲームごとに作るのではなく、「ネイティブ希望」と宣言したアプリが使える共通の高速パスとして、Android・iOSそれぞれに1実装ずつ用意する（Phase 6 / A-6-2b・A-6-3bで対応、17章参照）。アプリ側は`Platform_Init()`の引数でバックエンドを指定するだけでよい（13.2節）。

#### SDL2・SurfaceView・Metalの技術的背景

**SDL2（Simple DirectMedia Layer 2）**

Windows/Linux/macOS向けのクロスプラットフォーム描画ライブラリ（zlib license）。ウィンドウ生成・ピクセル描画・入力処理を統一APIで扱える。

CAR-01エミュレータでの用途：
- WSL2(Linux)上での開発中ネイティブウィンドウ表示
- Windows配布版への`SDL2.dll`同梱（ユーザーは何もインストール不要）

ライセンス上の利点：zlib licenseはバイナリ配布時に表示義務がなく、SDK同梱・配布に一切制約がない。

SDL2の内部実装は実際には以下の通りプラットフォーム別に最適な描画APIを使い分けている：
```
SDL2（抽象化レイヤー）
  Windows → Direct3D / OpenGL
  Android → SurfaceView / OpenGL ES  ← SDL2を使っても裏では同じ
  iOS     → Metal / OpenGL ES        ← SDL2を使っても裏では同じ
  Linux   → X11 / Wayland
```
このためモバイルでSDL2を経由することは「ラッパーの層が1つ増えるだけ」となり、Android/iOSではSurfaceView/Metalを直接使う方が合理的である。

---

**SurfaceView（Android）**

AndroidのUIフレームワークは基本的にメインスレッドで動作するが、SurfaceViewはそこから独立した専用描画サーフェスを持ち、別スレッドから直接ピクセルを書き込める。

```
通常のAndroid View:
  メインスレッド → UIフレームワーク → 画面
  （他の処理と競合しやすく描画が遅れることがある）

SurfaceView:
  専用描画スレッド → SurfaceView → 画面
  （独立しているため60fps安定描画が実現しやすい）
```

UnityやUnreal EngineなどのゲームエンジンがAndroidで高速描画できるのはSurfaceView（またはその後継のTextureView）を使っているためである。CAR-01エミュレータでは、Spikeが生成した240×240のピクセル配列をSurfaceViewに直接書き込む形で実装する。

---

**Metal（iOS）**

Appleが2014年にOpenGLの代替として導入した独自グラフィックスAPI。GPUへの命令を「コマンドバッファ」にまとめてGPUに投げる設計で、OpenGLより低オーバーヘッド。現在iOSのすべてのゲームアプリ・AR・カメラ処理がMetalを使用している。

CAR-01エミュレータでの使い方は、GPUの3D能力は不要で「240×240のテクスチャを画面にコピーするだけ」という最もシンプルな用途になる：

```
Spikeのフレームバッファ(240×240 RGB565)
    ↓
MTLTextureに転送
    ↓
フルスクリーンにスケーリングして表示
```

---

**プラットフォーム別の描画担当まとめ**

| プラットフォーム | 開発時 | 配布時（デフォルト） | 配布時（ネイティブオプトイン） |
|---|---|---|---|
| Windows(WSL2) | SDL2 | SDL2.dll同梱exe | SDL2.dll同梱exe |
| Android | SDL2(WSL2) + WebView(最終確認) | SurfaceView非使用・WebView | SurfaceView |
| iOS | SDL2(WSL2) + WebView(最終確認) | Metal非使用・WebView | Metal |

### 16.6 プラットフォーム別配布構成

```
開発時（mushipan3・アプリ開発者）：
  WSL2 Ubuntu 24.04でSpike+car01_plugin+SDL2版エミュレータを使用
  （ブリッジオーバーヘッドなし・高速イテレーション）

配布時（一般ユーザー）：
  Windows:
    car01_emulator.exe + SDL2.dll + car01_plugin.dll + panel/ をzip同梱
    ユーザーはzipを展開してexeをダブルクリックするだけ
    （SDL2.dll同梱のためランタイムインストール不要）

  Android:
    APK内にARM64ネイティブSpike同梱
    WebView+panel.html（デフォルト）
    または SurfaceView直描画パス（RENDER_BACKEND_NATIVE指定時）

  iOS:
    静的リンクしたSpike、WKWebView+panel.html、App Store配布
    または Metal直描画パス（RENDER_BACKEND_NATIVE指定時）
```

## 17. 開発ステップ全体像（Phase 0〜7）

本章は2026-06-19の議論で確定した開発ステップの全体像である。以前の会話で用いていた「Step A-1」「Step A-2」という呼称は本章の「Phase 0 / A-0-x」に対応する（下記対応表参照）。**以降は本章のPhase / A-x-x番号に統一する。**

### 旧呼称との対応関係

| 旧呼称 | 新呼称 |
|---|---|
| Step A-1（Spikeビルド・RV32EC動作確認） | Phase 0 / A-0-3〜A-0-4 |
| Step A-2（car01_plugin骨格） | Phase 0 / A-0-5 |
| Step A-3〜A-5（メモリマップ・スタブ） | Phase 1 / A-1-1〜A-1-3 |

### Phase 0：環境構築
目標：WSL2上でSpikeがRV32ECバイナリを実行できる状態

```
A-0-1  WSL2 Ubuntu 24.04 LTS セットアップ確認
A-0-2  依存パッケージインストール
A-0-3  riscv-isa-sim(Spike) クローン・ビルド・インストール
A-0-4  RV32EC最小バイナリで動作確認
A-0-5  car01_plugin用CMakeプロジェクト骨格作成
```

### Phase 1：CH32V003コアエミュレーション
目標：CH32V003メモリマップ上でファームウェアバイナリが起動できる状態

```
A-1-1  メモリマップ登録（内蔵Flash 16KB / SRAM 2KB）
A-1-2  SPI Flashスタブ（16MB RAM模擬・ファイル永続化）
A-1-3  FRAMスタブ（1MB RAM模擬・ファイル永続化）
A-1-4  Spikeデバッガ接続確認
A-1-5  共通プログラム・IAPシーケンスのエミュレータ上での起動確認
A-1-6  ペリフェラルスタブ補完（A-2-9着手時に判明・Phase 1の漏れとして追加）
       common_prog.elfをSpikeで起動したところhandle_reset()が
       RCC(0x40021000)・SysTick(0xE000F000)にアクセスして即フォルト停止した。
       以下のペリフェラルをcar01_pluginのmmioスタブに追加する：
         RCC     0x40021000〜0x400213FF（クロック制御）
         SysTick 0xE000F000〜0xE000F00F（CH32V003独自・ARM標準と異なるアドレス）
         AFIO    0x40010000〜0x400103FF（代替機能IO）
         GPIOA   0x40010800〜0x40010BFF
         GPIOC   0x40011000〜0x400113FF
         GPIOD   0x40011400〜0x400117FF
       読み書きを受け付けるだけのダミー実装で十分。
       アクセスフォルトが出るたびに順次追加する方針とする。
```

### Phase 2：ペリフェラルスタブ（表示・入力）
目標：TFT画面とジョイスティック入力がpanel.htmlで動く状態

```
A-2-1  SPI1ペリフェラルスタブ基盤（CSピン別ルーティング）
A-2-2  TFT ST7789スタブ（240×240フレームバッファ）
A-2-3  serial_monitor.py拡張（フレームバッファPNG push）
A-2-4  panel.html拡張①（TFT表示エリア追加）
A-2-5  I2Cペリフェラルスタブ基盤（スレーブアドレス別ルーティング）
A-2-6  ATtiny1604 I2Cスタブ（ジョイスティック）
A-2-7  CH1115 OLEDスタブ（48×88）
A-2-8  panel.html拡張②（CH1115表示・デバッグログエリア追加）
A-2-9  共通試験アプリ作成（エミュレータ・実機共通）
       ・TFT基本表示試験（T-1〜T-5）
         T-1: 初期化→白画面
         T-2: 単色塗りつぶし（赤/緑/青）
         T-3: 文字描画（英数字）
         T-4: 日本語表示（恵梨沙フォント）
         T-5: ウィンドウ描画（矩形）
       ・TFT描画速度ベンチマーク（B-1〜B-8）
         B-1: 全画面塗りつぶし速度（FPS計測）
         B-2: 矩形連続描画（大200×200/中100×100/小32×32 各100回）
         B-3: zlib圧縮画像展開・全画面転送速度（展開ms+転送ms+合計KB/s）
         B-4: RGB565全画面直接転送速度（240×240×2B=115.2KBの転送ms）
         B-5: テキスト縦スクロール速度（英数字・1行/フレーム時のFPS）
         B-6: テキスト縦スクロール速度（日本語・恵梨沙フォント）
         B-7: FPSゲーム模擬描画（レイキャスト風壁描画・垂直線240本のFPS）
         B-8: シューティング模擬描画（背景スクロール+敵×10+弾×20の同時描画FPS）
       ・FRAM/SPI Flash読み書き速度計測（P-1〜P-7）
       ・IAP切り替え速度計測（CS-1〜CS-8）
       ・FPS/ms/frame/KB/sをTFT右上に常時表示
       ・ログはRC:xxx形式でUSB HID出力（製品版）またはFRAM 2号（ブレッドボード版）
       参照：test_environment_spec_20260622.md §4b
```

### Phase 3：音声エミュレーション
目標：FM音源とADPCMがブラウザで鳴る状態

```
A-3-1  YMF825 FMエミュレータ（fmfm.core C++移植）
A-3-2  SPI1スタブへのYMF825組み込み（CS_YMF=PD2）← 変更なし
A-3-3  TIM1ペリフェラルスタブ（CCR1フック）
A-3-4  RCフィルタ（1次IIR、fc=10.6kHz）
A-3-5  オーディオミキサー（FM+ADPCM加算・正規化）
A-3-6  WebSocket PCMストリーム送出
A-3-7  panel.html拡張③（WebAudio API受信・再生）
```

### Phase 4：バッテリー残量・電源管理エミュレーション
目標：OLEDにバッテリー残量が表示される状態

```
A-4-1  ATtiny202 I2Cスタブ（一般・開発者向け、満充電固定）
A-4-2  panel.html拡張④（バッテリーバー固定表示）

--- mushipan3向け追加実装 ---
A-4-3  MCP73831充電ICスタブ（3状態切り替え）
A-4-4  ATtiny202完全エミュレーション（自作MIT AVRエミュレータ）
A-4-5  電源シーケンス検証UI（panel.html拡張）
```

### Phase 5：AVRコアエミュレーション（mushipan3向け・ATtiny1604）
目標：ATtiny1604ファームウェアがCPUレベルで動く状態

```
A-5-1  自作MITミニAVRエミュレータ本体（Phase4実装を拡張）
A-5-2  ATtiny1604ペリフェラル追加（GPIO×5・TWI・I2Cマスター）
A-5-3  UPDIデバッガインターフェース
A-5-4  ATtiny1604ファームウェア動作確認
```

### Phase 6：配布向けネイティブアプリ化
目標：App Storeからインストールするだけで遊べる状態

```
A-6-1  エミュレータコアのクロスコンパイル対応（CMake）
A-6-2  Android版（NDK + JNI + WebView、デフォルトバックエンド）
A-6-2b Android ネイティブ描画パス（SurfaceView直描画、オプトイン用）
A-6-3  iOS版（静的リンク + WKWebView、デフォルトバックエンド）
A-6-3b iOS ネイティブ描画パス（Metal直描画、オプトイン用）
A-6-4  Windows版
A-6-5  ゲームカートリッジ(.car)フォーマット定義
A-6-6  App Store / Google Play申請
```

### Phase 7：拡張コネクタ対応（将来）

```
A-7-1  拡張コネクタプロトコルエミュレーション設計
A-7-2  仮想拡張デバイスのプラグインAPI設計
A-7-3  ラテカキーボード仮想接続
A-7-4  その他拡張デバイス（都度追加）
```

### 優先度マップ

```
今すぐ着手  Phase 0 → 1 → 2 → 3
            （UIAPduino書き込みトラブルを回避しながら
             ソフトウェア開発・検証を全て先行できる）

実機完成後  Phase 1-5の実機検証
            write_program.py / HIDプロトコル確認

並行可能    Phase 4（mushipan3向け）はPhase 2完成後いつでも着手可能

最終目標    Phase 6でユーザーに届ける
            Phase 7は継続拡張
```

---

## 18. CH32V003 ハードウェア仕様（エミュレータ実装参考）

### 18.1 CPU

```
コア：RISC-V RV32EC
クロック：48MHz（内蔵PLL）
命令セット：RV32I（整数）+ C（圧縮命令）+ E（レジスタ16個に削減）
```

### 18.2 内蔵ペリフェラル（スタブ実装が必要なもの）

| ペリフェラル | 用途 | スタブ実装優先度 |
|---|---|---|
| SPI1 | TFT・SPI Flash・FRAM・YMF825共有バス | 高 |
| I2C1 | ATtiny202/1604・CH1115向け | 高（本改訂で優先度引き上げ） |
| TIM1 | フレームタイマー・ADPCM PWM音声 | 高 |
| DMA1_Channel3 | SPI1 TX DMA（TFT高速転送） | 中 |
| PFIC | 割り込みコントローラ | 高 |
| USB | USB HID（rv003usb） | 低（書き込みはPC側から行う） |

### 18.3 SPI1の共有バス構成

```
SPI1（MOSI=PC6・MISO=PC7・SCK=PC5）に以下が接続：
  TFT（ST7789）   CS=PC0  DC=PC4  RST=PD5
  SPI Flash       CS=PA2
  FRAM（1チップ） CS=PA1
  YMF825          CS=PD2  RST=PD7（NRST兼用）

重要な制約：
  SPIバスが1本なので同時アクセス不可
  FRAMとTFTの同時DMAアクセスは不可能
  → FRAMフレームバッファ方式は採用しない設計判断
```

**ピン変更履歴（cartridge_master §2.13より）：**
- 2026-06-09：PD3/PD4をUSB専用化に伴いCS_FLASH→PA1・CS_FRAM1→PA2へ移動
- 2026-06-09：CS_FRAM2（PD6）廃止。製品は1チップFRAM構成
- 2026-06-09：DC_DISP→PC4・RST_DISP→PD5・PWM_AUDIO→PC3に再配置
- 2026-06-11：PA1=CS_FRAM1・PA2=CS_FLASHに入れ替え確定

**試験環境仕様書（2026-05）との相違について：**
試験環境仕様書に記載されていた「SPI Flash CS=PD3・FRAM CS=PD4」は2026-06-09のGPIO全面改訂前の旧定義であり、現在は無効。

### 18.4 DMA1_Channel3（SPI1 TX）

```c
// DMA設定（参考）
DMA1_Channel3->PADDR = (uint32_t)&SPI1->DATAR;
DMA1_Channel3->MADDR = (uint32_t)buffer;
DMA1_Channel3->CNTR  = size;
DMA1_Channel3->CFGR  = DMA_CFGR1_DIR | DMA_CFGR1_MINC | DMA_CFGR1_EN;
SPI1->CTLR2 |= SPI_CTLR2_TXDMAEN;
```

---

## 19. 試験仕様書との対応関係

**参照元：** `test_environment_spec_20260622.md`（旧`test_environment_spec_20260520.md`）

試験環境仕様書はCAR-01実機試験の手順書だが、エミュレータ開発においても直接活用できる情報が含まれている。本章はその対応関係を記録する。

### 19.1 試験仕様書から得られた確定情報

**HIDプロトコル（write_program.py）**

未決事項#1として「実機待ち」としていたが、試験仕様書で既に確定済みと判明した。

```
64B固定フォーマット：
  [0]    cmd   （コマンド種別）
  [1]    seq   （シーケンス番号）
  [2-5]  addr  （書き込み先アドレス）
  [6]    length（データ長）
  [7-63] data[56]（データ本体）

VID=0x1209 / PID=0xB803
USB Full Speed物理制約（64B上限）により固定
minichlink不要・pip install hidのみで動作
```

**ブートローダ・共通プログラムのアドレス**

5章メモリマップの内容と一致することを試験仕様書で確認済み。

```
ブートローダ：   0x0000〜0x07FF（UIAPduinoブートローダ、0x0800へジャンプ）
共通プログラム：  0x0800〜（boot_main()エントリポイント）
RcApiテーブル：  0x08A0（割り込みベクタテーブル160B直後）
App Area：       0x1E00〜
```

**ログ設計（LogEntry構造体）**

FRAMログ・panel.htmlデバッグ表示・RC:xxx形式の設計に活用する。

```c
typedef struct {
    uint32_t timestamp_ms;  // 起動からの経過ms
    uint8_t  module_id;     // モジュールID（下表参照）
    uint8_t  event_id;      // イベントID
    uint8_t  payload[10];   // 任意データ（アドレス・値等）
} LogEntry;  // 計16B
// FRAM 0x00000〜の専用領域に格納
// 256KB / 16B = 16,384エントリ蓄積可能
```

| モジュールID | モジュール名 |
|---|---|
| 0x00 | boot |
| 0x01 | spi_manager |
| 0x02 | keyscan |
| 0x03 | tft_oled |
| 0x04 | eink |
| 0x05 | iap |
| 0x06 | app_loader |
| 0x07 | load_resource |
| 0x08 | SPI Flash管理処理 |
| 0x09 | FRAM管理 |
| 0xFE | 試験アプリ |
| 0xFF | システム（boot_main等） |

**iap_call() / iap_return() タイミング期待値**

エミュレータの検証基準・パフォーマンステスト目標値として活用する。

| フェーズ | 設計見込み |
|---|---|
| SRAM退避（2KB→FRAM） | 約0.7ms |
| SPI Flash読み出し（9KB） | 約3ms |
| 内蔵Flash消去（9KB） | 約80ms（最大ボトルネック） |
| 内蔵Flash書き込み（9KB） | 約26ms |
| **iap_call()合計** | **約110ms** |
| **iap_return()合計** | **約115ms** |

### 19.2 試験項目のエミュレータ先行実行

試験仕様書Phase 3の機能試験は実機なしにエミュレータ上で先行実行できる。Phase 2完了後（A-2-8完了時点）から着手可能。

| 試験カテゴリ | 試験ID | エミュレータ対応 |
|---|---|---|
| SPIマネージャ | S-1〜S-4 | ✅ Phase 1完了後から対応可 |
| FRAM | F-1〜F-4 | ✅ Phase 1完了後から対応可 |
| SPI Flash | SF-1〜SF-4 | ✅ Phase 1完了後から対応可 |
| TFT表示 | T-1〜T-5 | ✅ Phase 2完了後から対応可（panel.html目視確認） |
| IAP | I-1〜I-4 | ✅ Phase 2完了後から対応可 |
| リソースロード | LR-1〜LR-3 | ✅ Phase 2完了後から対応可 |
| スループット計測 | P-1〜P-7 | ⚠️ エミュレータ上の計測値は実機と異なる（参考値のみ） |
| コンテキストスイッチ時間 | CS-1〜CS-8 | ⚠️ 同上。19.1節の期待値と比較して論理的正しさの確認のみ |

---



```bash
# エミュレータ起動（デバッグモード）
spike -d --isa=rv32ec \
  --extlib=./car01_plugin.so \
  car01_firmware.elf

# よく使うデバッガコマンド
(spike) reg 0               # レジスタ一覧表示
(spike) pc                  # プログラムカウンタ表示
(spike) mem 0x20000000 64   # SRAMの先頭64バイト表示
(spike) until pc 0x1E00     # App Areaの先頭まで実行
(spike) step                 # 1命令実行
(spike) run 1000             # 1000命令実行

# SPI Flashの特定アドレスを確認（プラグイン経由）
(spike) spiflash 0x098000 256
```

---

## 20. 開発環境構築（mushipan3向け）

### 20.1 前提

WSL2上のVSCodeで開発を行う。Spikeはネイティブ実行できるため、追加の仮想化レイヤーは不要。

**注意点：**
- プロジェクトはWSL2側のファイルシステムに置くこと（`/mnt/c/...`経由はビルドが遅い）
- USBパススルー（実機UART接続時）は別途`usbipd-win`が必要だが、Spike単体（純ソフトシミュレーション）では不要

### 20.2 Step A-0：Spikeビルド手順

```bash
# 1. 依存パッケージのインストール
sudo apt update
sudo apt install -y \
  build-essential autoconf automake libtool \
  device-tree-compiler \
  libboost-regex-dev libboost-system-dev \
  python3 python3-pip \
  git cmake \
  gcc-riscv64-unknown-elf \
  libsdl2-dev

# 2. Spikeのクローン
git clone https://github.com/riscv-software-src/riscv-isa-sim.git
cd riscv-isa-sim

# 3. 【重要】platform.hにCH32V003向けパッチを適用
#    SpikeのDebug Module(0x0)とCH32V003 Flash(0x0)のアドレス衝突を回避する
#    Boot ROM(0x1000)とCH32V003共通プログラム(0x0800〜)の衝突も回避する
sed -i 's/#define DEBUG_START        0x0$/#define DEBUG_START        0x60000000/' \
  riscv/platform.h
sed -i 's/#define DEFAULT_RSTVEC     0x00001000/#define DEFAULT_RSTVEC     0x50000000/' \
  riscv/platform.h

# パッチ確認
grep "DEBUG_START\|DEFAULT_RSTVEC" riscv/platform.h
# → #define DEFAULT_RSTVEC     0x50000000
# → #define DEBUG_START        0x60000000

# 4. ビルド
mkdir build && cd build
../configure --prefix=$HOME/riscv-spike
make -j$(nproc)
make install

# 5. PATHを通す（~/.bashrcに追記）
echo 'export PATH=$HOME/riscv-spike/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 6. RV32EC動作確認（HTIF tohost方式・RV32Eはa7=x17が使えないため）
# ※ 通常のecall(li a7,93)はRV32Eの16レジスタ制約でアセンブルエラーになる
# ※ SpikeのHTIF tohostアドレスに書き込むことで終了を通知する
```

**環境再構築時の注意：** platform.hパッチは手動変更のためSpikeを再クローンした場合は必ず再適用すること（A-1-5で判明・Phase 0実施時に確認済み）。

### 20.2b car01_pluginのビルドと起動

```bash
cd /path/to/ratecaputer_ctrg/emulator/car01_plugin
cmake -B build -DSPIKE_PREFIX=$HOME/riscv-spike
cmake --build build

# 以降はプロジェクトルート（ratecaputer_ctrg/）から実行すること。
# car01_pluginがtft_frame.bin・input_state.bin等をカレントディレクトリに生成するため。
cd /path/to/ratecaputer_ctrg

# 起動コマンド（共通プログラム起動の場合）
export PATH=$HOME/riscv-spike/bin:$PATH
spike --isa=rv32ec \
  -m0x0:0x4000,0x20000000:0x800 \
  --extlib=emulator/car01_plugin/build/libcar01_plugin.so \
  --extension=car01_plugin \
  src/UIAPduino/common_prog/common_prog.elf

# 起動コマンド（試験アプリ起動の場合）
# test_app.binをプロジェクトルートにコピーしてから実行
cp src/UIAPduino/test_app_phase2/test_app.bin .
spike --isa=rv32ec \
  -m0x0:0x4000,0x20000000:0x800 \
  --extlib=emulator/car01_plugin/build/libcar01_plugin.so \
  --extension=car01_plugin \
  src/UIAPduino/common_prog/common_prog.elf
```



WSL2上で同様にC++ツールチェーン（gcc/clang+cmake）を使用する。AVR8命令セットのクリーンルーム実装にはMicrochip公開データシート（tinyAVR 0-series Family Manual）を一次資料として使用する。

---

## 21. 開発環境構築（アプリ開発者向け）

### 21.1 現時点（Phase 6完成前）

Phase 6（配布向けネイティブアプリ化）が完了するまでの間、アプリ開発者はmushipan3と同じWSL2環境でビルド・実行する。

**SDL2版エミュレータ（開発時推奨）：**

```
1. 20.2節と同じ手順でWSL2+Spike環境を構築
2. SDL2をインストール
     sudo apt install -y libsdl2-dev
3. car01_plugin（ビルド済みバイナリ配布予定）とSDL2版エミュレータをビルド
4. RENDER_BACKEND_SDL2指定でアプリを起動→WSL2上にネイティブウィンドウが開く
5. Platform API（13.2節）を使ってアプリ・ゲームを開発
```

SDL2版はWebView版と異なりブリッジオーバーヘッドがないため、シューティング等の高速描画ジャンルでも入力遅延の切り分けが正確に行える。**開発中はSDL2版を使い、配布時のバックエンド指定（WEBVIEW/NATIVE）は最終確認用として使うことを推奨する。**

**WebView版エミュレータ（配布動作確認用）：**

```
1. 上記に加えてserial_monitor.pyを起動
2. ブラウザでpanel.htmlを開く（Windows/WSL2いずれでも可）
3. 配布時の表示・音声・OLEDの最終確認に使う
```

### 21.2 Phase 6完成後（将来）

App Store / Google Play配布アプリに組み込みエミュレータが同梱されるため、開発者は以下のいずれかで開発できるようになる予定。

```
A. 配布アプリ内の開発者モードで直接ロード・実行
B. PC向けスタンドアロン版（exe）でアプリ開発・デバッグ→.carファイル出力
C. .carファイルを実機カートリッジに書き込み、または配布アプリへロード
```

詳細な開発者向け手順は別紙「CAR-01_developer_guide」（配布用統合ガイド）に集約する。

---

## 22. 未決事項

| # | 項目 | 状態 |
|---|---|---|
| 1 | write_program.pyのHIDプロトコル詳細 | ✅ 試験環境仕様書(2026-05)で確定済みと判明。64B固定・cmd/seq/addr/length/data[56]構造体・VID=0x1209/PID=0xB803・USB Full Speed物理制約(64B上限)による |
| 2 | Spikeプラグインのビルド方法 | ✅ WSL2上でCMake |
| 3 | SDL2版Platformレイヤーの解像度設定 | ✅ 240×240に統一 |
| 4 | TFTスタブのスケール係数 | ✅ CSSレスポンシブ対応により解消 |
| 5 | YMF825スタブの音声出力 | ✅ fmfm.core(MIT)のC++移植で完全実装方針確定 |
| 6 | エミュレータのWindows/Mac/Linux対応 | ✅ 開発:WSL2／配布:各OSネイティブ組み込み |
| 7 | rv003usb HIDディスクリプタのWindows動作確認 | 🔒 実機待ち（UIAPduino） |
| 8 | AVRエミュレータのライセンス（新規） | ✅ 自作MIT AVR8エミュレータで解消 |
| 9 | App Store配布ポリシー（新規） | ✅ 2024年4月ガイドライン4.7改訂により解消（申請時に最新ガイドライン要再確認） |
| 10 | .carパッケージフォーマットの仕様（新規） | 🔲 未着手（Phase 6 / A-6-5で設計） |
| 11 | 拡張コネクタプロトコル詳細（新規） | 🔲 未着手（Phase 7） |
| 12 | レンダリングバックエンド方式（新規） | ✅ ジャンル別ハイブリッド方式（WebViewデフォルト／ネイティブはオプトイン）で確定（16.5節） |
| 13 | SDL2のアプリ開発者向け提供・ライセンス（新規） | ✅ zlib licenseにより配布制約なし。SDK同梱・アプリ開発者向け提供を確定 |
| 14 | SPIバスCSピン定義の誤り（新規） | ✅ cartridge_master_20260617 §2.13の変更履歴で判明・修正済み。CS_FLASH=PA2・CS_FRAM=PA1・FRAM2廃止・DC_DISP=PC4・RST_DISP=PD5（18.3節・car01_memmap.h修正済み） |
| 15 | CH32V003ペリフェラルのSpikeスタブ漏れ（新規） | 🔧 対処中。common_prog.elfをSpikeで起動した際にRCC(0x40021000)・SysTick(0xE000F000)へのアクセスフォルトを確認。A-1-6としてcar01_pluginにペリフェラルスタブを追加中 |

実機待ちの2件（#1・#7）は、今回のエミュレータ先行開発の動機そのものが「実機トラブルの回避」であるため、Phase 0〜5の進行のブロッカーにはならない。

---

## 23. 使用ソフトウェアとライセンス一覧

| コンポーネント | ライセンス | 用途 | 配布形態 |
|---|---|---|---|
| Spike (riscv-isa-sim) | BSD | RISC-Vエミュレーションコア | バイナリ同梱 |
| fmfm.core（C++移植） | MIT | FM音源(YMF825)エミュレーション | ソースに組み込み |
| YMF825公式サンプル（参考） | MIT（Yamaha Corporation） | レジスタ仕様リファレンス | 参照のみ |
| SDL2 | zlib license | 開発時ネイティブ表示（mushipan3・アプリ開発者用） | バイナリ同梱（表示義務なし） |
| 自作MIT AVR8エミュレータ | MIT | ATtiny202/1604エミュレーション（mushipan3向け） | ソースに組み込み |

CAR-01 SDK自体もMITライセンスでの配布を予定している。

---

*本書は2026年6月19日時点の設計方針をまとめたものである。実装作業は別チャットで進める。本改訂は同日のチャットでの議論（YMF825/AVR/配布アーキテクチャ調査、Phase 0〜7策定）を反映したものであり、A/B/C/D各基板の詳細仕様は各基板の個別仕様書を一次資料とする。*
