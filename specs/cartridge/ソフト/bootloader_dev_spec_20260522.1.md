# フリスクカートリッジ CAR-01 共通プログラム開発仕様書

**対象カートリッジ: CAR-01（FRISK × UIAPduino × e-ink / TFT）**
**対象MCU: UIAPduino Pro Micro CH32V003F4P6**
**Rev. 2.1 — 2026年5月**
**本書はClaude Codeへの入力として使用する**

> **注意**: 本書はCAR-01（UIAPduino/CH32V003）専用である。
> XIAO nRF52840（CAR-02）・XIAO ESP32-S3（CAR-03）は
> ブートシーケンスが根本的に異なるため本書の対象外とする。

---

## 0. 全体概要（CAR-01専用）

フリスクカートリッジ（CAR-01）の共通プログラム群を開発する。

**本書の範囲:**
- 起動シーケンス用共通プログラム（内蔵Flash 0x0800〜・機種別枠）
- RAM展開型IAPとIAPコンテキストスイッチ機能
- SPI Flash管理処理（FRAM固定領域・32KB）
- アプリ向け機能の提供形態と設計方針

---

## 0.1 ハードウェア仕様（CAR-01）

| 項目 | 仕様 |
|------|------|
| MCU | CH32V003F4P6 RISC-V 32bit 48MHz |
| 内蔵Flash | 16KB（0x00000000〜0x00003FFF） |
| 内蔵SRAM | 2KB（0x20000000〜0x200007FF） |
| 外付けFRAM | MB85RS4MTYPF-G-BCE1 × **1個** / グレードにより256KB or 512KB |
| SPI Flash | W25Q128JV等 **16MB**（全アプリコンテンツ・恵梨沙フォント格納） |
| FM音源 | YMF825-EZ SPI |
| ディスプレイ（TFT・最初に着手） | ST7789 1.3inch IPS 240×240px（調達中） |
| ディスプレイ（e-ink カラー） | Waveshare 1.54inch e-Paper (G) 200×200px 4色（調達中） |
| ディスプレイ（e-ink 白黒） | Waveshare 1.54inch e-Paper V2 200×200px（調達検討中） |

**外付けSRAMは廃止。FRAMのSRAM相当作業領域で代替する。**

**グレード構成（FRAMのみ異なる・基板フットプリントは同一）:**

| グレード | FRAM | SRAM相当作業領域 | 対象 |
|---------|------|----------------|------|
| プレミアム | 512KB | 448KB | メディアリッチアプリ |
| スタンダード | 256KB | 192KB | シンプルアプリ |

### SPIバス CS割り当て

```c
#define CS_EINK   PC0   // e-ink / TFT兼用
#define CS_YMF    PD2   // YMF825 FM音源
#define CS_FLASH  PD3   // SPI Flash W25Q128JV
#define CS_FRAM   PD4   // SPI FRAM（1チップ）
// PD6: 未使用（解放済み）
```

### その他ピン割り当て

```c
#define DC_EINK      PC3
#define RST_EINK     PC4
#define BUSY_EINK    PD0
#define PWM_AUDIO    PD5
#define CART_READY   PA1   // 拡張コネクタ上列1・準備完了通知（OUTPUT）
#define GPIO2_PIN    PA2   // 拡張コネクタ上列2・KB_NOTIFY確認（INPUT）
#define I2C_SDA      PC1
#define I2C_SCL      PC2
```

---

## 0.2 内蔵Flashレイアウト

UIAPduinoブートローダが先頭2KBを占有するため、共通プログラムは0x0800から配置する。
**サイズはUIAPduinoブートローダ（2KB）を常に含めて計算する。**

**全機種統一レイアウト（リンカスクリプト: ORIGIN=0x00000800, LENGTH=5632）:**
```
0x0000 ┌─────────────────────────┐
       │ UIAPduinoブートローダ   │ 2KB（0x0000〜0x07FF）
0x0800 ├─────────────────────────┤ ← リンカORIGIN・LENGTH=5632（5.5KB）
       │ 共通プログラム領域      │ 5.5KB枠（0x0800〜0x1BFF）
       │ UIAPduino2KB含む合計    │ 7.5KB
0x1C00 ├─────────────────────────┤
       │ 共通モジュール置き場    │ 512B（0x1C00〜0x1DFF）
       │ （開発者自由領域）      │ コンテキストスイッチで上書きされない
0x1E00 ├─────────────────────────┤
       │ App Area               │ 8KB（0x1E00〜0x3FFF）
0x3FFF └─────────────────────────┘
```

| 領域 | アドレス | サイズ | 備考 |
|------|---------|--------|------|
| UIAPduinoブートローダ | 0x0000〜0x07FF | 2KB枠 | 計算基準2,048B |
| 共通プログラム（1層） | 0x0800〜0x1BFF | **5.5KB枠** | リンカLENGTH=5632 |
| 共通モジュール置き場 | 0x1C00〜0x1DFF | **512B** | 開発者自由領域 |
| App Area（3層） | 0x1E00〜0x3FFF | **8KB** | IAPで書き換え |

**実測値（全機能実装・削減完了・確定）:**
- 共通プログラム単体: **5,140B**
- UIAPduinoブートローダ（2,048B）と合算: **7,188B**
- 5.5KB枠（5,632B）に対する余裕: **492B** ✅
- RcApiテーブル配置: **0x08A0**

---

## 0.3 FRAMレイアウト（1チップ・グレードで容量のみ異なる）

```
FRAM（256KB or 512KB・1チップ）
0x00000 ┌──────────────────────┐
        │ 管理領域             │ 8KB（SPI Flashカタログヘッダキャッシュ）
        │                      │ キャッシュ対象: valid/disp_type/title/
        │                      │ flash_addr/total_size (26B) × 58エントリ = 1,508B
        │                      │ block_bitmapはSPI Flash上のみ
0x02000 ├──────────────────────┤
        │ セーブデータ         │ 16KB（0x02000〜0x05FFF）
0x06000 ├──────────────────────┤
        │ 共通プログラム領域   │ 8KB（自己復元用バックアップ）
0x08000 ├──────────────────────┤
        │ SPI Flash管理処理    │ 32KB（固定）
0x10000 ├──────────────────────┤
        │ SRAM相当作業領域     │ プレミアム: 448KB
        │                      │ スタンダード: 192KB
```

### SRAM相当作業領域のレイアウト（参考）

```
0x10000 ┌──────────────────────┐
        │ コンテキストスタック  │ 最大7段 × 2,062B = 14,434B（約14.5KB）
0x13836 ├──────────────────────┤
        │ 恵梨沙フォント展開   │ 55KB（6877文字×8B）
0x21836 ├──────────────────────┤
        │ アプリ自由作業領域   │ プレミアム: 約365KB
        │                      │ スタンダード: 約109KB
```

---

## 0.4 SPI Flash構造（16MB）

