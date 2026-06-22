# CAR-01 試験環境目論見書

**Rev. 2.0 — 2026年6月22日**

---

## 0. 概要

CAR-01（UIAPduino/CH32V003）の共通プログラムおよびハードウェアの動作確認を行うための試験環境を定義する。

### 試験の目的

1. UIAPduinoブートローダ改変の動作確認
2. 共通プログラム群の各機能の動作確認
3. SPI Flash・FRAMの読み書き動作確認
4. TFTディスプレイの表示確認
5. IAPおよびコンテキストスイッチの動作確認

### 試験の進め方

```
Phase 1: ブートローダ書き換えと確認
    ↓
Phase 2: 共通プログラム書き込みと基本動作確認
    ↓
Phase 2.5: spi_flash_mgr書き込みとSPI Flash試験資材投入
    ↓
Phase 3: 各機能の個別試験（試験アプリから呼び出し）
    ↓
Phase 4: 統合試験（ダミーアプリケーションによる総合確認）
```

**重要**: SPI Flashへの試験資材投入はPhase 2.5で実施する。
spi_flash_mgrが存在しないとSPI Flashにデータを書き込めないため、
Phase 3のSPI Flash試験・日本語表示試験・リソースロード試験が実施できない。

---

## 1. 試験ハードウェア構成

### 1.1 必要な部品・機材

| 部品 | 仕様 | 用途 |
|------|------|------|
| UIAPduino Pro Micro CH32V003 V1.4 | CH32V003F4P6・48MHz | 試験対象MCU |
| SPI FRAM 1号（試験対象） | MB85RS4MTYPF-G-BCE1 256KB or 512KB | 通常動作用 |
| SPI FRAM 2号（ログ用） | 同上 | ログ蓄積用（開発機のみ） |
| SPI Flash | W25Q128JV 16MB | アプリ・データ格納 |
| TFT液晶 | ST7789 1.3inch 240×240px | 表示試験用 |
| ブレッドボード | 標準サイズ | 配線 |
| ブレッドボード電源モジュール | 3.3V出力 | 試験中の給電 |
| USB-Cケーブル | データ通信対応 | プログラム書き込み専用 |
| Windows PC | minichlink動作確認済み | 書き込み・ログ吸い上げ |

**重要**: USB-Cは試験プログラム書き込み専用とする。試験実行中はブレッドボード電源で給電する。USB給電のまま試験プログラムを動かすと、USB HIDとして認識されている状態でGPIOやSPIを操作するため競合が発生する可能性がある。

**試験資材の書き込み手段:**

| 対象 | 手段 | タイミング |
|------|------|----------|
| 内蔵Flash（bootloader_writer・common_prog・spi_flash_mgr・test_app） | write_program.py（USB HID経由） | Phase 1〜2.5 |
| SPI Flash（恵梨沙フォント・カタログ・テストデータ） | spi_flash_mgr経由（USB HID） | Phase 2.5 |
| FRAMログ吸い上げ | log_capture.py（USB HID経由） | Phase 3試験後 |

**minichilink（SWDデバッガ）は使用しない。**

**書き込み順序の原則:**
spi_flash_mgrはUIAPduinoのHID機能でApp Areaに書き込んだ後、
GPIO2=HIGHで起動してSPI FlashへのデータをPC側から投入する。
spi_flash_mgrが存在しないとSPI Flashへの書き込みができない。

### 1.2 ブレッドボード配線

```
ブレッドボード電源モジュール
  → 3.3V レール

UIAPduino（3.3V動作設定）
  VCC → 3.3Vレール
  GND → GNDレール

SPI バス共通配線
  PC5（SCK）  → SCKライン
  PC6（MOSI） → MOSIライン
  PC7（MISO） → MISOライン

SPI FRAM（試験対象・製品版1チップ構成）
  CS  → PA1  ※旧PD4から変更（2026-06-09 USB競合回避・2026-06-11 PA1/PA2入れ替え確定）
  SCK → SCKライン
  MOSI→ MOSIライン
  MISO→ MISOライン
  VCC → 3.3Vレール
  GND → GNDレール

SPI FRAM 2号（ログ用・開発機のみ・試験環境専用）
  CS  → PD6（注: 製品版ではPD6はLOCAL_UPDI_A/IRQ_N兼用。試験環境ではFRAMログ用として流用）
  SCK → SCKライン
  MOSI→ MOSIライン
  MISO→ MISOライン
  VCC → 3.3Vレール
  GND → GNDレール

SPI Flash W25Q128JV
  CS  → PA2  ※旧PD3から変更（2026-06-09 USB競合回避・2026-06-11 PA1/PA2入れ替え確定）
  SCK → SCKライン
  MOSI→ MOSIライン
  MISO→ MISOライン
  VCC → 3.3Vレール
  GND → GNDレール

TFT液晶（ST7789）※TFT機能試験時のみ接続
  CS  → PC0
  DC  → PC4  ※旧PC3から変更（2026-06-09 PWM_AUDIO移動に伴い再配置）
  RST → PD5  ※旧PC4から変更（2026-06-09）
  SCK → SCKライン
  MOSI→ MOSIライン
  VCC → 3.3Vレール
  GND → GNDレール
```

