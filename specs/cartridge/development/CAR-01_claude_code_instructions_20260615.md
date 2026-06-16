# フリスクカートリッジ CAR-01 共通プログラム実装指示書

**対象: Claude Code**
**目的: 共通プログラム（UIAPduino込み7.5KB以内）の実装完了・サイズ検証前残実装・サイズ確定**
**Rev. 1.1 — 2026年5月22日**

---

## ⚠️ 作業開始前に必ず実行すること

1. `docs/`配下の**全mdファイル**を読む（`cat docs/*.md`）
2. 読み終えた後、**作業計画を以下の形式で提示**する：
   - 実施する作業の一覧
   - 実施順序とその理由
   - 各作業の完了条件
3. ユーザーが「進めてください」と言ってから作業を開始する

**作業計画の確認なしに実装を始めてはいけない。**
**試験環境ツールより共通プログラム群の実装を先に完成させること。**
**仕様書に記載のない機能を独自に追加してはいけない。追加が必要と判断した場合はユーザーに確認すること。**

### 事前作業0: keyscan.cをseesawデバイス対応に修正

**今回の作業に含めること。省略不可。**
事前作業1〜3・Task D/E/Fと同じセッションで実装する。
実施順序は事前作業1〜3の後でよい。

**デバイス仕様:**
- I2Cアドレス: 0x50（seesaw標準）
- 接続: SDA=PC1 / SCL=PC2
- 2軸ジョイスティック + 6ボタン（大4つ・小2つ）
- seesawはMCP23017とレジスタ体系が全く異なるため再実装が必要

**seesawのボタン読み出し方法:**
```c
// seesawのGPIOバンク読み出しコマンド
// レジスタ: SEESAW_GPIO_BASE(0x01) + SEESAW_GPIO_BULK(0x04)
// 4バイト返却（GPIO状態のビットマップ）
// ボタンのピン番号はADA-5743のデータシートを参照すること
#define SEESAW_ADDR        0x50
#define SEESAW_GPIO_BASE   0x01
#define SEESAW_GPIO_BULK   0x04
```

実装前にAdafruit ADA-5743のデータシートおよびArduinoライブラリソースを
参照してピン番号とBTN_*マスクの対応を確認すること。

### 事前作業: 恵梨沙フォント描画関数の実装とfont.c削除

英数字ビットマップフォント（font.c・font.h・177B）は不要になった。
**ただし削除前に恵梨沙フォント描画関数を実装すること。**

**理由**: boot_main()内でアプリ選択UIより前に恵梨沙フォント（55KB）をSPI Flash 0x008000からFRAM 0x13836に転送する。アプリ選択画面（app_loader_run）は共通プログラム内で実行されるため、日本語タイトル表示も共通プログラム内の恵梨沙フォント描画関数が必要。

**実施順序:**

**Step 1: tft_oled.cに恵梨沙フォント描画関数を追加**

```c
// FRAM 0x13836 + font_idx × 8 から8Bを読んで1文字描画
// 恵梨沙フォント: 8×8px・1bit（1B=1行・MSBが左端）
void tft_draw_char_elysia(uint16_t font_idx,
                           uint16_t x, uint16_t y,
                           uint16_t fg, uint16_t bg);

void tft_draw_string_elysia(const uint16_t *indices, uint8_t len,
                              uint16_t x, uint16_t y,
                              uint16_t fg, uint16_t bg);
```

**Step 2: app_loader.cのタイトル表示を恵梨沙フォントに切り替え**

CatalogEntryのtitle[8]（uint16_t型・恵梨沙フォントインデックス列）を
tft_draw_string_elysia()で表示する。

**Step 3: font.c / font.hを削除**

```bash
rm common_prog/font.c common_prog/font.h
# Makefileのfont.cを削除
# tft_oled.cのfont.hインクルードを削除
```

Step 1〜3完了後にサイズ計測を行い177Bの解放を確認すること。

---

## 0. この指示書の読み方

本指示書はClaude Codeへの実装指示である。詳細な設計背景・判断理由は以下の目論見書を参照すること。

- `cartridge_spec_master.md` — カートリッジ共通規格・CAR-01個別仕様
- `bootloader_dev_spec.md` — 共通プログラム開発仕様書（Rev.2.0）

**本指示書の目的はサイズ計測のための仮実装である。** 各モジュールをコンパイルしてバイナリサイズを計測し、目標サイズ以内に収まることを確認する。サイズ超過の場合はその場で削減する。

---

## 1. ハードウェア前提

### MCU

```
CH32V003F4P6
  クロック: 48MHz（外部クリスタルなし・内蔵RC発振）
  内蔵Flash: 16KB（0x00000000〜0x00003FFF）
  内蔵SRAM:  2KB（0x20000000〜0x200007FF）
```

### SPIバス CS割り当て

```c
// 全デバイスで共通SPIバス（SPI1）を使用
// SCK=PC5 / MOSI=PC6 / MISO=PC7
#define CS_EINK   PC0   // e-ink ディスプレイ
#define CS_YMF    PD2   // YMF825 FM音源
#define CS_FLASH  PD3   // SPI Flash W25Q128JV（16MB）
#define CS_FRAM   PD4   // SPI FRAM（256KB or 512KB・1チップ）
// PD6: 未使用（解放済み）
```

### その他のピン割り当て