```
SPI Flash（16MB・W25Q128JV等）
0x000000 ┌──────────────────────┐
         │ カタログテーブル      │ 32KB固定（58エントリ×538B）
0x008000 ├──────────────────────┤
         │ 恵梨沙フォント        │ 64KB固定（6877文字×8B=55,016B）
0x018000 ├──────────────────────┤
         │ システム予約領域      │ 512KB
         │ （ブートローダ更新    │ 先頭4KBにアップデートプログラム
         │  ブロック含む）       │ +新ブートローダバイナリ+CRC32
0x098000 ├──────────────────────┤
         │ アプリ領域            │ 約15.4MB・4KBブロック×3968本
0xFFFFFF └──────────────────────┘
```

### カタログエントリ構造体（確定版）

```c
typedef struct {
    uint8_t  valid;              // 0xFF=空き / 0x01=有効 / 0x00=削除済み（論理削除）
    uint8_t  disp_type;          // 0x01=e-ink（8KB焼く）/ 0x02=TFT・OLED（10KB焼く）
    uint16_t title[8];           // タイトル（恵梨沙フォントインデックス列・最大8文字）
    uint32_t flash_addr;         // SPI Flash上の先頭アドレス（4KBアライン）
    uint32_t total_size;         // アプリ全体サイズ（バイト）
    uint8_t  block_bitmap[512];  // 4096ブロック×1bit（重複検知用・SPI Flash上のみ）
} CatalogEntry;  // 計538バイト

// テーブル全体: 538B × 58エントリ = 31,204B ≈ 30.5KB
// 50本（ユーザーアプリ）+ 8本（システム予約）= 58エントリ
```

---

## 0.5 UIAPduinoブートローダの3層構造

CH32V003とUIAPduinoの組み合わせでは、ブートローダが3層存在する。

**層1: System Flash（物理固定・書き換え不可）**
CH32V003が工場出荷時から持つUARTブートローダ。別物理領域に存在し消去・上書き不可。通常起動時はスキップされる。我々の開発では原則使用しない。

**層2: UIAPduinoブートローダ（0x0000〜0x07FF・2KB）**
USB HIDでPCと通信。タイムアウト後に0x0800へジャンプして共通プログラムを起動する。ハードウェアプロテクトなし（ソフトウェアガードのみ）。0x0800以降実行中であれば先頭2KBへの書き込みが物理的に可能。

**層3: ラテカ共通プログラム（0x0800〜・機種別枠）**
我々が開発・配置する共通プログラム。UIAPduinoのUSB書き込み機能で0x0800から配置する。

### ブートローダセルフアップデート機能

boot_main()起動時に上下キー同時押しで呼び出す。

```
SPI Flashのシステム予約ブロック内（先頭4KBのみ使用）:
0x000: アップデートプログラム（〜512B・SRAM展開型）
0x200: 新ブートローダバイナリ（2KB）
0xA00: CRC32チェックサム（4B）
```

動作: アップデータをApp Areaへロード→IAP実行→CRC32検証→0x0000〜0x07FF上書き→App Area消去→リセット。WCH-LinkE不要。

---

## 1. 参照リポジトリと流用方針

### 1.1 UIAPduino ブートローダ（rv003usb）

| 機能 | 流用方針 |
|------|---------|
| USB HID列挙 | **そのまま流用**。ドライバ不要でPC・Androidと通信可能 |
| タイムアウト後のジャンプ | **0x0800へのジャンプを確認・必要に応じて修正** |
| USB書き込み処理 | 0x0800以降のみ書き込み可能（ソフトウェアガード） |

**重要制約**: UIAPduinoブートローダはUser Flash先頭2KB（0x0000〜0x07FF）に常駐し、タイムアウト後に0x0800へジャンプする。本共通プログラムは0x0800から配置し、UIAPduinoのUSB書き込み機能で転送する。

**ブートローダのセルフアップデート**: 通常は不要だが、ブートローダを改修する場合はboot_main()の上下キー同時押しでセルフアップデートモードを呼び出す。

### 1.2 Arduboy FX ブートローダ（cathy3k）

| 機能 | 流用方針 |
|------|---------|
| SPI Flashアプリカタログ | 設計参考のみ（本システムはハイブリッド方式を採用） |
| アプリ一覧UIのロジック | ページ切り替え方式の参考として活用 |

---

## 2. モジュール仕様

### モジュール構成と配置

```
内蔵Flash（共通プログラム領域）:
├── boot.c          ブートローダ拡張（〜2KB）
├── spi_manager.c   SPIマネージャ（〜512B）
├── keyscan.c       キースキャン処理（〜1KB）
├── tft_oled.c      TFT・OLEDドライバ（〜1KB）
├── eink.c          e-inkドライバ（〜1.5KB・e-ink機種のみ）
├── iap.c           RAM展開型IAP（〜1KB）
├── iap_ctx.S       IAPコンテキストスイッチアセンブリスタブ（〜512B）
├── app_loader.c    アプリロード処理（〜1KB）
└── load_resource.c リソースロード機能（追加予定・〜200B）
# font.c（英数字フォント177B）は削除済み
# 理由: boot_main()でアプリ選択UI前に恵梨沙フォントをFRAMに転送するため不要

サイズ実績（全機能実装・削減完了・確定）: 5,320B
UIAPduino込み合計: 7,368B / 5.5KB枠（5,632B）に対して余裕312B
```

---

### 2.1 boot.c — ブートローダ拡張

**サイズ目標**: 2KB以内

**役割**: UIAPduinoオリジナルブートローダ（rv003usb）がタイムアウト後に0x0800へジャンプして本モジュールを呼び出す。

**処理フロー**:

```c
void boot_main(void) {
    // 1. ハードウェア初期化
    spi_manager_init();
    tft_init();        // TFT機種 / e-ink機種はeink_init()
    keyscan_init();

    // PA1=CART_READY（OUTPUT）/ PA2=GPIO2（INPUT）設定
    RCC->APB2PCENR |= RCC_APB2Periph_GPIOA;
    GPIOA->CFGLR = (GPIOA->CFGLR & ~(0xFu << 4)) | (0x3u << 4); // PA1出力
    GPIOA->CFGLR = (GPIOA->CFGLR & ~(0xFu << 8)) | (0x4u << 8); // PA2入力

    // 2. 200ms待機（キーボードがGPIO2をセットできるようにする）
    // 理由: カートリッジ電源ONをキーボードが即座に検出する手段がないため
    delay_ms(200);

    // 3. 上下キー同時押し → ブートローダセルフアップデートモード
    uint8_t btn = keyscan_get();
    if ((btn & BTN_UP) && (btn & BTN_DOWN)) {
        bootloader_self_update();  // 戻らない
    }

    // 4. GPIO2（PA2）確認 → 外部通信モード or 通常起動
    if (GPIOA->INDR & (1u << 2)) {
        extern_comm_run();  // spi_flash_mgrをIAP経由でApp Areaにロードして起動（要実装・サイズ検証前に完了）
    }

    // 5. 恵梨沙フォントをSPI Flash（0x008000）→ FRAM作業領域へ転送
    // 転送量: 55KB / 転送時間: 約18ms（SPI 24MHz）
    // この後からアプリタイトル選択画面で日本語表示が可能になる
    font_load_from_flash();

    // 6. CART_READYをHIGHにしてキーボードに準備完了を通知
    GPIOA->BSHR = (1u << 1);

    // 7. 決定ボタン確認
    bool btn_ok = keyscan_get_button(BTN_OK);

    if (!btn_ok) {
        // 8a. App Areaにアプリが存在するか確認
        if (app_area_has_program()) {
            app_loader_jump_to_app();  // 戻らない
        }
    }

    // 8b. アプリロード処理
    app_loader_run();  // 戻らない
}
```