### 1.3 FRAM 2号のCS割り当て（ログ用・試験環境専用）

製品版では1チップFRAM構成（CS=PA1）が確定しており、PD6はLOCAL_UPDI_A/IRQ_N兼用ピンとなっている。試験環境では引き続きFRAM 2号をPD6に接続してログ用途に使用する（製品版との差異は試験環境固有の設定）。

```c
// 試験環境のみ追加定義
#define CS_FRAM      PA1  // FRAM 1号（製品版と同じ）
#define CS_FRAM_LOG  PD6  // FRAM 2号（ログ用・試験環境専用）
// 注意: 製品版ではPD6はLOCAL_UPDI_A/IRQ_N。試験中はYMF825使用不可
```

---

## 2. Phase 1: ブートローダ書き換えと確認

### 2.1 ブートローダ改変の背景

UIAPduinoの標準ブートローダ（rv003usb V1.41）は、タイムアウト後のジャンプ先が0x0800以外になっている可能性がある。ラテカ共通プログラムは0x0800から配置するため、ジャンプ先を0x0800に修正した改変ブートローダへの書き換えが必要。

UIAPduino公式の「Bootloader with modified behavior」はタイムアウト値とコードの一部変更であり、ジャンプ先の変更ではないことをバイナリ解析で確認済み。

### 2.2 書き換え手順

```
1. Claude Codeがrv003usbのソースを確認し
   ジャンプ先を0x0800に修正してビルド
       ↓
2. bootloader_writer.cにバイナリとCRC32を埋め込みビルド
       ↓
3. USB経由でUIAPduinoにbootloader_writer.binを書き込む
   （0x0800〜に配置）
       ↓
4. UIAPduinoが自動再起動
   bootloader_writer.binが起動
       ↓
5. CRC32検証OK → 0x0000〜0x07FFを新ブートローダで上書き
   App Area消去 → ソフトウェアリセット
       ↓
6. 新ブートローダで起動確認
```

### 2.3 書き換え確認アプリ

書き換え成功を視覚的に確認するための最小アプリ。

**動作**: オレンジLED（PD4）を100ms間隔で素早く点滅させる。出荷時の「キャンドル風ゆっくり点灯」と明確に区別できる。

```c
// bootloader_confirm.c
// UIAPduinoのオレンジLED（PD4）を100ms間隔で点滅
// 出荷時のキャンドル風点灯と明確に異なる動作で書き換え成功を確認する

void main(void) {
    SystemInit();
    RCC->APB2PCENR |= RCC_APB2Periph_GPIOD;
    // PD4: 出力 Push-Pull 50MHz
    GPIOD->CFGLR = (GPIOD->CFGLR & ~(0xFu << 16)) | (0x3u << 16);

    while(1) {
        GPIOD->BSHR = (1u << 4);   // HIGH（点灯）
        Delay_Ms(100);
        GPIOD->BCR  = (1u << 4);   // LOW（消灯）
        Delay_Ms(100);
    }
}
```

**成功判定**: 書き換え後にUSB書き込みでこのアプリを0x0800〜に書き込んで実行。100ms点滅が確認できれば書き換え成功。

**失敗時**: 出荷時のキャンドル動作のままであれば書き換え失敗。WCH-LinkEでの復旧が必要になる可能性あり。

---

## 3. Phase 2: 共通プログラム書き込みと基本動作確認

### 3.1 書き込み手順