```c
#define DC_EINK      PC3   // e-ink DC
#define RST_EINK     PC4   // e-ink RST
#define BUSY_EINK    PD0   // e-ink BUSY
#define PWM_AUDIO    PD5   // ADPCM PWM出力
#define CART_READY   PA1   // 拡張コネクタ上列1・準備完了通知（OUTPUT）
#define GPIO2_PIN    PA2   // 拡張コネクタ上列2・KB_NOTIFY確認（INPUT）
#define I2C_SDA      PC1   // I2C SDA（ゲームパッド）
#define I2C_SCL      PC2   // I2C SCL（ゲームパッド）
```

### メモリアドレス定義

```c
// 内蔵Flash
#define COMMON_PROG_BASE  0x00000000  // 共通プログラム先頭
#define APP_AREA_EINK     0x00002000  // App Area（e-ink機種）
#define APP_AREA_TFT      0x00001800  // App Area（TFT・OLED機種）

// FRAM（共通・グレードによらず）
#define FRAM_CATALOG_CACHE 0x00000000  // カタログキャッシュ（8KB）
#define FRAM_SAVEDATA      0x00002000  // セーブデータ（16KB・0x00002000〜0x00005FFF）
#define FRAM_COMMON_PROG   0x00006000  // 共通プログラムバックアップ（8KB・0x00006000〜0x00007FFF）
#define FRAM_FLASH_MGR     0x00008000  // SPI Flash管理処理（32KB）
#define FRAM_WORK_BASE     0x00010000  // SRAM相当作業領域先頭

// FRAM作業領域内
#define FRAM_CTX_STACK     0x00010000  // コンテキストスタック（〜14KB）
#define FRAM_FONT_BASE     0x00013800  // 恵梨沙フォント展開領域（55KB）

// SPI Flash
#define FLASH_CATALOG_BASE 0x000000    // カタログテーブル（32KB）
#define FLASH_FONT_BASE    0x008000    // 恵梨沙フォント（64KB・固定）
#define FLASH_SYS_BASE     0x018000    // システム予約（512KB）
#define FLASH_APP_BASE     0x098000    // アプリ領域

// MCP23017 ゲームパッド（仮・実機確認後に変更）
#define KEYSCAN_I2C_ADDR   0x20        // MCP23017デフォルトアドレス
#define KEYSCAN_I2C_FREQ   400000      // 400kHz
```

---

## 2. ビルド環境

### 使用SDK

`ch32v003fun`（cnlohr/ch32fun）を使用する。

```bash
# リポジトリのクローン
git clone https://github.com/cnlohr/ch32fun.git

# ビルド確認
cd ch32fun/examples/blink
make
```

### コンパイラ

```
riscv32-unknown-elf-gcc
```

### Makefile（各モジュール共通）

```makefile
TARGET = common_prog
SRCS = spi_manager.c keyscan.c tft_oled.c iap.c app_loader.c boot.c
SRCS += font.c

CH32FUN_DIR = ../ch32fun
include $(CH32FUN_DIR)/ch32fun.mk

# サイズ確認用
size: $(TARGET).elf
	$(CROSS)size $(TARGET).elf
	$(CROSS)objdump -h $(TARGET).elf | grep -E "\.text|\.data|\.bss"
```

### サイズ計測コマンド

```bash
make size
# .text（コード）+ .rodata（定数）= Flash使用量
# 目標: UIAPduino込み7KB以内（両機種統一）
```

---

## 3. 実装するモジュール一覧

| モジュール | ファイル | サイズ目標 | 優先度 |
|-----------|---------|-----------|--------|
| SPIマネージャ | spi_manager.c / spi_manager.h | 0.5KB | 最高（全モジュールの基盤） |
| キースキャン | keyscan.c / keyscan.h | 1KB | 高 |
| TFT・OLEDドライバ | tft_oled.c / tft_oled.h | 1KB | 高（TFTから着手） |
| ビットマップフォント | font.h | 200B | 高 |
| RAM展開型IAP（パターンA） | iap.c / iap.h | 0.5KB | 高 |
| アプリロード処理 | app_loader.c / app_loader.h | 1KB | 高 |
| e-inkドライバ | eink.c / eink.h | 3KB | 中（調達後） |
| ブートローダ拡張 | boot.c | 2KB | 高（最後に統合） |
| IAPコンテキストスイッチ | iap_ctx.S + iap.c追記 | 計測対象 | 高（最後に本実装・必須） |

**実装順序: spi_manager → keyscan → tft_oled → font → iap（パターンA） → app_loader → boot → iap_ctx（コンテキストスイッチ本実装）**
**eink.cはe-ink調達後に実装する。**
**IAPコンテキストスイッチは最後に本実装する。スタブで代替しない。全機能込みで8KBに収まることを確認するのが本指示書の目的。**

---

## 4. 各モジュールの実装仕様

### 4.1 spi_manager.c

**役割**: ハードウェアSPIの直接制御とCS信号の排他管理。HALを使わずレジスタを直接操作する。

**サイズ目標**: 0.5KB以内