**app_area_has_program()の判定方法**:
App Area先頭4バイトが0xFFFFFFFF（未書き込み）でなければアプリありと判定する。

**自己復元（UIAPduino交換後）**:
App Area未書き込みかつGPIO2=LOWの場合、アプリロード処理へ進む前に
FRAMの共通プログラム領域（0x06000〜0x07FFF）から自己IAPを実行する。
詳細はセクション4.3参照。

**ブートローダセルフアップデート**:
上下キー同時押し検出時（BTN_UP | BTN_DOWN）に呼び出す。

```c
static void bootloader_self_update(void) {
    tft_draw_string("BOOTLOADER UPDATE", 8, 100, TFT_RED, TFT_BLACK);
    tft_draw_string("DO NOT POWER OFF", 8, 116, TFT_WHITE, TFT_BLACK);
    Delay_Ms(2000);
    // SPI Flashのシステム予約ブロックからアップデートプログラムをApp Areaに書き込む
    iap_run(BOOTLOADER_UPDATE_BLOCK_ADDR);  // 戻らない
}
```

---

### 2.2 app_loader.c — アプリロード処理

**サイズ目標**: 1KB以内

**役割**: FRAMの管理領域（SPI Flashカタログのヘッダキャッシュ）からアプリ一覧を取得し、ディスプレイに表示してユーザーに選択させ、選択されたアプリをSPI FlashからIAPで書き込む。

**アプリ一覧表示方式: ページ切り替え**

スクロールは実装せず、ページ切り替え方式とする。
TFT（240×240px）で1画面34行表示可能。50アプリでも2ページ以内に収まる。

```
BTN_UP / BTN_DOWN   : 選択項目移動
BTN_LEFT / BTN_RIGHT: ページ切り替え（表示オフセット変更のみ）
BTN_OK              : 選択確定 → iap_run()
```

**CatalogEntry（FRAMキャッシュ対象はヘッダ部のみ）**:

```c
typedef struct __attribute__((packed)) {
    uint8_t  valid;             // 0x01=有効 / 0xFF=空き / 0x00=削除済み
    uint8_t  disp_type;        // 0x01=e-ink / 0x02=TFT・OLED
    uint16_t title[8];         // 恵梨沙フォントインデックス列（8文字）
    uint32_t flash_addr;       // SPI Flash上の先頭アドレス
    uint32_t total_size;       // 全体サイズ
    uint8_t  block_bitmap[512];// 4096ブロック×1bit（通常起動時不要・SPI Flash上のみ）
} CatalogEntry;                // 計 538B

// FRAMにキャッシュするのはヘッダ部（valid〜total_size = 26B）のみ
// 26B × 58エントリ = 1,508B（8KB管理領域に余裕で収まる）
#define CATALOG_ENTRY_HEADER_SIZE 26
```

**処理フロー**:

```c
void app_loader_run(void) {
    // FRAMキャッシュからアプリ一覧読み込み
    // キャッシュ対象はヘッダ部（26B）のみ。block_bitmap（512B）はSPI Flash上のみ
    CatalogEntry entries[MAX_APPS];
    int count = fram_read_catalog_cache(entries, MAX_APPS);

    int sel = 0, page = 0;
    bool redraw = true;

    for (;;) {
        if (redraw) {
            tft_fill(TFT_BLACK);
            int page_start = page * PAGE_SIZE;
            for (int i = 0; i < PAGE_SIZE && page_start + i < count; i++) {
                int idx = page_start + i;
                // titleはASCIIとして表示（日本語は恵梨沙フォント展開済みのため対応可）
                // ... 表示処理 ...
            }
            redraw = false;
        }

        uint8_t btn = keyscan_wait();
        if (btn & BTN_UP)    { sel = (sel > 0) ? sel-1 : sel; redraw = true; }
        if (btn & BTN_DOWN)  { sel = (sel < count-1) ? sel+1 : sel; redraw = true; }
        if (btn & BTN_LEFT)  { page = (page > 0) ? page-1 : page; redraw = true; }
        if (btn & BTN_RIGHT) { page++; redraw = true; }
        if (btn & BTN_OK)    { iap_run(entries[sel].flash_addr); }
    }
}
```

---

### 2.3 spi_manager.c — SPIマネージャ

**サイズ目標**: 0.5KB以内

```c
typedef enum {
    SPI_DEV_EINK  = 0,  // CS: PC0
    SPI_DEV_YMF   = 1,  // CS: PD2
    SPI_DEV_FLASH = 2,  // CS: PD3（SPI Flash W25Q128JV）
    SPI_DEV_FRAM  = 3,  // CS: PD4（FRAM 1チップ）
} SpiDevice;
// 注: 外付けSRAMは廃止。CS_SRAM(PD3)はCS_FLASHに再割り当て。
//     CS_FRAM2(PD6)は廃止・解放。
```

**実装方針**:
- SPI速度はFRAM 40MHz・SPI Flash 24MHz・e-ink 4MHzと異なるため、`spi_cs_select()`内でデバイスごとにSPIクロック分周を切り替える
- FRAMとSPI FlashはどちらもCH32V003のSPI 24MHz上限で動作（実機検証推奨）

---

### 2.4 eink.c — e-inkディスプレイドライバ

**サイズ目標**: 1.5KB以内（最小実装）

**対象**: 以下のいずれか（調達状況により選択）
- Waveshare 1.54inch e-Paper V2（200×200px・白黒・部分更新対応・調達検討中）
- Waveshare 1.54inch e-Paper (G)（200×200px・4色・静的表示専用・調達中）

初期実装はe-Paper V2（白黒・部分更新）を推奨。e-Paper (G)（4色）は別関数として追加実装する。

**起動シーケンス用の最小実装機能**:

```c
void eink_init(void);
void eink_full_update(void);
void eink_fill(uint16_t color);
void eink_draw_string_elysia(const uint16_t *indices, uint8_t len,
                              uint16_t x, uint16_t y,
                              uint16_t fg, uint16_t bg);
void eink_set_window(uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1);
void eink_write_pixel(uint16_t color);
void eink_sleep(void);
```