```
USB経由でcommon_prog.binを0x0800〜に書き込む
    ↓
UIAPduinoブートローダのタイムアウト後に
0x0800（boot_main）へジャンプ
    ↓
200ms待機 → GPIO2確認 → 恵梨沙フォントロード
    ↓
CART_READY HIGH出力
    ↓
アプリ選択UI（App Areaが空の場合）
```

### 3.2 基本動作確認項目

| 確認項目 | 確認方法 | 期待結果 |
|---------|---------|---------|
| 0x0800へのジャンプ | オレンジLEDの動作変化 | boot_main()が起動する |
| 200ms待機 | ロジアナでPA1を観測 | 200ms後にCART_READY HIGH |
| GPIO2確認 | PA2をHIGHにして起動 | 外部通信モード（現在はスタブ） |
| フォントロード | ログ確認 | 約18msで完了 |
| CART_READY | PA1をテスタで確認 | HIGH出力 |
| アプリ選択UI | TFT表示確認 | アプリ一覧画面が表示される |

---

## 4. Phase 3: 各機能の個別試験

試験アプリ（App Areaに書き込むプログラム）から共通プログラムの各機能を呼び出し、所定の結果が得られることを確認する。

### 4.1 入力系のダミー化

試験中はキーボード等の外部入力デバイスを接続しない。試験アプリ内でタイマーやカウンタを使ってイベントを自動発生させることで、入力系に依存せずに機能を試験できるようにする。

```c
// 試験アプリ内でのダミーイベント生成例
static uint32_t tick = 0;

void test_timer_tick(void) {
    tick++;
    // 500ms ごとにイベント発生
    if (tick % 50 == 0) {
        test_event(EVENT_OK_PRESS);
    }
    // 2000ms ごとに別のイベント
    if (tick % 200 == 0) {
        test_event(EVENT_NEXT_TEST);
    }
}
```

### 4.2 試験項目一覧

#### SPIマネージャ試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| S-1 | FRAM書き込み・読み出し | 既知データを書いて読み戻す | データ一致 |
| S-2 | SPI Flash読み出し | カタログ先頭32Bを読む | 有効データ or 0xFF |
| S-3 | SPI速度切り替え | デバイス切り替え後に正常通信 | エラーなし |
| S-4 | CS排他制御 | 複数デバイスを順番にアクセス | 干渉なし |

#### FRAM試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| F-1 | 全アドレス書き込み・読み出し | 0x00000〜末尾まで4KBずつ | データ一致 |
| F-2 | 管理領域キャッシュ | カタログヘッダを書いて読む | 一致 |
| F-3 | セーブデータ領域 | 0x02000〜0x05FFFに書き込み | 一致 |
| F-4 | 作業領域（コンテキストスタック） | 0x10000〜にContextEntryを書く | 一致 |

#### SPI Flash試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| SF-1 | 恵梨沙フォント読み出し | 0x008000から55KB読む | CRC一致 |
| SF-2 | カタログ読み出し | 0x000000から32KB読む | 正常データ |
| SF-3 | アプリ領域書き込み | 0x098000に試験データを書く | 一致 |
| SF-4 | セクタ消去 | 4KBセクタ消去後に読む | 0xFF埋め |

#### TFTディスプレイ試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| T-1 | 初期化 | tft_init()実行 | 画面が白くなる |
| T-2 | 塗りつぶし | tft_fill(赤/緑/青) | 全画面単色 |
| T-3 | 文字描画（英数） | tft_draw_string()でABC | 正しく表示 |
| T-4 | 日本語表示 | 恵梨沙フォントで文字描画 | 正しく表示 |
| T-5 | ウィンドウ描画 | tft_set_window()で矩形 | 指定範囲のみ描画 |

#### IAP試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| I-1 | iap_run()（パターンB） | SPI Flashからブロックを焼く | 再起動後に新ブロックが起動 |
| I-2 | iap_call()（パターンA-1） | 同一アプリ内モジュール呼び出し | モジュールが起動して戻る |
| I-3 | iap_return() | App Area書き戻し確認 | 呼び出し元に正しく戻る |
| I-4 | コンテキストスタック | 多段ネスト（最大7段） | 全段正しく退避・復元 |

#### load_resource()試験

| # | 試験項目 | 試験方法 | 期待結果 |
|---|---------|---------|---------|
| LR-1 | 画像リソースロード | res_type_id=0x1_001でロード | FRAM作業領域に正しく展開 |
| LR-2 | 音声リソースロード | res_type_id=0x2_001でロード | 正しく展開 |
| LR-3 | ディレクトリ拡張 | 512エントリ超のアプリで試験 | 拡張ブロックから正しく検索 |