```c
// spi_manager.h

typedef enum {
    SPI_DEV_EINK  = 0,  // CS: PC0
    SPI_DEV_YMF   = 1,  // CS: PD2
    SPI_DEV_FLASH = 2,  // CS: PD3
    SPI_DEV_FRAM  = 3,  // CS: PD4
} SpiDevice;

// SPI速度設定（デバイスごとに切り替え）
// FRAM: 40MHz → CH32V003上限24MHzで動作
// SPI Flash: 24MHz
// e-ink: 4MHz
// YMF825: 10MHz

void spi_manager_init(void);
void spi_cs_select(SpiDevice dev);    // CS Low + 速度切り替え
void spi_cs_deselect(SpiDevice dev);  // CS High
uint8_t spi_transfer(uint8_t data);
void spi_write_buf(const uint8_t *buf, uint16_t len);
void spi_read_buf(uint8_t *buf, uint16_t len);
```

**実装上の注意**:
- ch32funのSPI1レジスタを直接操作すること（HAL不使用）
- CS切り替え時にSPIクロック分周比を変更する（デバイスにより最大速度が異なる）
- 同時アクセス防止のためCS操作は必ずselect/deselect対で使うこと

---

### 4.2 keyscan.c

**役割**: MCP23017（I2C接続）からボタン入力を読み取る。起動時のボタン確認に使用。

**サイズ目標**: 1KB以内

**I2Cアドレス**: 0x20（仮・A0/A1/A2全てGND想定）

```c
// keyscan.h

// ボタンマスク定義
#define BTN_UP     0x01
#define BTN_DOWN   0x02
#define BTN_LEFT   0x04
#define BTN_RIGHT  0x08
#define BTN_OK     0x10
#define BTN_CANCEL 0x20

void    keyscan_init(void);
uint8_t keyscan_get(void);                    // 現在のボタン状態（ポーリング）
uint8_t keyscan_wait(void);                   // ボタン押下を待機して返す
bool    keyscan_get_button(uint8_t btn_mask); // 特定ボタンが押されているか
```

**実装上の注意**:
- MCP23017のGPIOA（8bit）をINPUTとして設定する
- I2C 400kHz（Fast mode）で動作させること
- ch32funのI2Cレジスタ直接操作を使用すること
- デバウンス処理は最小限（1〜2ms待機程度）

---

### 4.3 tft_oled.c

**役割**: ST7789（1.3inch IPS 240×240px）の最小ドライバ。アプリタイトル一覧表示に使用。

**サイズ目標**: 1KB以内

```c
// tft_oled.h

void    tft_init(void);
void    tft_fill(uint16_t color);                // 画面塗りつぶし
void    tft_draw_char(uint8_t c, uint16_t x, uint16_t y, uint16_t fg, uint16_t bg);
void    tft_draw_string(const char *str, uint16_t x, uint16_t y, uint16_t fg, uint16_t bg);
void    tft_set_window(uint16_t x0, uint16_t y0, uint16_t x1, uint16_t y1);
void    tft_write_pixel(uint16_t color);
```

**実装上の注意**:
- ST7789の初期化シーケンスを最小限に絞ること（サイズ優先）
- 色フォーマット: RGB565（16bit）
- SPI経由（CS_EINK兼用ではなく専用CSピンを使用すること→現状のCAR-01ではe-inkとTFTは排他なので同じCS_EINKピンを使用してよい）
- font.hの5×5ビットマップフォントを使って文字描画を実装する

---

### 4.4 font.h

**役割**: 起動シーケンス内の英数文字表示用ビットマップフォント。

**サイズ目標**: 200B以内

```c
// font.h

// 5×5ピクセル・40文字（A-Z・0-9・スペース・ピリオド・記号2文字）
// 1文字あたり5バイト（各バイトが1列・上位3bitは未使用）
// 文字インデックス: 0=スペース 1=. 2=記号1 3=記号2
//                  4〜29=A-Z  30〜39=0-9

#define FONT_CHAR_WIDTH  5
#define FONT_CHAR_HEIGHT 5
#define FONT_CHAR_COUNT  40

extern const uint8_t font_5x5[FONT_CHAR_COUNT][5];

// 文字コード→フォントインデックス変換
static inline int font_index(char c) {
    if (c == ' ') return 0;
    if (c == '.') return 1;
    if (c >= 'A' && c <= 'Z') return c - 'A' + 4;
    if (c >= 'a' && c <= 'z') return c - 'a' + 4;  // 小文字→大文字扱い
    if (c >= '0' && c <= '9') return c - '0' + 30;
    return 0;  // 未対応文字はスペース
}
```

**実装上の注意**:
- `const uint8_t font_5x5[40][5]` を `.rodata` セクションに配置すること
- 実際のフォントデータは読みやすい標準的な5×5ドットフォントで実装すること

---

### 4.5 iap.c

**役割**: RAM展開型IAPの実装。Flash書き換えをSRAM上で実行することで、実行中のFlashを安全に書き換える。

**サイズ目標**: 0.5KB以内（SRAM実行部は1.5KB以内）

```c
// iap.h

// パターンA: ソフトリセット型IAP（戻らない）
// flash_addr: SPI Flash上のプログラムブロック先頭アドレス
void iap_run(uint32_t flash_addr);

// パターンA拡張: コンテキストスイッチ型IAP（本実装・最後に追加）
// アセンブリスタブ（iap_ctx.S）が必要
// call_typeを第3引数として追加（確定）
#define IAP_CALL_INTERNAL  0  // A-1: 同一アプリ内モジュール
#define IAP_CALL_EXTERNAL  1  // A-2: 別アプリ

// 情報受け渡し領域（FRAM・メタ情報領域内に配置）
#define IAP_ARG_INTERNAL   0x00016680UL  // A-1引数（64B）
#define IAP_RET_INTERNAL   0x000166C0UL  // A-1戻り値（64B）
#define IAP_ARG_EXTERNAL   0x00016700UL  // A-2引数（64B）
#define IAP_RET_EXTERNAL   0x00016740UL  // A-2戻り値（64B）

void iap_call(uint32_t flash_addr, uint8_t module_id, uint8_t call_type);
void iap_return(void);
uint8_t iap_stack_depth(void);
```