**実装状況**: SSD1681（白黒・Waveshare e-Paper V2）対応版を実装済み。LTO実寄与960B。
**e-ink 4色版（UC8154）**: 部品調達後に追加実装予定。

---

### 2.4b tft_oled.c — TFT・OLEDディスプレイドライバ

**サイズ目標**: 1KB以内
**対象**: ST7789 1.3inch IPS 240×240px（AliExpress調達中）
**開発着手**: CAR-01から実装。TFT（ST7789 1.3inch 240×240）が最初の対象。

**起動シーケンス用の最小実装機能**（実測896B・目標以内）:

```c
void tft_init(void);
void tft_fill(uint16_t color);
void tft_set_window(uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1);
void tft_write_pixel(uint16_t color);
/* 恵梨沙フォント描画（FRAM展開済み・8×8px） */
void tft_draw_char_elysia(uint16_t font_idx, uint16_t x, uint16_t y,
                           uint16_t fg, uint16_t bg);
void tft_draw_string_elysia(const uint16_t *indices, uint8_t len,
                              uint16_t x, uint16_t y,
                              uint16_t fg, uint16_t bg);
```

**実装状況**: ST7789対応版を実装済み。ASCII版（tft_draw_char/tft_draw_string）は恵梨沙フォント版に統一済み。

---

### 2.5 iap.c — RAM展開型IAP

**サイズ目標**: 1KB以内（コンテキストスイッチ込み）

**IAPパターン一覧**:

| パターン | 関数 | 戻り | 用途 |
|---------|------|------|------|
| B | iap_run() | なし | ソフトリセット・ノベルゲーム等 |
| A-1 | iap_call() / iap_return() | あり | 同一アプリ内モジュール呼び出し |
| A-2 | iap_call() / iap_return() | あり | 別アプリ呼び出し |

**A-1とA-2はフローが同一。** メタ情報領域はSPI Flashがマスターデータのため
コンテキストスタックへの退避は不要。iap_return()時にcaller_flash_addrから
再読み出すことで完全に復元できる。

**iap_run()（パターンB）の処理フロー**:

```
iap_run(flash_addr)  // SPI Flash上のアドレスを引数に取る
    ↓
SPI Flashの指定アドレスからプログラムブロック（16KB）を読み込み
  ├─ 先頭8KB（両機種統一）→ App Area（0x1E00〜0x3FFF）に書き込む
  └─ 後半8KB（メタ情報領域）→ FRAMのSRAM相当作業領域（FRAM_META_BASE）に転送
    ↓
ソフトウェアリセット（PFIC_SYSTEMRESETまたはWWDGトリガー）
```

**定数（7.5KB統一案・確定）:**
```c
app_base = 0x00001E00UL  // 両機種統一
app_size = 0x00002000UL  // 8KB
meta_size = IAP_BLOCK_SIZE - app_size  // 16KB - 8KB = 8KB
```

**iap_call()（パターンA）の処理フロー**:

```
1. コンテキストスタックに退避（sram_save_to_fram経由）
   ├─ caller_flash_addr（呼び出し元ブロックのSPI Flashアドレス）
   ├─ SP・RA
   └─ SRAMイメージ（2KB）
   ※ メタ情報領域は退避しない（caller_flash_addrから復元可能）

2. 呼び出し先ブロック（8KB・両機種統一）をApp Areaに書き込む
   A-1: メタ情報領域はFRAMに転送しない（呼び出し元のディレクトリを維持）
   A-2: 呼び出し先のメタ情報領域（8KB）をFRAM作業領域に転送

3. ソフトウェアリセット → 呼び出し先が起動
```

**iap_return()の処理フロー**:

```
1. コンテキストスタックからcaller_flash_addrを取り出す
2. SPI Flash caller_flash_addrから8KB → App Area（0x1E00〜0x3FFF）に書き戻す（IAPが必要）
3. SPI Flash caller_flash_addr+8KBから8KB → FRAM作業領域（FRAM_META_BASE）に書き戻す
   （メタ情報領域の復元）
4. SRAMイメージを内蔵SRAMに復元
5. SPを復元してRAにジャンプ → 呼び出し元の続きから再開
```

**ContextEntry構造体（確定版）**:

```c
typedef struct {
    uint8_t  restore_flag;       // 1=有効
    uint8_t  module_id;          // 呼び出し元モジュールID
    uint8_t  call_type;          // 0=IAP_CALL_INTERNAL / 1=IAP_CALL_EXTERNAL
    uint8_t  _pad;               // アライメント予備
    uint32_t sp;                 // 呼び出し元スタックポインタ
    uint32_t ra;                 // 呼び出し元戻りアドレス
    uint32_t caller_flash_addr;  // 呼び出し元ブロックのSPI Flashアドレス
    uint8_t  sram_image[2048];   // 内蔵SRAM 2KBスナップショット
} ContextEntry;  // 計2,064B（+2B）

// 最大7段: 2,064B × 7 + ヘッダ16B = 14,464B ≈ 14.5KB

// iap_call()の呼び出し種別
#define IAP_CALL_INTERNAL  0  // A-1: 同一アプリ内モジュール
#define IAP_CALL_EXTERNAL  1  // A-2: 別アプリ

// 情報受け渡し領域（FRAM作業領域内・固定アドレス）
#define IAP_ARG_INTERNAL   0x00016200UL  // A-1引数（64B）
#define IAP_RET_INTERNAL   0x00016240UL  // A-1戻り値（64B）
#define IAP_ARG_EXTERNAL   0x00016280UL  // A-2引数（64B）
#define IAP_RET_EXTERNAL   0x000162C0UL  // A-2戻り値（64B）
```

**PENDING（未決事項）**:

| # | 項目 | 内容 |
|---|------|------|
| 1 | iap_call()/iap_return()の実装 | ✅実装済み：caller_flash_addr含むContextEntry書き込み・App Area書き戻し・メタ情報転送 |
| 2 | コンテキストスイッチ前後の情報受け渡し | ✅解決：FRAM固定領域4区画（A-1/A-2各64B・0x16680〜0x177F） |
| 3 | パターンA-1/A-2の判別方法 | ✅解決：第3引数call_type（IAP_CALL_INTERNAL=0/IAP_CALL_EXTERNAL=1） |
| 4 | iap_return()の所要時間計測 | ⏳実機試験時に計測（試験項目CS-5） |

---

### 2.5b iap_ctx.S — IAPコンテキストスイッチ（アセンブリスタブ）

**詳細仕様**: セクション5B.5を必ず参照すること。