---

## 5. ログ設計

### 5.1 ログエントリ構造体

```c
typedef struct {
    uint32_t timestamp_ms;  // 起動からの経過ms
    uint8_t  module_id;     // モジュールID
    uint8_t  event_id;      // イベントID
    uint8_t  payload[10];   // 任意データ（アドレス・値等）
} LogEntry;  // 計16B

// FRAM 2号（CS: PD6）の先頭から格納
// 256KB / 16B = 16,384エントリ蓄積可能
```

### 5.2 モジュールID定義

| モジュールID | モジュール名 |
|-------------|------------|
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

### 5.3 イベントID定義（抜粋）

| イベントID | 内容 |
|-----------|------|
| 0x00 | 初期化開始 |
| 0x01 | 初期化完了 |
| 0x10 | 読み出し開始（payloadにアドレス） |
| 0x11 | 読み出し完了（payloadにサイズ・時間） |
| 0x20 | 書き込み開始 |
| 0x21 | 書き込み完了 |
| 0xE0 | エラー発生（payloadにエラーコード） |
| 0xF0 | 試験ステップ開始 |
| 0xF1 | 試験ステップ合格 |
| 0xF2 | 試験ステップ不合格 |

### 5.4 製品版でのデバッグログ

製品版（一般カートリッジ）でも`DEBUG_MODE`定義時は、FRAM作業領域の末尾4KBをログバッファとして使用できる。

```c
#ifdef DEBUG_MODE
  #define LOG_FRAM_ADDR  (FRAM_WORK_BASE + FRAM_WORK_SIZE - 4096)
  #define LOG_BUF_SIZE   4096  // 256エントリ分
#endif
```

---

## 6. PC側ツール

### 6.1 ツール一覧

| ツール | 言語 | 機能 | 優先度 |
|--------|------|------|--------|
| write_program.py | Python | USB HID経由でプログラム書き込み | 高 |
| log_capture.py | Python | FRAM 2号からログを吸い上げ | 高 |
| log_viewer.py | Python | ログの可視化・解析 | 高 |
| dummy_data_gen.py | Python | SPI Flash用ダミーデータ生成 | 高 |
| image_convert.py | Python | 画像→RGB565変換（TFT用） | 中 |
| catalog_gen.py | Python | カタログテーブル生成 | 中 |

### 6.2 write_program.py

USB HID経由でUIAPduinoに対してプログラムを書き込む。

```
python3 write_program.py --file bootloader_confirm.bin --addr 0x0800
python3 write_program.py --file common_prog.bin --addr 0x0800
python3 write_program.py --file test_app.bin --addr 0x2000
```

### 6.3 log_capture.py

FRAM 2号からログエントリを読み出してJSONまたはCSV形式で出力する。

```
python3 log_capture.py --output log_20260519.json
```

### 6.4 log_viewer.py

ログを時系列で可視化する。モジュール別フィルタ・試験合格/不合格の集計機能を持つ。

```
python3 log_viewer.py --input log_20260519.json --module iap
```

### 6.5 dummy_data_gen.py

試験用のSPI Flashデータを生成する。

```
python3 dummy_data_gen.py \
  --app test_app.bin \
  --image test_image_rgb565.bin \
  --output spi_flash_test.bin
```

生成するデータ：
- カタログエントリ（CatalogEntry・538B×1〜数エントリ）
- 恵梨沙フォント（実データまたはダミー）
- ダミーアプリ（試験用バイトコード）
- ダミー画像（RGB565形式・試験用）

### 6.6 image_convert.py

PNG/JPEGをRGB565バイナリに変換してSPI Flash上のリソースとして使用できる形式に変換する。（TFT試験用・まずはこちらから着手）

```
python3 image_convert.py --input test.png --output test_rgb565.bin
```

---

## 7. ダミーアプリケーション仕様

試験アプリはApp Area（0x2000〜・最大10KB）に書き込むプログラム。共通プログラムの各機能を呼び出して結果をログに記録する。

### 7.1 試験アプリの構成

