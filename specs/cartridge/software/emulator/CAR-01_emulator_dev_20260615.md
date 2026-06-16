# ラテカピューターカートリッジ エミュレータ・開発環境目論見書

**2026年6月策定**  
**対象ハードウェア：CAR-01（UIAPduino版 / CH32V003 RISC-V 48MHz）**  
**ステータス：方針確定・実装着手前**

---

## 1. 目的と位置付け

### 1.1 エミュレータの位置付け

エミュレータはゲームエンジン開発のためだけのツールではない。**CAR-01プラットフォーム開発全体の基盤ツール**であり、カートリッジ開発工程の最初から使うものである。

```
【カートリッジ開発工程】← エミュレータが最初から必要
  共通プログラムのデバッグ
    → iap_call()・iap_return()の動作確認
    → FRAMアドレスマップの検証
    → RcApiテーブルの動作確認
    → メモリレイアウトの検証

  ブートローダの改変・検証
    → アプリ選択UIの動作確認
    → 恵梨沙フォント表示の確認
    → IAPシーケンスの検証
    → 自己復元機能の検証

  書き込みツールとの連携確認
    → write_program.pyが正しいアドレスに書けているか
    → HIDプロトコルの動作確認

【ゲームエンジン開発】← その次
  エンジンのロジック検証
    → 実機なしでバイナリレベルの動作確認
    → バグがエンジン（環境）にあるかアプリにあるかの切り分け
    → 実機試験と並行してデバッグ
```

### 1.2 現在の開発環境

```
手元にあるもの：
  Arduboy実機 ✅
  PC ✅
  CAR-01カートリッジ ⚠️ 構築中

CAR-01完成前にできること：
  共通プログラム・ブートローダのエミュレータ上での検証
  PC版Platformレイヤー（SDL）でエンジン開発・デバッグ
  Spikeエミュレータでバイナリレベル検証
  書き込みツール・変換ツールの開発
```

---

## 2. エミュレータ選定

### 2.1 選定結果

**Spike（riscv-isa-sim）を採用する。**

| 選択肢 | ライセンス | 採否 | 理由 |
|--------|-----------|------|------|
| Spike（riscv-isa-sim） | BSD | ✅ 採用 | 配布制約なし・RV32EC対応・軽量・デバッガ内蔵 |
| QEMU | GPL v2 | ⚠️ 条件付き | 配布時ソース公開義務・周辺実装例が豊富だが制約あり |
| Wokwi | プロプライエタリ | ❌ 不採用 | 改変・組み込み不可 |
| Velxio | AGPL v3 | ❌ 不採用 | AGPLはGPLより厳しい（ネット配信でもソース公開義務） |

**ライセンス選定の根拠：** CAR-01のSDKをMITライセンスで配布する計画があるため、エミュレータもGPLなどの伝染性ライセンスを避けることが重要。SpikeはBSDライセンスで配布制約がない。

### 2.2 Spikeの既存機能

Spikeが既に提供しているもの（追加実装不要）：

```
CPUコア（RV32EC命令セット） ← これが最大の難所
デバッガ（-dオプション）
  ステップ実行
  レジスタダンプ
  メモリダンプ
  ブレークポイント設定
メモリマップ設定
```

---

## 3. 実装範囲

### 3.1 最小構成（ステップ1対応）

| 対象 | 実装方法 | 優先度 |
|------|---------|--------|
| CPUコア（RV32EC） | Spikeがそのまま対応 | ✅ 不要 |
| 内蔵Flash（16KB） | Spikeのメモリマップに追加 | 高 |
| 内蔵SRAM（2KB） | Spikeのメモリマップに追加 | 高 |
| SPI Flash（16MB） | Spikeプラグイン（C++） | 高 |
| FRAM（256KB/512KB） | Spikeプラグイン（C++） | 高 |
| TFT（ST7789） | フレームバッファ→PNG/SDL出力スタブ | 高 |
| seesawゲームパッド | キー入力スタブ | 中 |
| YMF825 | 無音スタブ（ステップ1では不要） | 低 |
| e-ink | 画像出力スタブ | 低 |