```asm
.global iap_call_entry
iap_call_entry:
    mv   t0, ra          # 呼び出し元RA（ABI確立前に取得）
    mv   t1, sp          # 呼び出し元SP（ABI確立前に取得）
    mv   a2, a1
    mv   a1, t0
    mv   a0, t1
    call sram_save_to_fram   # C関数でFRAMへ退避
    call iap_run_internal    # Flash書き換え→リセット（戻らない）

.global iap_return_entry
iap_return_entry:
    call sram_restore_from_fram  # FRAMからSRAMイメージ復元・a0=sp・a1=ra
    mv   sp, a0
    jr   a1              # 呼び出し元へ復帰
```

---

### 2.6 keyscan.c — キースキャン処理

**サイズ目標**: 1KB以内

**デバイス**: Adafruit seesawミニI2Cゲームパッド（ADA-5743・スイッチサイエンス SKU9115）
**I2Cアドレス**: 0x50（seesaw標準）
**接続**: SDA=PC1 / SCL=PC2 / STEMMA QT互換

**実装状況**: seesaw対応版を実装済み（ADA-5743・0x50・I2C 400kHz）。ジョイスティック（アナログ2軸）とボタン6個に対応。

```c
#define SEESAW_ADDR    0x50

#define BTN_UP     0x01
#define BTN_DOWN   0x02
#define BTN_LEFT   0x04
#define BTN_RIGHT  0x08
#define BTN_OK     0x10
#define BTN_CANCEL 0x20

void    keyscan_init(void);
uint8_t keyscan_get(void);
uint8_t keyscan_wait(void);
bool    keyscan_get_button(uint8_t btn_mask);
```

---

### 2.7 font.c / font.h — ビットマップフォント（削除済み）

**削除済み**（177B削減）

**削除理由**: boot_main()でアプリ選択UI前に恵梨沙フォント（55KB）をFRAMに展開するため、英数字専用の5×5ビットマップフォントは不要。全ての文字描画は恵梨沙フォント（tft_draw_string_elysia / eink_draw_string_elysia）で統一。

---

### 2.8 load_resource.c — リソースロード機能

**実装状況**: 実装済み。LTO実寄与252B。RcApiテーブルにバインド済み。

任意のリソースをSPI FlashからFRAM作業領域に呼び出す機能。OSのページング機能に相当する。

```c
// res_type_id: [15:12]=種別(4bit) [11:0]=ID(12bit)
// fram_dest:   FRAM作業領域上の展開先アドレス
void load_resource(uint16_t res_type_id, uint32_t fram_dest_addr);
```

**処理フロー**:
```
1. res_type_idから種別を取り出す
2. FRAM作業領域のtype_start[種別]を参照 → 拡張ディレクトリのblock_cell取得
3. SPI Flashのディレクトリを線形サーチ → block_cell・sizeを取得
   （連続性保証により最大2ブロックで発見）
4. SPI Flashの対象アドレスからfram_dest_addrにデータを転送
```

**重ね合わせはアプリの責務。** 共通プログラムはロード・転送までを担い合成には関与しない。

---

## 3. SPI Flash管理処理（FRAM固定領域 0x08000〜0x0FFFF・32KB）

FRAMの固定領域（0x08000〜0x0FFFF・32KB）に格納する独立したプログラム。boot_main()がGPIO2=HIGHを検出した場合にIAP経由で内蔵Flashに書き込まれ実行される（外部通信モード）。

### 3.1 役割

| 役割 | 内容 |
|------|------|
| PCとのデータ交換 | USB HID経由でSPI Flashの読み書き消去を行う |
| カタログ管理 | SPI Flash上のアプリカタログを更新する |
| FRAMキャッシュ更新 | カタログ変更後にFRAMの管理領域キャッシュを更新する |

### 3.2 SPI Flashカタログ設計（✅確定）

**レイアウト**:
```
0x000000  カタログテーブル（32KB固定）
0x008000  恵梨沙フォント（64KB固定・0x008000は定数として定義）
0x018000  システム予約領域（512KB）
0x098000  アプリ領域（約15.4MB・4KBブロック×3968本）
0xFFFFFF
```

**CatalogEntry**: セクション0.4参照。

**FRAMキャッシュ対象**: ヘッダ部26B×58エントリ=1,508Bのみ。block_bitmap（512B）はSPI Flash上のみ。

**ブロック管理**:
- 4KBブロック単位（W25Q128の消去最小単位=4KBセクタと一致）
- 書き込み時: 連続する空きブロックを探して配置（連続空きなければデフラグ後に配置）
- デフラグ: PC側ツールが担当・UIAPduinoは関与しない・最悪ケース約4分半（NOR Flash特性）
- キーボードからの書き込みはデフラグが必要な場合はエラー返却

**残PENDING**:
- HIDレポートフォーマットの確定（PC側ツールとの合意が必要）
- PC側書き込みツールの設計・実装

### 3.3 書き込み・デフラグ方針

NOR Flash（W25Q128）はNAND Flash（マイクロSD等）より書き込み・消去が遅い。
消去前に全バイトを0で書き込む必要があるアーキテクチャに起因する。

| 操作 | 単位時間 | 16MB最悪ケース |
|------|---------|--------------|
| セクタ消去（4KB） | 45ms | 約3分 |
| ページ書き込み（256B） | 0.7ms | 約46秒 |
| **合計（最悪）** | | **約4分半** |

実際のデフラグは断片化した部分のみが対象で**現実的には数十秒以内**。

### 3.4 アプリ内ブロックディレクトリ

プログラムブロック後半のメタ情報領域に格納。コンバータが自動生成。

**リソースエントリ構造体（8バイト固定）**:

```c
typedef struct {
    uint16_t res_type_id;   // [15:12]=リソース種別(4bit) [11:0]=リソースID(12bit)
                            // 先頭に置く理由：プログラムが種別でまず分類するため
    uint16_t block_cell;    // [15:4]=ブロックID(12bit) [3:0]=セルID(4bit)
    uint8_t  size[3];       // リソースサイズ（バイト・最大16MB）
    uint8_t  reserved;      // 予備
} ResourceEntry;  // 計8バイト
```

**リソース種別**:

| 値 | 内容 | 備考 |
|----|------|------|
| **0x0** | **プログラム** | **ディレクトリ#0専用・FRAM展開済みで最速アクセス** |
| 0x1 | 画像 | |
| 0x2 | 音声（BGM） | |
| 0x3 | 音声（効果音） | |
| 0x4 | 音声（ボイス） | |
| 0x5 | シナリオ | |
| 0x6〜0xD | 未定義（予備） | |
| 0xE | ディレクトリ拡張ブロック | |
| 0xF | FM音色定義拡張ブロック | |

**リソース種別開始アドレス表（type_start・32B）**:

メタ情報領域のオフセット0x1000に配置。16種別×2B=32B。
各種別のディレクトリが格納されている拡張ブロックの先頭block_cellを格納。

```c
uint16_t type_start[16];
// type_start[0x0] = プログラム（ディレクトリ#0先頭 → FRAM展開済み）
// type_start[0x1] = 画像ディレクトリの先頭block_cell
// 未使用: 0xFFFF
```

**メタ情報領域レイアウト**:

| オフセット | サイズ | 内容 |
|-----------|--------|------|
| 0x0000 | 4KB | アプリ内ブロックディレクトリ#0（512エントリ×8B・プログラム専用） |
| 0x1000 | 32B | リソース種別開始アドレス表（type_start[16]） |
| 0x1020 | 約1KB | FM音色定義（48音色・拡張可） |
| 0x1420 | 512B | シナリオ継続情報（SCN_PTR・フラグ群） |
| 0x1620 | 128B | アプリメタ情報 |
| 0x16A0 | 約352B | 予備 |
| 0x1800〜 | 2KB | 予備（e-ink機種のみ） |

### 3.5 USB HID通信仕様

UIAPduinoのrv003usb USB HID機能を使用。ドライバ不要でWindows/Linux/macOS/Androidから接続可能。

**通信フォーマット（HIDレポート 64バイト固定）**（PENDING: PC側ツールとの合意が必要）:

```c
typedef struct {
    uint8_t  cmd;        // コマンドID
    uint8_t  seq;        // シーケンス番号
    uint32_t addr;       // SPI Flashアドレス
    uint16_t length;     // データ長
    uint8_t  data[56];   // データペイロード
} HidReport;             // 計64バイト

#define CMD_FLASH_READ    0x01  // SPI Flashから読み出し
#define CMD_FLASH_WRITE   0x02  // SPI Flashへ書き込み
#define CMD_FLASH_ERASE   0x03  // SPI Flashセクタ消去（4KB単位）
#define CMD_CATALOG_UPDATE 0x04 // カタログ更新後のFRAMキャッシュ更新
```

### 3.6 SPI Flash（W25Q128JV）基本コマンド

| コマンド | 値 | 説明 |
|---------|-----|------|
| READ | 0x03 | データ読み出し（最大24MHz） |
| WRITE | 0x02 | ページプログラム（256B単位） |
| WREN | 0x06 | 書き込み許可 |
| SECTOR ERASE | 0x20 | セクタ消去（4KB） |
| CHIP ERASE | 0xC7 | 全消去 |
| READ STATUS | 0x05 | ステータス読み出し（BUSY確認） |

---

## 4. 恵梨沙フォントのロード

### 4.1 格納場所
- **SPI Flash**: 固定アドレス `0x008000` に常駐（確定）
- 実データ: 6877文字 × 8バイト = 55,016バイト（≈55KB）
- 64KB枠（0x008000〜0x017FFF）に格納。残り9KBはパディング

```c
#define FONT_FLASH_ADDR  0x008000  // SPI Flash上の固定アドレス（確定）
#define FONT_FRAM_ADDR   0x13836   // FRAM作業領域内の展開先（コンテキストスタック直後）
```

### 4.2 ロードタイミング

**ブートローダ起動時**（アプリ起動前）にSPI FlashからFRAM作業領域へ転送する。
これにより、アプリタイトル選択画面から日本語タイトルの表示が可能になる。
アプリ開発者がフォントロード処理を書く必要はない。

```c
static void font_load_from_flash(void) {
    // SPI Flash 0x008000 → FRAM FONT_FRAM_ADDR（0x13836）へ55KB転送
    // 転送時間: 約18ms（SPI 24MHz）
    uint8_t chunk_buf[1024];
    uint32_t remaining = 55 * 1024;
    uint32_t src = FONT_FLASH_ADDR;
    uint32_t dst = FONT_FRAM_ADDR;
    while (remaining > 0) {
        uint16_t sz = (remaining > sizeof(chunk_buf)) ? sizeof(chunk_buf) : remaining;
        flash_rd_seq(src, chunk_buf, sz);
        fram_wr_seq(dst, chunk_buf, sz);
        src += sz; dst += sz; remaining -= sz;
    }
}
```

### 4.3 自己復元機能（UIAPduino交換後の復元）

UIAPduinoを新品に交換した場合、FRAMの共通プログラム領域（0x06000〜0x07FFF・8KB）からApp Areaへ自動IAPを実行する（自己IAP）。

```
UIAPduino交換後の起動:
    ↓
boot.c: App Area先頭 = 0xFFFFFFFF を検出
    ↓
FRAM 0x06000〜0x07FFF（共通プログラムバックアップ）を読み込み
    ↓
RAM展開型IAPで App Areaに書き込む
    ↓
ソフトウェアリセット → 共通プログラムが正常起動
```

---

## 5. IAPブロック切り替え

IAPによるブロック切り替えには2つのパターンがある。

| 項目 | **パターンA：IAPコンテキストスイッチ** | **パターンB：IAPソフトリセット** |
|------|--------------------------------|-------------------------------|
| 別名 | — | 1ブロック完結型IAP駆動 |
| SRAM退避 | する（2KB全退避・FRAMへ） | **しない** |
| 復帰 | する（呼び出し元へ戻る） | **しない** |
| 状態継続 | caller_flash_addrからApp Area・メタ情報を復元 | **FRAMに明示保存→再起動後参照** |
| 実装複雑度 | 高（アセンブリスタブ必要） | **低（iap_runをそのまま呼ぶだけ）** |
| 適した用途 | 同一アプリ内モジュール呼び出し・別アプリ呼び出し | **ノベルゲーム等・FRAM状態管理アプリ** |

---

### 5A. パターンB：IAPソフトリセット（1ブロック完結型）

ノベルゲーム等、数MBに及ぶシナリオをPC側で「1ブロック＝最大10KB以下」のバイナリに分割し、ブロック末尾でiap_run()を呼び出してFlashを書き換え、ソフトウェアリセット後に続きから再開する方式。

```c
// アプリ（App Area内・10KB）
// バイトコード評価ループ実行中
//     ↓
// CMD_BLOCK_NEXT [SPI_FLASH_ADDR] 検知

// 1. 継続に必要な情報をFRAMに保存（アプリが責任を持つ）
fram_write(SCN_PTR_ADDR, next_flash_addr);
fram_write(ENV_STATE_ADDR, env_state);
fram_write(FLAGS_ADDR, flags, FLAGS_SIZE);

// 2. iap_run()を呼び出す（戻らない）
iap_run(next_flash_addr);

// 3. 再起動後のアプリ初期化処理
uint32_t scn_ptr = fram_read(SCN_PTR_ADDR);
if (scn_ptr != 0xFFFFFFFF) {
    interpreter_resume(scn_ptr);  // 前回の続きから再開
} else {
    interpreter_start();          // 新規起動
}
```

---

### 5B. パターンA：IAPコンテキストスイッチ（モジュール間呼び出し機能）

### 5B.1 設計の背景と目的

内蔵Flash 16KBの制約上、大規模なアプリは複数のモジュールに分割してSPI Flash上に配置する必要がある。FRAMのSRAM相当作業領域にSRAMイメージと実行コンテキストを退避することで、IAPによる切り替えをコンテキストスイッチとして実装できる。