```c
// test_app.c
// 共通プログラムの各機能を順番に試験するアプリ

void app_main(void) {
    log_write(MODULE_TEST, 0xF0, "試験開始");

    // 試験ステップ1: SPI FRAM 読み書き
    test_fram_rw();

    // 試験ステップ2: SPI Flash 読み出し
    test_flash_read();

    // 試験ステップ3: TFT表示
    test_tft_display();

    // 試験ステップ4: load_resource()
    test_load_resource();

    // 試験ステップ5: IAP（パターンB）
    // ※ このステップは再起動を伴うため最後に実行
    test_iap_pattern_b();

    log_write(MODULE_TEST, 0xF1, "全試験完了");
    // TFTに結果表示
    tft_draw_string("ALL PASS", 80, 100, TFT_GREEN, TFT_BLACK);
    for(;;);
}
```

### 7.2 試験結果の判定

- TFT画面への結果表示（PASS/FAIL）
- FRAM 2号へのログ記録
- PC側のlog_viewer.pyで集計・確認

---

## 8. 試験進行順序

```
Step 1: ブートローダ書き換え（Claude Codeで実施）
    └─ 確認: bootloader_confirm.binで100ms点滅確認

Step 2: 共通プログラム書き込み
    └─ 確認: boot_main()起動・CART_READY HIGH・アプリ選択UI表示

Step 3: ダミーデータ作成・SPI Flash書き込み
    └─ dummy_data_gen.pyでSPI Flash用データ生成
    └─ PC側ツールでSPI Flashに書き込み

Step 4: 試験アプリ書き込み
    └─ write_program.pyで0x2000〜に書き込み

Step 5: 各機能の試験実行
    └─ 試験アプリが自動実行・ログをFRAM 2号に記録

Step 6: ログ吸い上げ・解析
    └─ log_capture.pyでログ取得
    └─ log_viewer.pyで結果確認

Step 7: 問題があれば修正→Step 4に戻る
```

---

## 9. 未決事項

| # | 項目 | 状態 | 対応方針 |
|---|------|------|---------|
| 1 | FRAM 2号のCSピン | ✅確定 | PD6を試験環境専用として確定（製品版ではLOCAL_UPDI_A/IRQ_N兼用） |
| 2 | HIDレポートフォーマット | ✅確定 | 64B固定・cmd/seq/addr/length/data[56]の構造体で確定。USB Full Speedの物理制約（64B上限）による |
| 3 | ログのUSB転送方法 | ✅確定 | Phase 3 Task A（spi_flash_mgr HID通信実装）完了後にUSB HID経由でFRAM 2号のログをlog_capture.pyで吸い上げる |
| 4 | e-ink試験環境 | ⏳保留 | e-ink Display調達後に追記 |
| 5 | キースキャン試験 | ✅方針確定 | MCP23017ではなくseesawデバイス（ADA-5743・SKU9115・I2Cアドレス0x50）を使用 |
| 6 | 恵梨沙フォント実データ | ✅方針確定 | Claude Codeで自動取得・dummy_data_gen.pyに組み込んでSPI Flashに書き込み |
| 7 | SPIバスCSピン定義 | ✅修正済み | cartridge_master §2.13変更履歴(2026-06-09/11)により全面改訂。CS_FLASH=PA2・CS_FRAM=PA1・TFT DC=PC4・RST=PD5に更新（本Rev.2.0で反映） |

---

## 9b. エミュレータとの連携

`CAR-01_emulator_dev_20260622.md`のPhase 2完了以降、本仕様書のPhase 3試験項目をエミュレータ上で先行実行できる。実機試験前に論理的なバグを洗い出すことで実機試験の成功率を高める。

| 試験カテゴリ | 試験ID | エミュレータ先行実行 |
|---|---|---|
| SPIマネージャ | S-1〜S-4 | ✅ エミュレータPhase 1完了後から可能 |
| FRAM | F-1〜F-4 | ✅ 同上 |
| SPI Flash | SF-1〜SF-4 | ✅ 同上 |
| TFT表示 | T-1〜T-5 | ✅ エミュレータPhase 2完了後（panel.html目視確認） |
| IAP | I-1〜I-4 | ✅ エミュレータPhase 2完了後 |
| リソースロード | LR-1〜LR-3 | ✅ 同上 |
| スループット計測 | P-1〜P-7 | ⚠️ エミュレータ値は参考のみ（実機との差異あり） |
| コンテキストスイッチ時間 | CS-1〜CS-8 | ⚠️ 論理的正しさの確認のみ。実測値は19.1節の期待値と実機で確認 |

---

## 10. 設計変更履歴