### 3.2 CAR-01メモリマップ（Spike設定用）

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

### 3.3 SPI周辺デバイスのスタブ実装方針

SpikeはCSR（制御ステータスレジスタ）へのアクセスをフックできる。SPI1のレジスタアクセスをフックして、疑似的なSPIデバイスとして応答させる。

```c
// Spikeプラグインの概念コード
class SpiFlashPlugin : public simif_t {
  uint8_t flash_data[16 * 1024 * 1024];  // 16MBバッファ
  
  // SPI1_DATAへの書き込みをフック
  void write(reg_t addr, size_t len, const uint8_t* bytes) override {
    if (addr == SPI1_DATA_REG) {
      process_spi_byte(*bytes);
    }
  }
  
  // SPI1_DATAの読み出しをフック
  bool read(reg_t addr, size_t len, uint8_t* bytes) override {
    if (addr == SPI1_DATA_REG) {
      *bytes = spi_response();
      return true;
    }
    return false;
  }
};
```

### 3.4 TFTスタブの実装方針

ST7789の描画コマンドをフックしてPC上でウィンドウ表示する。

```
方式A：PNG出力（最小実装・SDLなし）
  set_window()・write_pixel()をフックしてフレームバッファに書く
  フレーム完了時にPNGファイルとして出力
  ゲームの各フレームを静止画として確認できる

方式B：SDL2リアルタイム表示（推奨）
  フレームバッファをSDL2テクスチャとして表示
  60FPSでリアルタイムに動作を確認できる
  キーボード入力をseesawゲームパッドとして模擬できる
```

---

## 4. PC版Platformレイヤー（SDL）

### 4.1 目的

CAR-01完成を待たずにエンジン開発を先行するための抽象化レイヤー。Catacombsのオリジナルソースと同じ手法（`#ifdef _WIN32`）を採用。

```
Platformレイヤー：
  CAR-01版：TFT直接描画・SPI Flash・FRAM・seesaw
  PC版：SDL2ウィンドウ・ファイルI/O・キーボード入力
  
  同一のエンジンコードが両方でコンパイル可能
```

### 4.2 Platform API設計

```c
// platform.h（共通インターフェース）

// 描画
void Platform_Init(int width, int height);
void Platform_DrawPixel(int x, int y, uint16_t color);
void Platform_FillRect(int x, int y, int w, int h, uint16_t color);
void Platform_DrawBitmap(int x, int y, const uint8_t* bitmap, int w, int h);
void Platform_Present(void);  // フレーム転送

// 入力
uint8_t Platform_GetInput(void);  // ビットマスクで返す

// SPI Flash / FRAM（PC版はファイルで模擬）
void Platform_FlashRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramWrite(uint32_t addr, const uint8_t* buf, size_t len);

// タイミング
void Platform_SetFrameRate(int fps);
bool Platform_NextFrame(void);
uint32_t Platform_GetTick(void);
```

### 4.3 入力マッピング

| CAR-01（seesaw） | PC版（キーボード） | SDL2定数 |
|-----------------|-----------------|---------|
| 十字キー上 | W / ↑ | SDLK_w / SDLK_UP |
| 十字キー下 | S / ↓ | SDLK_s / SDLK_DOWN |
| 十字キー左 | A / ← | SDLK_a / SDLK_LEFT |
| 十字キー右 | D / → | SDLK_d / SDLK_RIGHT |
| Aボタン | Z / Space | SDLK_z / SDLK_SPACE |
| Bボタン | X | SDLK_x |

---

## 5. 書き込み・読み出しツール群

### 5.1 write_program.py

USB HID経由でCAR-01の内蔵FlashまたはSPI Flashにデータを書き込むPCツール。

**HIDレポートフォーマット（確定済み）：**

```
64バイト固定フォーマット：
  [0]    cmd（コマンド種別）
  [1]    seq（シーケンス番号）
  [2-5]  addr（書き込み先アドレス）
  [6]    length（データ長）
  [7-63] data[56]（データ本体）
```