**コンテキストスイッチの2パターン**:
- **パターンA-1**: 同一アプリ内の別モジュール呼び出し（メタ情報は同一・維持）
- **パターンA-2**: 別アプリの呼び出し（メタ情報は別・iap_return()時にcaller_flash_addrから復元）

**重要な制約**: コンテキストスイッチ中はオーディオ・アニメーション等が途絶えるため、**場面転換・シーン切り替えなど無音のタイミングに限定して使用する。**

### 5B.2 コンテキストスタック構造

```c
typedef struct {
    uint8_t  restore_flag;
    uint8_t  module_id;
    uint32_t sp;
    uint32_t ra;
    uint32_t caller_flash_addr;  // iap_return()でApp Area・メタ情報を復元するために必要
    uint8_t  sram_image[2048];
} ContextEntry;  // 計2,062B

// 最大7段: 2,062B × 7 + ヘッダ16B = 14,450B ≈ 14.5KB
```

### 5B.3 所要時間の推定

| フェーズ | 所要時間 |
|---------|---------|
| 内蔵SRAM 2KB → FRAM退避 | 0.7ms |
| SPI Flashからモジュール読み出し（16KB） | 5.5ms |
| **内蔵Flash消去（最大のボトルネック）** | **80ms** |
| 内蔵Flash書き込み | 26ms |
| FRAMからSRAMイメージ復元 | 0.7ms |
| **合計（標準）** | **≈117ms** |
| **合計（最悪ケース）** | **≈290ms** |

### 5B.4 iap.cへの追加実装

セクション2.5のiap_call()/iap_return()の実装詳細参照。

```c
void iap_call(uint32_t flash_addr, uint8_t module_id, uint8_t call_type);
void iap_return(void);
uint8_t iap_stack_depth(void);
```

### 5B.5 アセンブリスタブの実装（重要）

セクション2.5bのiap_ctx.S参照。詳細はbootloader_dev_spec.mdのセクション5B.5参照。

---

## 6. ビルド環境

### 6.1 開発環境

| 項目 | 内容 |
|------|------|
| SDK | ch32v003fun（cnlohr/ch32fun） |
| コンパイラ | riscv32-unknown-elf-gcc |
| 最適化 | -Os（サイズ優先） |
| アーキテクチャ | -march=rv32ec -mabi=ilp32e |
| 書き込みツール | minichlink（ch32fun付属） |

### 6.2 コンパイル制約

| 制約 | 内容 |
|------|------|
| Flash使用量 | 両機種統一: UIAPduino込み7KB以内（共通プログラム5KB枠） |
| RAM使用量 | 内蔵SRAM 2KB以内（.data + .bss + スタック） |
| HAL不使用 | レジスタ直叩きのみ（ch32funのHALは不使用） |
| .ram_func | Flash書き換えルーチンはSRAMで実行 |

### 6.3 リンカスクリプト

```
MEMORY {
    /* 両機種統一 */
    FLASH (rx) : ORIGIN = 0x00000800, LENGTH = 5K
    RAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 2K
}
```

---

## 7. 実装順序（推奨）

| Step | モジュール | 確認方法 |
|------|-----------|---------| 
| 1 | spi_manager.c | FRAM・SPI Flashへの単純Read/Writeが通るか確認 |
| 2 | keyscan.c | I2CでMCP23017のボタン状態が読めるか確認 |
| 3 | tft_oled.c | 初期化→文字列1行表示が動くか確認（TFT ST7789から着手） |
| 3b | eink.c | e-ink調達後に実装（V2白黒→(G)4色の順） |
| 4 | app_loader.c | FRAMキャッシュからアプリ一覧が読めてTFTに表示できるか確認 |
| 5 | iap.c | SPI FlashのブロックをApp Areaに焼いて再起動できるか確認 |
| 6 | boot.c | ボタン判定→app_loader or App Area起動の分岐が正しく動くか確認 |
| 7 | SPI Flash管理処理 | USB HID経由でPCからSPI Flashへ書き込めるか確認 |
| 8 | font_load_elysia() | 恵梨沙フォントがFRAM作業領域に展開されTFTで日本語が表示できるか確認 |
| 9 | load_resource() | リソースIDを指定してSPI FlashからFRAM作業領域にロードできるか確認 |

---

## 8. アプリ向け機能の設計方針

共通プログラム（起動シーケンス用）とは別に、アプリが利用できる機能を3層で提供する。

### 8.1 機能の3層構造

| 層 | 提供形態 | 呼び出し方 | 用途 |
|----|---------|-----------|------|
| **1層** | 共通プログラム（FRAM固定） | 直接呼び出し | 表示最小・キースキャン最小・リソースロード |
| **2層** | SPI Flash上の共通モジュール | IAPコンテキストスイッチ | 機能リッチな表示等 |
| **3層** | アプリのブロックに組み込み | 直接実装 | 独自機能・オーディオ等 |

**2層の使用制約**: コンテキストスイッチ中はオーディオ・アニメーションが途絶えるため、場面転換・シーン切り替えなど無音のタイミングに限定する。

### 8.2 オーディオ機能の位置づけ

コンテキストスイッチ中は音声が途絶えるため、**オーディオ機能は各アプリのブロックに組み込む（3層）。**

App Area（9KB・両機種統一）内にアプリ本体と合わせて収める（〜1KB目標）。

### 8.3 キースキャン機能

| 層 | 対象デバイス |
|----|------------|
| 共通プログラム（1層・最小） | ゲームパッド・ジョイスティック（I2C） |
| アプリ組み込み（3層・リッチ） | キーボード等・拡張コネクタ接続デバイス |

### 8.4 ノベルゲームインタプリタ

バイトコード方式を採用する方向で詳細は未決定。以下を参考として設計する。

| 参考事例 | 特徴 |
|---------|------|
| **GB Studio** | 超軽量バイトコード・組み込み制約との親和性が高い |
| **NScripter** | バイトコードコンパイル済み・日本語ノベル標準 |
| **Ren'Py** | Pythonベース・スクリプト構文の参考 |
| **吉里吉里KAG** | タグベース・HTMLライク |

---

## 8a. アプリ向けAPI公開方針（確定）

### 公開範囲

| カテゴリ | 公開先 | 理由 |
|---------|--------|------|
| RcApiテーブル（12関数） | 全アプリ向け公開 | 機種非依存の標準API・固定アドレス0x08A0 |
| FRAM低レベルI/O（fram_write等） | **共通プログラム内部専用** | アプリによる誤書き込み・破壊を防ぐ |
| SPI Flash低レベルI/O（flash_read等） | **共通プログラム内部専用** | カタログ・フォント領域の保護 |
| セーブデータAPI | **2層ライブラリとして提供** | FRAM保護領域（0x02000〜0x05FFF）に限定 |

### セーブデータAPI（2層ライブラリ・将来実装）