**iap_run()の処理フロー**:

```
1. SPI Flashの指定アドレスからプログラムブロック（16KB）をFRAM作業領域に読み込み
   - 先頭9KB（両機種共通・App Area統一）→ 内蔵SRAMの作業バッファへ
   - 残り領域（メタ情報）→ FRAM作業領域 FRAM_WORK_BASE へ転送
2. Flash書き換えルーチン本体を内蔵SRAM（2KB）の後半にコピー
3. 内蔵SRAMのルーチンに制御を渡す
4. [SRAM上で実行]
   App Area消去（両機種統一: 0x1C00〜0x3FFF・9KB）
   App Areaに書き込む
5. ソフトウェアリセット（PFIC_SYSRESETRQ）
```

**実装上の注意**:
- Flash書き換えルーチンは必ず `__attribute__((section(".ram_func")))` でSRAMに配置すること
- 書き換え中はSPIアクセス禁止（CS全てHigh固定）
- `iap_call` / `iap_return` は最後に本実装する（アセンブリスタブ iap_ctx.S が必要。bootloader_dev_spec.md セクション5B.5参照）
- 参考: WCH公式 `ch32v003/EVT/EXAM/IAP` サンプルコード
- **SRAMバッファアドレスは必ず `.ram_func` セクションの末尾より後ろにすること。**
  `.ram_func` は `0x20000000` から配置されるため、`0x20000000` をバッファ先頭にすると
  `flash_read()` が `ram_flash_write()` を上書きしてしまう。
  ビルド後にmapファイルで `.ram_func` の末尾アドレスを確認し、その直後（4Bアライン）を
  SRAMバッファの先頭アドレスとして使用すること。
  例: `.ram_func` が `0x20000000〜0x200000AA` なら `sram_buf = 0x200000B0`

---

### 4.5b iap_ctx.S — IAPコンテキストスイッチ（本実装）

**役割**: iap_call()呼び出し時に内蔵SRAMイメージ・SP・RAをFRAMに退避し、iap_return()で復元する。CからはSP・RAを正確に取得できないためアセンブリで実装する。

**詳細仕様**: `bootloader_dev_spec.md` セクション5B.5を必ず参照すること。

概要：

```asm
# iap_ctx.S (RISC-V RV32I)
.section .text
.global iap_call_entry
iap_call_entry:
    mv   t0, ra          # 呼び出し元の戻りアドレス（ABI確立前に取得）
    mv   t1, sp          # 呼び出し元のSP（ABI確立前に取得）
    mv   a2, a1          # caller_module_id
    mv   a1, t0          # ra
    mv   a0, t1          # sp
    call sram_save_to_fram   # C関数でFRAMへSRAMイメージ・SP・RAを保存
    call iap_run_internal    # Flash書き換え→リセット（戻らない）

.global iap_return_entry
iap_return_entry:
    call sram_restore_from_fram  # FRAMからSRAMイメージを復元・a0=sp・a1=ra
    mv   sp, a0
    jr   a1              # 呼び出し元へ復帰
```

C側の `sram_save_to_fram()` / `sram_restore_from_fram()` はiap.cに実装する。

**実装上の注意**:
- `sram_save_to_fram()` に `__attribute__((noinline))` を付けること
- FRAM書き込み中は割り込みを無効化すること（`__disable_irq()` / `__enable_irq()`）
- コンテキストスタック最大7段。8段目はエラー処理（無限ループ等）
- コンテキストスタックのアドレス: `FRAM_CTX_STACK`（0x00010000）
- 1コンテキスト = 2,058バイト（SRAMイメージ2KB + 復帰情報58B）

### 4.6 app_loader.c

**役割**: FRAMのカタログキャッシュからアプリ一覧を取得し、ディスプレイに表示してユーザーに選択させ、選択されたアプリをIAPで書き込む。

**サイズ目標**: 1KB以内

```c
// app_loader.h

// カタログエントリ構造体（SPI Flashカタログと同一・FRAMにキャッシュ済み）
typedef struct {
    uint8_t  valid;              // 0xFF=空き / 0x01=有効 / 0x00=削除済み
    uint8_t  disp_type;          // 0x01=e-ink / 0x02=TFT・OLED（App Area 9KB・両機種統一）
    uint16_t title[8];           // 恵梨沙フォントインデックス列（最大8文字）
    uint32_t flash_addr;         // SPI Flash上の先頭アドレス
    uint32_t total_size;         // 全体サイズ
    uint8_t  block_bitmap[512];  // 4096ブロック×1bit
} CatalogEntry;  // 計538バイト

// FRAMカタログキャッシュからアプリ一覧を読み込む
// 戻り値: 有効エントリ数
int  app_loader_read_catalog(CatalogEntry *entries, int max_count);

// アプリ選択UIを実行してIAPを呼び出す（戻らない）
void app_loader_run(void);

// App Areaにアプリが存在するか確認
bool app_area_has_program(void);

// App Areaへジャンプ（戻らない）
void app_loader_jump_to_app(void);
```