**実装すべきコマンド：**

| cmd | 内容 |
|-----|------|
| 0x01 | 内蔵Flash書き込み |
| 0x02 | SPI Flash書き込み |
| 0x03 | 内蔵Flash読み出し |
| 0x04 | SPI Flash読み出し |
| 0x05 | 書き込み完了確認 |

**確認が必要な事項：**
- rv003usbのHIDディスクリプタがWindows環境でドライバ不要で動作するか（実機確認必須）

### 5.2 log_capture.py

FRAMの指定領域からデータを吸い出してPC上でログとして表示するツール。デバッグ用。

### 5.3 dummy_data_gen.py

SPI Flashのテストデータを生成するツール。書き込みツールの動作確認に使用。

---

## 6. 変換ツール群

### 6.1 char_convert.py（最優先）

UTF-8またはShift-JIS文字列を恵梨沙フォントインデックス列（uint16_t[]）に双方向変換するツール。

```python
# 使用例
python char_convert.py encode "こんにちは" 
# → [0x0034, 0x0068, 0x0012, ...]（恵梨沙インデックス列）

python char_convert.py decode 0x0034 0x0068 0x0012
# → "こんにちは"
```

TFT表示の最低限テストに必須。write_program.pyと並んで最初に実装すべきツール。

### 6.2 画像変換ツール（scenario_compile.py の一部）

```
PNG → RGB565バイナリ（TFT直接書き込み用）
PNG → 1bitバイナリ（Arduboy互換・移植用）
PNG → インデックスカラー256色 + uzlib圧縮（ノベルゲーム背景用）
```

### 6.3 シナリオコンパイラ（scenario_compile.py）

```
scenario.json → scenario.bin
  パラメータ検証
  バランスチェック警告
  バイナリパッキング
  SPI Flashアドレス解決
```

---

## 7. 実装優先順位とロードマップ

### 7.1 CAR-01完成前（今すぐ着手可能）

```
優先度A（最高）：カートリッジ開発工程の基盤
  write_program.py          USB HID書き込みツール
  char_convert.py           文字コード変換ツール
  Spikeプラグイン基本構成
    内蔵Flash・SRAMのメモリマップ設定
    SPI Flash・FRAMスタブ（共通プログラム検証に必要）
    TFTスタブ（PNG出力・最小構成）

優先度B：共通プログラム・ブートローダ検証
  IAPシーケンスのエミュレータ上での動作確認
  RcApiテーブルの検証
  FRAMアドレスマップの検証
  TFTスタブ（SDL2リアルタイム表示に昇格）
  seesawゲームパッドスタブ（キーボード入力）

優先度C：ゲームエンジン開発支援
  PC版Platformレイヤー（SDL2）
  log_capture.py
  dummy_data_gen.py
  YMF825スタブ（無音→将来実装）
```

### 7.2 CAR-01完成後

```
実機での確認事項：
  write_program.py の実機動作確認
  TFT・SPI Flash・FRAMの動作確認
  IAPの動作確認（iap_run・iap_call・iap_return）
  パフォーマンス実測（FPS・DMA効果）
```

### 7.3 ステップ定義

```
ステップ1（カートリッジ開発基盤）：
  Spikeプラグイン基本構成完成
  共通プログラム・IAPシーケンスのエミュレータ上での動作確認
  write_program.py完成 + 実機でTFT表示確認
  char_convert.py完成

ステップ2（エンジン開発先行）：
  PC版Platformレイヤー完成
  TFTスタブSDL2対応
  シューティングエンジン Scene 1〜4 PC版完成

ステップ3（実機統合）：
  全ツールの実機確認
  エンジンのCAR-01版ビルド確認
  picovaders MOD互換性検証
```

---

## 8. CH32V003 ハードウェア仕様（エミュレータ実装参考）

### 8.1 CPU