| 日付 | 内容 |
|------|------|
| 2026-05 | Rev.1.0 初版作成 |
| 2026-05 | FRAM 2号CSピン確定（PD6） |
| 2026-05 | HIDレポートフォーマット確定（64B固定・cmd/seq/addr/length/data[56]） |
| 2026-05 | キースキャンをseesawデバイス（ADA-5743）に変更確定 |
| 2026-05 | 試験資材書き込み手段確定（内蔵Flash: USB HID / SPI Flash: 暫定直書き） |
| 2026-05 | 恵梨沙フォント実データ取得方針確定（Claude Codeで自動取得） |
| 2026-05 | Phase 2.5（spi_flash_mgr書き込みとSPI Flash試験資材投入）を追加 |
| 2026-05 | write_program.pyをUSB HID直接通信（rv003usbスクラッチパッドプロトコル）で実装完了 |
| 2026-05 | minichlink不要・pip install hidのみで書き込み可能に |
| 2026-05 | 試験資材書き込み手段を整理（内蔵Flash: UIAPduino HID / SPI Flash: spi_flash_mgr経由） |
| 2026-05 | UIAPduinoのLED仕様確認（PD4・オレンジ・ブートローダで点灯済み） |
| 2026-05 | 標準ブートローダと改変ブートローダのバイナリ解析完了 |
| 2026-06-22 | Rev.2.0 CSピン定義全面修正（cartridge_master §2.13変更履歴 2026-06-09/11反映） |
| 2026-06-22 | §1.2 FRAM CS: PD4→PA1・SPI Flash CS: PD3→PA2・TFT DC: PC3→PC4・TFT RST: PC4→PD5 |
| 2026-06-22 | §1.3 製品版1チップFRAM構成の注記を追加（試験環境と製品版の差異を明記） |
| 2026-06-22 | §9b エミュレータとの連携節を新設 |
| 2026-06-22 | ファイル名を test_environment_spec_20260622.md に更新 |

---

## 4b. 追加試験項目

### スループット・ターンアラウンドタイム計測

試験アプリ内でタイマー（SysTick）を使い各転送のサイクル数を計測してFRAMログに記録する。

```c
// 計測方法（試験アプリ内）
uint32_t t0 = SysTick->CNT;
// 転送処理
uint32_t t1 = SysTick->CNT;
uint32_t cycles = t0 - t1;  // CH32V003は48MHz
uint32_t us = cycles / 48;  // マイクロ秒
log_write(MODULE_TEST, EVENT_PERF, us);
```

#### メモリ間スループット計測試験

| # | 転送経路 | 計測内容 | 計測サイズ |
|---|---------|---------|----------|
| P-1 | USB → SPI Flash | USB HID受信→Flash書き込み | 4KB（1セクタ） |
| P-2 | SPI Flash → FRAM | flash_read→fram_write | 1KB・4KB・16KB |
| P-3 | SPI Flash → 内蔵Flash | IAPでApp Area書き込み | 9KB（App Area） |
| P-4 | SPI Flash → 内蔵SRAM | flash_read→SRAMバッファ | 256B・1KB・2KB |
| P-5 | FRAM → 内蔵Flash | FRAMからIAP書き込み | 9KB |
| P-6 | FRAM → 内蔵SRAM | fram_read→SRAMバッファ | 256B・1KB・2KB |
| P-7 | 内蔵Flash → 内蔵SRAM | memcpy（.ram_func転送） | 256B・1KB |

#### コンテキストスイッチ所要時間計測

| # | 試験項目 | 計測内容 |
|---|---------|---------|
| CS-1 | iap_call()総所要時間 | 呼び出し〜呼び出し先起動まで |
| CS-2 | SRAM退避時間 | sram_save_to_fram()（2KB→FRAM） |
| CS-3 | App Area消去時間 | Flash消去（9KB・64Bセクタ×144回） |
| CS-4 | App Area書き込み時間 | SPI Flash→内蔵Flash（9KB） |
| CS-5 | iap_return()総所要時間 | 戻り処理〜呼び出し元再開まで |
| CS-6 | App Area書き戻し時間 | SPI Flash→内蔵Flash（9KB） |
| CS-7 | メタ情報復元時間 | SPI Flash→FRAM（6KB） |
| CS-8 | SRAM復元時間 | sram_restore_from_fram()（FRAM→2KB） |

**期待値参考（設計値）:**