アプリ開発者向けのセーブデータAPIはFRAMのセーブ領域のみにアクセスを限定した2層ライブラリとして提供する。共通プログラムのサイズに影響しない。

```c
// savedata.h（2層ライブラリ）
// FRAM セーブデータ領域: 0x02000〜0x05FFF（16KB）
bool save_write(uint8_t slot, const void *data, uint16_t size);
bool save_read(uint8_t slot, void *data, uint16_t size);
bool save_exists(uint8_t slot);
void save_erase(uint8_t slot);
```

### iap.hの低レベル関数の扱い

iap.hで公開されるFRAM/Flash低レベル関数（fram_write・fram_read・flash_read等）は共通プログラム内部のモジュール間共有専用とする。アプリ向けのヘッダには含めない。

## 8b. ディスプレイ統一API（RcApi・旧LtcApi）

### 設計確定内容

アプリ（3層）から共通プログラム（1層）の機能を呼び出すための固定アドレスAPIテーブル。
詳細設計はclaude_code_instructions_v1.2.mdのTask E参照。

**色定数（uint16_t統一）:**
```c
typedef uint16_t DispColor;
#define DISP_BLACK  0x0000U  // 全機種: 黒
#define DISP_WHITE  0xFFFFU  // 全機種: 白
#define DISP_RED    0xF800U  // TFT/カラーOLED: 赤 / e-ink4色: 赤 / 白黒: 黒寄り
#define DISP_GREEN  0x07E0U  // TFT/カラーOLED: 緑 / 白黒: 白寄り
#define DISP_BLUE   0x001FU  // TFT/カラーOLED: 青 / 白黒: 黒寄り
```

**固定アドレス:** 0x00000804（リンカで強制配置）

**テーブル構造:**
- 先頭: ディスプレイ非依存関数（iap_run・load_resource・keyscan_*）→ 機種共通アドレス
- 末尾: ディスプレイ依存関数（disp_init・disp_fill・disp_draw_string等）→ 機種ごとに異なる実装

**e-ink/OLEDでの色変換:** 輝度閾値（color > 0x7BEF）で白黒に変換（ドライバ内1行）

## 9. 未決事項

| # | 項目 | 内容 |
|---|------|------|
| 1 | USB HID VID/PID | rv003usbデフォルトを使うか専用に取得するか |
| 2 | HIDレポートフォーマット確定 | ✅確定：64B固定・cmd(1B)/seq(1B)/addr(4B)/length(2B)/data(56B)。USB Full Speed物理制約（64B上限）による |
| 3 | 恵梨沙フォントのAPIフォーマット | FRAM作業領域展開後のアクセス方法をアプリAPIとして定義 |
| 4 | キースキャンのデバイス・I2Cアドレス | ✅確定：seesawデバイス（ADA-5743）・I2Cアドレス0x50。keyscan.cのseesaw対応修正はPhase 3試験過程で実施 |
| 5 | SPI Flashカタログフォーマット | ✅解決：CatalogEntry構造体確定（セクション3.2参照）。HIDレポートフォーマット・PC側ツールはPENDING継続 |
| 6 | TFT・OLED版対応 | tft_oled.cは実装・サイズ計測済み（896B）。e-ink調達後にeink.cを実装 |
| 7 | 200ms waitの調整 | キーボードのGPIO2セットまでの実測時間に応じて調整が必要な場合がある |
| 8 | extern_comm_run()の実装 | FRAMの固定領域（0x08000〜）のSPI Flash管理処理をIAP経由で起動する仕組みの設計 |
| 9 | 恵梨沙フォントのSPI Flash上アドレス確定 | ✅解決：FONT_FLASH_ADDR = 0x008000（固定）に確定 |
| 10 | ノベルゲームシナリオフォーマット | バイトコード方式で詳細設計（PC側ツールと並行） |
| 11 | オーディオ再生機能の実装 | YMF825再生コマンド送信・PWM/ADPCM再生を〜1KBに収める設計 |
| 12 | YMF825初期化テンプレートブロック | 開発者向けテンプレートの作成 |
| 13 | CAR-02/03ブートシーケンス仕様書 | 別書として作成（nRF52840/ESP32-S3それぞれ） |
| 14 | アセンブリスタブの動作検証 | iap_call_entry/iap_return_entryをCH32V003実機で検証 |
| 15 | SRAMイメージ転送中の割り込み制御 | 転送中の割り込み無効化タイミングを実装・検証 |
| 17 | スマホ用アプリ開発 | iPhone向け専用iOSアプリ・Android向けWebHIDツール |
| 18 | XIAO版アプリ内ブロックディレクトリ | UIAPduino版との差異を別途設計 |
| 19 | iPhone USB HID 100mAディスクリプタ確認 | rv003usbのUSBディスクリプタがiPhone接続条件を満たすか検証 |
| 20 | UIAPduinoブートローダのジャンプ先確認 | 実機でタイムアウト後のジャンプ先が0x0800であることを確認 |
| 21 | セルフアップデート機能の実装 | SPI Flashシステム予約ブロックへのアップデートプログラム配置 |
| 22 | コンテキストスイッチ前後の情報受け渡し | ✅解決：FRAM固定領域4区画（A-1引数/戻り値・A-2引数/戻り値 各64B・0x16200〜0x162FF） |
| 23 | iap_call()のA-1/A-2判別方法 | ✅解決：第3引数call_type（IAP_CALL_INTERNAL=0/IAP_CALL_EXTERNAL=1）で明示指定 |
| 24 | iap_return()のApp Area書き戻し実装 | ✅実装済み：caller_flash_addrから8KB（App Area）+8KB（メタ情報）を復元 |
| 25 | load_resource()のサイズ確認 | ✅実装済み・サイズ確認済み（LTO実寄与252B）|
| 26 | 共通プログラム枠統一案 | ✅確定：7.5KB統一案・共通5,320B・UIAPduino込み7,368B・余裕312B |
| 27 | LtcApi→RcApiへの改名 | Latecaputer（誤綴り）→Ratecaputer Cartridge APIとして全ソース・仕様書のLtcApi/ltc_api.*をRcApi/rc_api.*に改名。7KB統一案確定後に実施 |
| 28 | Latecaputerの名残の確認・除去 | ソースコード・仕様書・フォルダ名等にLatecaputerの誤綴りが残っていないか確認して修正。27番と同時に実施 |
| 29 | アプリ開発環境の設計 | 以下を含む未検討課題：①文字コード（UTF-8/Shift-JIS）→恵梨沙フォントインデックス変換ツール（PC側）②恵梨沙フォントインデックス列を扱う開発フロー③エミュレーション・シミュレーションによる試験環境（実機なしでの開発・デバッグ手段）④アプリ開発者向けSDK・ドキュメントの整備方針 |
| 28 | Latecaputerの名残の確認・除去 | ソースコード・仕様書・フォルダ名等にLatecaputerの誤綴りが残っていないか確認して修正。27番と同時に実施 |