```
コア：RISC-V RV32EC
クロック：48MHz（内蔵PLL）
命令セット：RV32I（整数）+ C（圧縮命令）+ E（レジスタ16個に削減）
```

### 8.2 内蔵ペリフェラル（スタブ実装が必要なもの）

| ペリフェラル | 用途 | スタブ実装優先度 |
|------------|------|----------------|
| SPI1 | TFT・SPI Flash・FRAM・YMF825共有バス | 高 |
| I2C1 | seesawゲームパッド | 中 |
| TIM1 | フレームタイマー・PWM音声 | 高 |
| DMA1_Channel3 | SPI1 TX DMA（TFT高速転送） | 中 |
| PFIC | 割り込みコントローラ | 高 |
| USB | USB HID（rv003usb） | 低（書き込みはPC側から行う） |

### 8.3 SPI1の共有バス構成

```
SPI1（MOSI=PC6・MISO=PC7・SCK=PC5）に以下が接続：
  TFT（ST7789）   CS=PC0  DC=PC3
  SPI Flash       CS=PC4
  FRAM-1          CS=PD6
  FRAM-2          CS=PD3（予定）
  YMF825          CS=PC?

重要な制約：
  SPIバスが1本なので同時アクセス不可
  FRAMとTFTの同時DMAアクセスは不可能
  → FRAMフレームバッファ方式は採用しない設計判断
```

### 8.4 DMA1_Channel3（SPI1 TX）

CH32V003でSPI1のTX DMAはDMA1_Channel3が担当することが確認済み（実装例あり）。

```c
// DMA設定（参考）
DMA1_Channel3->PADDR = (uint32_t)&SPI1->DATAR;  // 転送先：SPI1データレジスタ
DMA1_Channel3->MADDR = (uint32_t)buffer;         // 転送元：バッファ
DMA1_Channel3->CNTR  = size;                     // 転送数
DMA1_Channel3->CFGR  = DMA_CFGR1_DIR |          // メモリ→ペリフェラル
                        DMA_CFGR1_MINC |          // メモリアドレス自動増加
                        DMA_CFGR1_EN;             // 有効化
SPI1->CTLR2 |= SPI_CTLR2_TXDMAEN;               // SPI TX DMA有効化
```

---

## 9. デバッガ使用方法（Spike）

### 9.1 基本的な使い方

```bash
# エミュレータ起動（デバッグモード）
spike -d --isa=rv32ec \
  --extlib=./car01_plugin.so \
  car01_firmware.elf

# よく使うデバッガコマンド
(spike) reg 0      # レジスタ一覧表示
(spike) pc         # プログラムカウンタ表示
(spike) mem 0x20000000 64  # SRAMの先頭64バイト表示
(spike) until pc 0x1E00    # App Areaの先頭まで実行
(spike) step        # 1命令実行
(spike) run 1000    # 1000命令実行
```

### 9.2 SPI Flashのデバッグ確認

```bash
# SPI Flashの特定アドレスを確認（プラグイン経由）
(spike) spiflash 0x098000 256  # アプリ領域の先頭256バイト表示
```

---

## 10. 未決事項

| # | 項目 | 内容 |
|---|------|------|
| 1 | write_program.pyのHIDプロトコル詳細 | rv003usb側の実装と合わせて確定 |
| 2 | Spikeプラグインのビルド方法 | CMake設定・依存関係の整理 |
| 3 | SDL2版Platformレイヤーの解像度設定 | 240×240（CAR-01）vs 128×64（Arduboy互換） |
| 4 | TFTスタブのスケール係数 | PC表示時に適切なウィンドウサイズにする |
| 5 | YMF825スタブの音声出力 | 将来的にPCのサウンドカードで鳴らせるか |
| 6 | エミュレータのWindows/Mac/Linux対応 | SpikeはLinux前提・クロスコンパイル要検討 |
| 7 | rv003usb HIDディスクリプタのWindows動作確認 | 実機確認必須（ドライバ不要か） |

---

*本書は2026年6月1日時点の設計方針をまとめたものである。実装作業は別チャットで進める。*