**app_loader_run()の処理フロー**:

```
1. FRAMカタログキャッシュ（FRAM_CATALOG_CACHE）からエントリを読み込む
2. valid=0x01のエントリのみ抽出してタイトル一覧を作成
3. tft_draw_string()でタイトルを一覧表示（titleフィールドは英数8文字として扱う）
   ※ 恵梨沙フォントによる日本語表示は後で対応。現時点はASCIIタイトルのみ
4. keyscan_wait()でボタン入力を待機
   - BTN_UP/BTN_DOWN: 選択項目を変更・再描画
   - BTN_OK: 選択確定 → iap_run(entry.flash_addr)を呼び出す（戻らない）
```

**実装上の注意**:
- FRAMからCatalogEntryを読む際、block_bitmap(512B)は読み飛ばしてよい（タイトル表示に不要）
- 現時点はtitleフィールドの上位バイト（uint8_t相当）をASCIIとして扱う簡易実装でよい
- 日本語タイトル表示（恵梨沙フォント使用）は後続フェーズで実装

---

### 4.7 boot.c

**役割**: ブートローダ拡張。UIAPduinoオリジナルブートローダがApp Areaとして本モジュールを呼び出す。

**サイズ目標**: 2KB以内

```c
// boot.cの処理フロー（疑似コード）

void boot_main(void) {
    // 1. ハードウェア初期化
    spi_manager_init();
    tft_init();           // TFT機種の場合（e-ink機種はeink_init()）
    keyscan_init();

    // 2. 200ms待機
    // 理由: カートリッジ電源ONをキーボードが即座に検出できないため
    //       この間にキーボードがGPIO2（PA2）をHIGHにセットできる
    delay_ms(200);

    // 3. GPIO2（PA2）確認 → 外部通信モード or 通常起動
    if (gpio_read(GPIO2_PIN) == HIGH) {
        extern_comm_run();  // スタブ実装: while(1)
        // ここには戻らない
    }

    // 4. 恵梨沙フォントをSPI Flash→FRAM作業領域へ転送
    // SPI Flash 0x008000（55KB）→ FRAM_FONT_BASE（0x13800）
    // 転送時間: 約18ms（SPI 24MHz）
    font_load_from_flash();

    // 5. CART_READYをHIGHにしてキーボードに準備完了を通知
    gpio_set_high(CART_READY);

    // 6. キースキャン（決定ボタン確認）
    bool btn_pressed = keyscan_get_button(BTN_OK);

    if (!btn_pressed) {
        // 7a. App Areaにアプリが存在するか確認
        if (app_area_has_program()) {
            app_loader_jump_to_app();  // App Areaへジャンプ（戻らない）
        }
        // アプリなし → 7bへフォールスルー
    }

    // 7b. アプリロード処理
    app_loader_run();  // 戻らない
}

// スタブ実装
static void extern_comm_run(void) {
    gpio_set_high(CART_READY);  // 外部通信モードでもCART_READYはHIGHにする
    while(1);  // TODO: SPI Flash管理処理をIAP経由で起動
}

// 恵梨沙フォント転送
static void font_load_from_flash(void) {
    // SPI Flash 0x008000 → FRAM FRAM_FONT_BASE（0x13800）へ55KB転送
    // spi_manager.cのread/write関数を使って実装
}

// App Area先頭4バイトが0xFFFFFFFFでなければアプリありと判定
static bool app_area_has_program_check(void) {
    uint32_t *app_start = (uint32_t*)APP_AREA_EINK;  // or APP_AREA_TFT
    return (*app_start != 0xFFFFFFFF);
}
```

**自己復元機能**:
App Area未書き込み（0xFFFFFFFF）かつGPIO2=LOWの場合、FRAMの共通プログラムバックアップ領域（FRAM_COMMON_PROG=0x06000）からApp Areaへ自動IAPする。現時点はスタブ（単純にapp_loader_run()へフォールスルー）でよい。

---

## 5. サイズ計測手順

### 各モジュール単体でのサイズ確認

```bash
# 例: spi_manager.c単体のオブジェクトサイズ確認
riscv32-unknown-elf-gcc -Os -march=rv32ec -mabi=ilp32e \
  -c spi_manager.c -o spi_manager.o
riscv32-unknown-elf-size spi_manager.o
```

### 全体リンク後のサイズ確認

```bash
make size
# 出力例:
#    text    data     bss     dec     hex filename
#    5832       0      12    5844    16d4 common_prog.elf
# → text+data = 5832B = 5.7KB (TFT機種6KB目標以内 ✅)
```

### サイズ超過時の対応方針

1. コンパイラオプション確認: `-Os`（サイズ最適化）が有効か
2. 不要な初期化シーケンスの削除
3. 文字列リテラルの削減
4. 関数のインライン化抑制: `__attribute__((noinline))`
5. 定数を`.rodata`に明示配置

---

## 6. 完了条件

以下を全て確認できれば仮実装完了とする。