| フェーズ | 設計見込み |
|---------|----------|
| SRAM退避（2KB→FRAM） | 約0.7ms |
| SPI Flash読み出し（9KB） | 約3ms |
| 内蔵Flash消去（9KB） | 約80ms（最大のボトルネック） |
| 内蔵Flash書き込み（9KB） | 約26ms |
| **iap_call()合計** | **約110ms** |
| iap_return() App Area書き戻し | 約110ms（同上） |
| **iap_return()合計** | **約115ms** |

---

## 5. 試験手順書

### 5.1 事前準備

```
【用意するもの】
□ ブレッドボード・電源モジュール（3.3V）
□ UIAPduino（ブートローダ書き換え済み）
□ SPI FRAM 2個（CS: PD4・PD6）
□ SPI Flash W25Q128JV（CS: PD3）
□ TFT液晶 ST7789（CS: PC0）
□ Windows PC（Python環境済み・pip install hid）
□ USB-Cケーブル（データ通信対応）

【事前作業】
□ ブレッドボード配線完了（test_environment_spec セクション1.2参照）
□ ブートローダ書き換え済み（Phase 1完了）
□ 共通プログラムビルド済み（common_prog.bin）
□ 試験アプリビルド済み（test_app.bin）
□ SPI Flashにダミーデータ書き込み済み（dummy_data_gen.py実行済み）
```

### 5.2 Phase 1: ブートローダ書き換え確認

```
Step 1-1: USBケーブルをPCに接続（USB給電）
  → デバイスマネージャでUIAPduino HIDとして認識されること
  → オレンジLEDがキャンドル風に点灯していること（書き換え前確認）

Step 1-2: bootloader_writer.binを書き込む
  python3 tools/write_program.py --file src/UIAPduino/bootloader_writer/bootloader_writer.bin
  （UIAPduinoをブートローダモードで接続した状態で実行）
  （VID=0x1209 PID=0xB803 でHID直接通信・minichlink不要）

Step 1-3: 自動再起動後の動作確認
  → 数秒後に自動でブートローダ書き換えが実行される
  → 書き換え完了後にリセット

Step 1-4: bootloader_confirm.binを書き込む
  python3 tools/write_program.py --file src/UIAPduino/bootloader_confirm/bootloader_confirm.bin
  → オレンジLEDが100ms間隔で素早く点滅すること【目視確認】
  ← PASS: 書き換え成功
  ← FAIL（キャンドル風のまま）: 書き換え失敗→手順書5.5参照
```

### 5.2b Phase 2.5: spi_flash_mgr書き込みとSPI Flash試験資材投入

**目的**: SPI Flashに試験資材（恵梨沙フォント・ダミーアプリ・テストデータ）を
書き込むためのspi_flash_mgrを起動し、PC側ツールで資材を投入する。

**前提**: Phase 2完了済み・spi_flash_mgr.binのビルド完了（Phase 3 Task A完了）

```
Step 2.5-1: spi_flash_mgr.binをApp Area（0x1C00）に書き込む
  python3 tools/write_program.py --file spi_flash_mgr.bin --addr 0x1C00

Step 2.5-2: GPIO2（PA2）をHIGHにして電源ON
  → extern_comm_run()がspi_flash_mgrをApp Areaから起動
  → PCがVID:0x1209/PID:0xB803のHIDデバイスとして認識すること

Step 2.5-3: 恵梨沙フォントをSPI Flashに書き込む
  python3 tools/dummy_data_gen.py --with-font --output spi_flash_font.bin
  python3 tools/write_program.py --spi-flash --addr 0x008000 --file spi_flash_font.bin

Step 2.5-4: ダミーカタログ・テストデータをSPI Flashに書き込む
  python3 tools/dummy_data_gen.py --output spi_flash_test.bin
  python3 tools/write_program.py --spi-flash --addr 0x000000 --file spi_flash_test.bin

Step 2.5-5: GPIO2をLOWに戻して再起動→通常動作に戻ること確認
  → アプリ選択UIに日本語タイトルが表示されること【目視確認】

Step 2.5-6: 試験アプリをApp Areaに書き込む
  python3 tools/write_program.py --file test_app.bin --addr 0x1C00
```

**完了条件**:
- SPI Flashへの書き込みがエラーなく完了すること
- 通常起動後にアプリ選択UIで日本語タイトルが表示されること
- 試験アプリが起動すること

### 5.3 Phase 2: 共通プログラム基本動作確認