| 確認項目 | 目標 |
|---------|------|
| Flash使用量（共通プログラム単体） | 5,632B（5.5KB）以内 |
| UIAPduino込み合計 | 7,680B（7.5KB）以内 |
| spi_manager.c単体 | 0.5KB以内 |
| keyscan.c単体 | 1KB以内 |
| tft_oled.c単体 | 1KB以内 |
| font.h（rodata） | 200B以内 |
| iap.c単体 | 0.5KB以内 |
| app_loader.c単体 | 1KB以内 |
| boot.c単体 | 2KB以内 |
| IAPコンテキストスイッチ込みのFlash使用量 | UIAPduino込み7KB以内 |
| コンパイルエラーなし | ✅ |
| 警告0件（または説明付き） | ✅ |

---

## 6b. Phase 3 追加タスク（Phase 2完了後に着手）

### Task A: spi_flash_mgr.c のHID通信実装

UIAPduino公式のVID/PIDを使ってHID通信を実装する。

```c
#define USB_VENDOR_ID   0x1209  // pid.codes UIAPduino公式
#define USB_PRODUCT_ID  0xB803
```

1. `bootloader_custom/bootloader/` 配下のrv003usbソースでHID設定方法を確認する
2. spi_flash_mgr.cのhid_recv()/hid_send()スタブを実装する
3. PC側ツール（tools/write_program.py等）にVID/PIDを反映する

```python
import hid
device = hid.device()
device.open(0x1209, 0xB803)
```

**注意**: rv003usbはビットバングUSBのため通常のHIDライブラリと異なる場合がある。ソースを必ず確認してから実装すること。

### Task B: test_appから共通プログラム関数の呼び出し

**関数ポインタテーブル（APIジャンプテーブル）は実装しない。** サイズ影響が大きく（現在残り約100B）仕様書にも記載がない。

現段階ではmapファイルで関数アドレスを確認し直接アドレス指定で呼び出す方法を検討する。実装前にユーザーに方針を確認すること。

### Task C: サイズ確認

Phase 3実装完了後に必ずサイズ計測を行い、UIAPduino込み7KB以内（両機種統一）に収まることを確認する。

---

## 7. 注意事項

- **eink.cは今回実装しない**。e-ink調達後に別途実装する。
- **IAPコンテキストスイッチ（iap_call/iap_return）は最後に本実装する**。アセンブリスタブ（iap_ctx.S）が必要。bootloader_dev_spec.mdのセクション5B.5を参照。スタブで代替しない。
- **extern_comm_run()はスタブ**（while(1)）。SPI Flash管理処理完成後に本実装する。
- **キースキャンI2Cアドレス0x20は仮決定**。実機確認後に変更する可能性がある。
- **恵梨沙フォントによる日本語タイトル表示は後続フェーズ**。現時点はASCIIのみ対応。
- ch32funのサンプルコード（`ch32fun/examples/`）を積極的に参考にすること。特にSPI・I2Cの直接レジスタ操作部分。

---

## 8b. Phase 2b 追加タスク（eink.c実装とAPIテーブル設計）

### 前提：メモリマップ統一案

両機種（TFT・OLED / e-ink）を以下の統一レイアウトで実現する。

```
0x0000 ┌─────────────────────────┐
       │ UIAPduinoブートローダ   │ 2KB枠（計算基準・実測1,892B）
0x0800 ├─────────────────────────┤
       │ ラテカ共通プログラム    │ 5KB枠（0x0800〜0x1BFF）
       │ UIAPduino2KB込み合計    │ 7KB（両機種統一）
0x1C00 ├─────────────────────────┤
       │ App Area（3層）         │ 9KB（0x1C00〜0x3FFF）
0x3FFF └─────────────────────────┘
```

リンカスクリプト修正:
```
FLASH (rx) : ORIGIN = 0x00000800, LENGTH = 5K
```

**目標: UIAPduino込みで7KB（7,168B）以内に全モジュールを収める。**
現状3,984B + eink.c約1,024B + APIテーブル約64B = 約5,072B → UIAPduino込み約7,120B。
7,168Bまで残り約48B。サイズ計測を慎重に行うこと。

---

### Task D: eink.c の実装

**対象チップ:**
- Waveshare 1.54inch e-Paper V2（SSD1681・白黒・部分更新対応）を先行実装
- Waveshare 1.54inch e-Paper (G)（UC8154・4色）は後続で追加

**ピン定義（共通）:**
```c
// CS=PC0（CS_EINK・TFTと共用）/ DC=PC3 / RST=PC4 / BUSY=PD0
```

**実装すべき関数（最小実装・サイズ優先）:**

```c
void    eink_init(void);          // 初期化シーケンス（BUSY待機込み）
void    eink_full_update(void);   // フル更新（BUSY待機込み）
void    eink_fill(uint16_t color);// 全面塗りつぶし（DISP_BLACK/DISP_WHITE）
void    eink_draw_string(const char *str, uint16_t x, uint16_t y,
                         uint16_t fg, uint16_t bg);
void    eink_set_window(uint16_t x0, uint16_t y0,
                        uint16_t x1, uint16_t y1);
void    eink_write_pixel(uint16_t color); // 1ピクセル書き込み
void    eink_sleep(void);         // Deep Sleep（省電力）
```

**BUSY待機の実装:**
```c
static void eink_wait_busy(void)
{
    /* BUSY=PD0 が LOW になるまで待つ（HIGH=busy・LOW=idle） */
    while (GPIOD->INDR & (1u << 0));
}
```

**色変換ルール（disp_*統一APIに合わせる）:**
```c
// uint16_t color → 白黒変換（輝度で判定）
static uint8_t color_to_mono(uint16_t color)
{
    return (color > 0x7BEF) ? 1 : 0;  // 1=白・0=黒
}
```