```
Step 2-1: 電源をブレッドボード電源に切り替え
  （USB-Cはプログラム書き込み専用・試験実行中はブレッドボード電源）

Step 2-2: common_prog.binを書き込む
  python3 tools/write_program.py --file common_prog.bin

Step 2-3: 電源ON
  → TFTが白くなること（tft_init()成功）【目視確認】
  → 約220ms後にTFT画面にアプリ一覧が表示されること【目視確認】
    （SPI Flashが空の場合は「NO APP」表示）
  → PA1（CART_READY）がHIGHになること【テスタで確認】

Step 2-4: GPIO2（PA2）をHIGHにして電源ON
  → 外部通信モード（現在はスタブ・ループ停止）
  → TFT表示が止まること【目視確認】
```

### 5.4 Phase 3: 各機能試験

```
Step 3-1: test_app.binをApp Area（0x1C00）に書き込む
  python3 tools/write_program.py --file test_app.bin --addr 0x1C00

Step 3-2: 電源ON（決定ボタン未押下）
  → boot_main()がApp Areaを検出して試験アプリが起動
  → TFTに試験進捗が表示される【目視確認】
  → FRAM 2号（ログ用）にログが蓄積される

Step 3-3: 試験完了後にログを吸い上げ
  python3 tools/log_capture.py --output log_YYYYMMDD_HHMMSS.json

Step 3-4: ログを解析してPASS/FAILを確認
  python3 tools/log_viewer.py --input log_YYYYMMDD_HHMMSS.json

Step 3-5: 目視確認項目
  【T-1】TFT初期化後に画面が白くなること
  【T-2】赤・緑・青の全画面塗りつぶしが正しく表示されること
  【T-3】英数文字が正しく表示されること
  【T-4】日本語文字（恵梨沙フォント）が正しく表示されること
  【P-*】スループット計測値がFRAMログに記録されていること
```

### 5.5 失敗時の対応

| 症状 | 原因候補 | 対応 |
|------|---------|------|
| LED点滅しない | ブートローダ書き換え失敗 | WCH-LinkEで復旧・再試行 |
| TFT表示なし | TFT配線ミス・spi_manager不具合 | 配線確認・S-1試験から再実施 |
| NO APPのまま | SPI Flashへの書き込み未完了 | dummy_data_gen.py→書き込み再実施 |
| ログが空 | FRAM 2号（PD6）未接続 | 配線確認 |
| 全試験FAIL | 共通プログラムのバグ | ログのエラーコードを確認して修正 |

### 5.6 合格基準

```
Phase 1完了条件: オレンジLED 100ms点滅確認
Phase 2完了条件: TFT表示・CART_READY HIGH確認
Phase 3完了条件: 以下を全て満たすこと
  □ S-1〜S-4: SPIマネージャ試験 全PASS
  □ F-1〜F-4: FRAM試験 全PASS
  □ SF-1〜SF-4: SPI Flash試験 全PASS
  □ T-1〜T-5: TFT試験 全PASS（目視確認含む）
  □ I-1〜I-4: IAP試験 全PASS
  □ LR-1〜LR-3: リソースロード試験 全PASS
  □ P-1〜P-7: スループット計測完了（数値記録）
  □ CS-1〜CS-8: コンテキストスイッチ計測完了（数値記録）
```

---

## 6. ログ解析ツール（log_viewer.py）の出力仕様

```
=== 試験結果サマリー ===
試験日時: 2026-XX-XX XX:XX:XX
試験アプリバージョン: X.X

【機能試験】
SPIマネージャ (S-1〜S-4):  4/4 PASS
FRAM         (F-1〜F-4):  4/4 PASS
SPI Flash    (SF-1〜SF-4): 4/4 PASS
TFT          (T-1〜T-5):  5/5 PASS ※目視確認別途
IAP          (I-1〜I-4):  4/4 PASS
リソースロード(LR-1〜LR-3): 3/3 PASS

【スループット計測結果】
P-2 SPI Flash→FRAM (4KB): XXXXus (X.X KB/s)
P-3 SPI Flash→Flash (9KB): XXXXus (X.X KB/s)
...

【コンテキストスイッチ内訳】
CS-2 SRAM退避:      XXXus
CS-3 Flash消去:    XXXXus ← ボトルネック
CS-4 Flash書き込み: XXXXus
CS-1 iap_call()合計: XXXXus
CS-5 iap_return()合計: XXXXus
```