**実装上の注意:**
- Waveshare公式のサンプルコード（GitHub: waveshare/e-Paper）を参考にすること
- LUT（波形テーブル）はSSD1681の標準LUTを使用。ハードコードで約30B
- 初期化シーケンスは最小限に絞ること（サイズ優先）
- spi_manager.c の `spi_cs_select(SPI_DEV_EINK)` を使用すること
- ビルドしてサイズを計測し、eink.c単体で1,024B以内を目標とする

---

### Task E: ディスプレイ統一API（LtcApi）の実装

**設計方針:**
- `uint16_t`（RGB565）で色を統一
- `DISP_*`定数を定義し機種非依存で使えるようにする
- ディスプレイ依存関数はテーブルの**末尾**に配置し、非依存関数のアドレスが機種によって変わらないようにする
- テーブルは固定アドレスに配置（リンカスクリプトで強制配置）

**色定数定義（新規ファイル `ltc_api.h`）:**

```c
#ifndef LTC_API_H
#define LTC_API_H

#include <stdint.h>

/* ディスプレイ非依存色定数（uint16_t統一） */
typedef uint16_t DispColor;

#define DISP_BLACK   0x0000U  /* 全機種: 黒 */
#define DISP_WHITE   0xFFFFU  /* 全機種: 白 */
#define DISP_RED     0xF800U  /* TFT/カラーOLED: 赤 / e-ink4色: 赤 / 白黒: 黒寄り */
#define DISP_GREEN   0x07E0U  /* TFT/カラーOLED: 緑 / 白黒: 白寄り */
#define DISP_BLUE    0x001FU  /* TFT/カラーOLED: 青 / 白黒: 黒寄り */

/* 固定APIテーブルのアドレス */
#define LTC_API_ADDR 0x00000804UL

/* APIテーブル構造体 */
typedef struct {
    /* ── ディスプレイ非依存（アドレス固定・機種共通） ── */
    void     (*iap_run)(uint32_t flash_addr);
    void     (*iap_call)(uint32_t flash_addr, uint8_t module_id, uint8_t call_type);
    void     (*iap_return)(void);
    void     (*load_resource)(uint16_t res_type_id, uint32_t fram_dest);
    uint8_t  (*keyscan_get)(void);
    uint8_t  (*keyscan_wait)(void);
    bool     (*keyscan_get_button)(uint8_t btn_mask);
    /* ── ディスプレイ依存（テーブル末尾・機種によって関数が異なる） ── */
    void     (*disp_init)(void);
    void     (*disp_fill)(DispColor color);
    void     (*disp_draw_string)(const char *str, uint16_t x, uint16_t y,
                                  DispColor fg, DispColor bg);
    void     (*disp_set_window)(uint16_t x0, uint16_t y0,
                                 uint16_t x1, uint16_t y1);
    void     (*disp_write_pixel)(DispColor color);
} LtcApi;

/* アプリ側からの呼び出しマクロ */
#define LTC_API  ((const LtcApi *)LTC_API_ADDR)

#endif /* LTC_API_H */
```

**テーブル実体（`ltc_api.c`・common_prog内）:**

```c
#include "ltc_api.h"
#include "iap.h"
#include "keyscan.h"
#include "tft_oled.h"  /* TFT機種 */
/* #include "eink.h" */ /* e-ink機種（g_iap_disp_typeで切り替え） */

/* リンカスクリプトで 0x00000804 に強制配置 */
__attribute__((section(".api_table"), used))
const LtcApi ltc_api_instance = {
    /* ディスプレイ非依存 */
    .iap_run             = iap_run,
    .iap_call            = iap_call_entry,
    .iap_return          = iap_return_entry,
    .load_resource       = NULL,  /* load_resource実装後に追加 */
    .keyscan_get         = keyscan_get,
    .keyscan_wait        = keyscan_wait,
    .keyscan_get_button  = keyscan_get_button,
    /* ディスプレイ依存（TFT機種） */
    .disp_init           = tft_init,
    .disp_fill           = tft_fill,
    .disp_draw_string    = tft_draw_string,
    .disp_set_window     = tft_set_window,
    .disp_write_pixel    = tft_write_pixel,
    /* e-ink機種の場合はeink_*関数に差し替える */
};
```

**リンカスクリプト追加（common_prog.ld）:**

```ld
/* .api_table を 0x0804 に強制配置（.textの直後） */
.api_table 0x00000804 :
{
    KEEP(*(.api_table))
} > FLASH
```

**実装上の注意:**
- `.api_table`セクションが確実に0x0804に配置されることをmapファイルで確認すること
- `LtcApi`構造体のメンバ順序は絶対に変更しないこと（アプリとの固定アドレス依存のため）
- テーブルサイズ: 13ポインタ × 4B = 52B（目標64B以内）
- TFT機種とe-ink機種でディスプレイ関数の差し替えは`g_iap_disp_type`の値で分岐する

---

### Task F: サイズ確認と統一案の確定

全モジュール実装後に以下を計測・確認する。

```bash
make clean && make size
```

**合格基準:**

| 項目 | 目標 | 結果 |
|------|------|------|
| common_prog Flash使用量 | 5,120B（5KB）以内 | ? |
| UIAPduino込み合計 | 7,168B（7KB）以内 | ? |
| App Area開始アドレス | 0x1C00 | ? |
| .api_tableのアドレス | 0x00000804 | ? |
| eink.c単体サイズ | 1,024B以内 | ? |
| ltc_api.c単体サイズ | 64B以内 | ? |
| RAM（.data+.bss+.ram_func） | 2,048B以内 | ? |

**サイズ超過時の削減方針（優先度順）:**
1. eink.cの初期化シーケンスを短縮（LUTを内蔵LUTに委ねる）
2. `disp_write_pixel()`をテーブルから削除（アプリでの単一ピクセル描画は稀）
3. boot.cのFRAMバックアップ処理を簡略化
4. それでも超過する場合はユーザーに相談すること

---

## 8c. Phase 2c 残実装タスク（サイズ検証前に全て完了すること）

以下のA〜Fを全て実装してから`make size`でサイズ計測すること。
各作業完了後に個別にビルドが通ることを確認すること。

---

### 作業A: boot.c の extern_comm_run() を本実装する

**現状**: while(1)スタブ
**正しい実装**: spi_flash_mgrをFRAMからApp Areaにロードして起動する

```c
static void extern_comm_run(void)
{
    GPIOA->BSHR = CART_READY_PIN;  // 外部通信モードでもCART_READY HIGH
    iap_restore_from_fram(FRAM_SPI_MGR_ADDR, 0x00001E00UL, 0x00002000UL);
    for (;;);  // ここには戻らない
}
```

`FRAM_SPI_MGR_ADDR`は`iap.h`で`0x00008000UL`として定義済み。

---

### 作業B: iap_ctx.S の call_type 第3引数を確認・修正する

`sram_save_to_fram()`のシグネチャ:
```c
void sram_save_to_fram(uint32_t sp_val, uint32_t ra_val,
                        uint8_t module_id, uint32_t caller_flash_addr);
```

iap_ctx.Sで`iap_call_entry`が以下の引数を正しく渡しているか確認する：
- `a0` = sp_val
- `a1` = ra_val
- `a2` = module_id（iap_callの第2引数）
- `a3` = caller_flash_addr（flash_addrを渡す）

`call_type`（iap_callの第3引数）はContextEntryに保存する必要がある。
`sram_save_to_fram()`のシグネチャにcall_typeを追加するか、
別途FRAMに書き込む処理を追加すること。

**bootloader_dev_spec_20260520.1.mdのセクション5B.5を参照すること。**

---

### 作業C: ltc_api.h の iap_call シグネチャを修正する

```c
// 修正前
void (*iap_call)(uint32_t flash_addr, uint8_t caller_module_id);

// 修正後
void (*iap_call)(uint32_t flash_addr, uint8_t module_id, uint8_t call_type);
```

ltc_api.cの`.iap_call = iap_call_entry`はそのままでよい。

---

### 作業D: ltc_api.h と ltc_api.c の disp_draw_string を修正する

**シグネチャ変更**（const char* → const uint16_t*・恵梨沙フォントインデックス列）:

```c
// ltc_api.h 修正後
void (*disp_draw_string)(const uint16_t *indices, uint8_t len,
                          uint16_t x, uint16_t y,
                          DispColor fg, DispColor bg);
```

```c
// ltc_api.c 修正後
.disp_draw_string = tft_draw_string_elysia,  // NULLから変更
```

tft_oled.hのプロトタイプが一致していることを確認すること。

---

### 作業E: eink.c の FRAM アクセスを fram_read() に変更する

`eink_draw_string_elysia()`内でSPIを直接叩いている箇所を修正する。

```c
// 修正前（SPIを直接叩いている）
spi_cs_select(SPI_DEV_FRAM);
spi_transfer(0x03);
spi_transfer((uint8_t)(addr >> 16));
spi_transfer((uint8_t)(addr >>  8));
spi_transfer((uint8_t)addr);
spi_read_buf(bm, 8);
spi_cs_deselect(SPI_DEV_FRAM);

// 修正後（公開関数を使う）
fram_read(addr, bm, 8);
```

`iap.h`をインクルードすること（fram_read()が宣言されている）。

---

### 作業F: keyscan.h のコメントを修正する

```c
// 修正前
/* MCP23017 I2C接続 SDA=PC1 / SCL=PC2 / ADDR=0x20 */

// 修正後
/* seesaw Mini I2C Gamepad (ADA-5743) SDA=PC1 / SCL=PC2 / ADDR=0x50 */
```

---

## 最終サイズ計測（作業A〜F完了後）

```bash
make clean && make size
```

以下の全項目が合格であることを確認して報告すること：

| 確認項目 | 目標 | 実測値 |
|---------|------|--------|
| common_prog FLASH | 5,632B（5.5KB）以内 | ? |
| UIAPduino込み合計 | 7,680B（7.5KB）以内 | ? |
| .api_tableアドレス | 0x08A0 | ? |
| App Area開始アドレス | 0x1E00 | ? |
| RAM（.data+.bss+.ram_func） | 2,048B以内 | ? |

**サイズ超過時は削減前にユーザーに報告・相談すること。**

---

## 参照すべき仕様書

| 仕様書 | 参照箇所 |
|--------|---------|
| bootloader_dev_spec_20260520.1.md | セクション2.5・2.5b・5B |
| cartridge_spec_master_20260520.1.md | FRAMレイアウト・メモリマップ |
