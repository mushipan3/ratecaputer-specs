# CAR-01 試験設計書 Phase 0〜2

**作成日: 2026-06-24**
**対象仕様書:**
- CAR-01_common_program_spec_20260615.md
- cartridge_master_20260617.md
- test_environment_spec_20260622.md §4・§4b・§5
- CAR-01_emulator_dev_20260622.md §17 Phase 0〜2

---

## 0. 基本方針

- エミュレータ上で先行実施・実機環境構築後に追確認
- 試験プログラムはエミュレータ・実機共通（test_app_phase2と同一バイナリ）
- Phase 3以降の試験設計書は各Phase完了後に別ファイルで作成
- 試験IDは各モジュール・レベル固有のプレフィックスで管理する

### 0.1 試験レベル定義

| レベル | 略称 | 対象 | 格納場所 |
|---|---|---|---|
| L1 単体試験 | UT | モジュール単体のI/F | 本書 §1 |
| L2 結合試験 | IT | モジュール間シーケンス | 本書 §2 |
| L3 システム試験 | ST | エンドツーエンド動作 | 本書 §3 |

### 0.2 試験環境前提

| 項目 | 値 | 根拠 |
|---|---|---|
| CS_FLASH | PA2 | boards_C §4.1・cartridge_master §2.13 |
| CS_FRAM | PA1 | 同上 |
| TFT DC | PC4 | 同上 |
| TFT RST | PD5 | 同上 |
| CS_FRAM_LOG（試験環境専用） | PD6 | test_environment_spec §1.3 |
| ジョイスティック（エミュレータ） | KeyInputI2CStub / I2C 0x51 | car01_plugin KeyInputI2CStub |
| ジョイスティック（実機） | ATtiny1604 / I2C（boards_D §4.3） | keyscan_design §1 |
| FONT_FRAM_ADDR | 0x13880 | common_program_spec §7確定版 |
| FRAM_META_BASE | 0x23880 | 同上 |

---

## 1. L1 単体試験

### 試験実施順序

以下の順序で実施すること。
キー入力確認（UT-KEY-03〜05）が
全試験の前提となるため最優先で実施する。

優先順序：
1. UT-KEY-03→UT-KEY-04→UT-KEY-05（キー入力確認）
2. UT-SPI-01〜06（SPIマネージャ）
3. UT-FRAM-01〜04（FRAMアクセス）
4. UT-FLASH-01〜05（SPI Flash）
5. UT-TFT-01〜05（TFTドライバ）
6. UT-KEY-01〜02（キースキャン残り）
7. UT-TIM-01（SysTick）
8. UT-CATALOG-01〜07（カタログ系）
9. UT-RESOURCE-01〜09 / UT-LOADRES-01〜03（リソースロード系）
10. UT-FONT-01〜02（フォントロード系）
11. UT-APPLIST-01〜02（アプリ一覧系）
12. L2結合試験（§2.0 UT-IAP-01〜04 **最優先**→IT-BOOT〜IT-IAP）
13. L3システム試験（ST-DISP〜ST-IAP）

> UT-IAP-01〜04 は複数モジュール（SRAM・Flash・FRAM・IAP機構）にまたがるため L2 に分類し、L2の最優先（§2.0）として実施する。

---

各試験項目の記載フォーマット：

| フィールド | 説明 |
|---|---|
| 試験ID | UT-XXX-NN |
| 試験名 | 試験内容の短い名称 |
| 対応仕様 | 仕様書・セクション |
| 前提条件 | 試験実施前に満たすべき状態 |
| 試験手順 | 実施ステップ |
| 試験データ | 入力値・設定値 |
| 期待値 | 具体的な数値・状態 |
| 合格基準 | PASS/FAIL判定条件 |
| 環境 | エミュレータ(E) / 実機(R) / 両方(E+R) |

---

### 1.1 SPIマネージャ単体試験 (UT-SPI)

#### UT-SPI-01 SPI初期化

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-01 |
| 試験名 | spi_manager_init() 正常完了確認 |
| 対応仕様 | CAR-01_common_program_spec §5.1・spi_manager_design §2 |
| 前提条件 | RCC・GPIO初期化済み。SPI1ペリフェラルへのアクセス可能。 |
| 試験手順 | 1. spi_manager_init()を呼び出す<br>2. SPI1のCR1レジスタ値を読み出す<br>3. CS_FLASH(PA2)・CS_FRAM(PA1)・CS_YMF(PD2)・CS_EINK(PC0)がHIGH状態であることを確認する |
| 試験データ | 入力なし（初期化関数） |
| 期待値 | SPI1 CR1: SPE=1・BR[2:0]=0x00（24MHz）・MSTR=1・CPOL=0・CPHA=0<br>全CSピンHIGH（非選択状態） |
| 合格基準 | 期待値と一致すること |
| 環境 | E+R |
| 取得方法 | spi_manager_init()呼び出し後 SPI1->CTLR1（0x40013000）を直接読む。GPIOA->OUTDR（0x4001080C）bit2（PA2=CS_FLASH）、GPIOA->OUTDR bit1（PA1=CS_FRAM）、GPIOC->OUTDR（0x4001100C）bit0（PC0=CS_EINK）、GPIOD->OUTDR（0x4001140C）bit2（PD2=CS_YMF）を読む |
| 判定方法 | SPE=(ctlr1>>6)&1==1、BR=(ctlr1>>3)&7==0、MSTR=(ctlr1>>2)&1==1、CPOL=(ctlr1>>1)&1==0、CPHA=(ctlr1>>0)&1==0、全CSピンbit==1（HIGH） |
| ログ出力形式 | RC:UT-SPI-01 CTLR1=0x%08X SPE=%d BR=%d MSTR=%d CPOL=%d CPHA=%d CS_PA2=%d CS_PA1=%d CS_PC0=%d CS_PD2=%d PASS/FAIL |

#### UT-SPI-02 FRAM 1バイト書き込み・読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-02 |
| 試験名 | fram_write_byte() / fram_read_byte() 正常動作確認 |
| 対応仕様 | CAR-01_common_program_spec §5.2・iap_design §2 |
| 前提条件 | UT-SPI-01 PASS。CS_FRAM=PA1接続済み。 |
| 試験手順 | 1. fram_write_byte(0x00020000, 0xA5)を呼び出す<br>2. fram_read_byte(0x00020000)を呼び出して読み出す<br>3. 書き込み値と一致するか確認する<br>4. 別アドレス(0x00020001)で値0x5Aで繰り返す |
| 試験データ | addr=0x00020000, val=0xA5 / addr=0x00020001, val=0x5A |
| 期待値 | fram_read_byte()戻り値 = 0xA5（1回目）・0x5A（2回目） |
| 合格基準 | 書き込み値と読み出し値が完全一致すること |
| 環境 | E+R |
| 取得方法 | fram_write_byte(0x00020000,0xA5)後にfram_read_byte(0x00020000)、fram_write_byte(0x00020001,0x5A)後にfram_read_byte(0x00020001) |
| 判定方法 | r1==0xA5 かつ r2==0x5A |
| ログ出力形式 | RC:UT-SPI-02 r1=0x%02X r2=0x%02X PASS/FAIL |

#### UT-SPI-03 FRAM バースト書き込み・読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-03 |
| 試験名 | fram_write() / fram_read() 256Bバースト転送確認 |
| 対応仕様 | CAR-01_common_program_spec §5.2・iap_design §2 |
| 前提条件 | UT-SPI-02 PASS |
| 試験手順 | 1. 256Bのテストパターン（0x00〜0xFF）をSRAMバッファに準備する<br>2. fram_write(0x00020100, buf, 256)を呼び出す<br>3. fram_read(0x00020100, readbuf, 256)で読み戻す<br>4. bufとreadbufをmemcmpで比較する |
| 試験データ | addr=0x00020100, data[256]={0x00,0x01,...,0xFF}, len=256 |
| 期待値 | memcmp(buf, readbuf, 256) == 0 |
| 合格基準 | 全256バイト一致すること |
| 環境 | E+R |
| 取得方法 | 256Bパターン（0x00〜0xFF）をSRAMバッファに準備してfram_write(0x00020100,buf,256)、fram_read(0x00020100,rbuf,256)、memcmp(buf,rbuf,256)の戻り値 |
| 判定方法 | memcmp==0 |
| ログ出力形式 | RC:UT-SPI-03 memcmp=%d PASS/FAIL |

#### UT-SPI-04 SPI Flash 1バイト読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-04 |
| 試験名 | flash_read() 先頭1バイト読み出し確認 |
| 対応仕様 | CAR-01_common_program_spec §8・iap_design §2 |
| 前提条件 | UT-SPI-01 PASS。CS_FLASH=PA2接続済み。SPI Flashに有効なデータまたは0xFFが書き込まれた状態。 |
| 試験手順 | 1. flash_read(0x000000, buf, 1)を呼び出す<br>2. 読み出し値を確認する（SPI Flashが空の場合は0xFF） |
| 試験データ | addr=0x000000, len=1 |
| 期待値 | 読み出しが完了すること（フォルト・ハングなし）<br>SPI Flash書き込み済みの場合：カタログヘッダ先頭バイト（0x01=有効エントリ or 0xFF=空） |
| 合格基準 | flash_read()がハング・フォルトなく完了すること |
| 環境 | E+R |
| 取得方法 | flash_read(0x000000,buf,1)を呼んで関数から戻ってくることを確認 |
| 判定方法 | 関数が戻ればPASS（buf[0]の値も記録） |
| ログ出力形式 | RC:UT-SPI-04 buf0=0x%02X PASS/FAIL |

#### UT-SPI-05 CSピン排他制御確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-05 |
| 試験名 | spi_cs_select() / spi_cs_deselect() 排他制御確認 |
| 対応仕様 | spi_manager_design §3（CS制御） |
| 前提条件 | UT-SPI-01 PASS |
| 試験手順 | 1. spi_cs_select(SPI_DEV_FLASH)を呼び出す<br>2. PA2がLOW・PA1がHIGHであることを確認する（エミュレータ：GPIO観測；実機：ロジアナ）<br>3. spi_cs_deselect(SPI_DEV_FLASH)を呼び出す<br>4. PA2がHIGHに戻ることを確認する<br>5. spi_cs_select(SPI_DEV_FRAM)で同様にPA1=LOW・PA2=HIGHを確認する |
| 試験データ | 対象デバイス: SPI_DEV_FLASH(2)・SPI_DEV_FRAM(3) |
| 期待値 | 選択デバイスのCSピンのみLOW・他はHIGH |
| 合格基準 | CSピンの排他状態が期待値通りであること |
| 環境 | E+R |
| 取得方法 | spi_cs_select(SPI_DEV_FLASH)呼び出し後にGPIOA->OUTDR bit2（PA2）、bit1（PA1）、GPIOC->OUTDR bit0（PC0）、GPIOD->OUTDR bit2（PD2）を読む |
| 判定方法 | PA2==0（LOW=選択中）、PA1==1・PC0==1・PD2==1（HIGH=非選択） |
| ログ出力形式 | RC:UT-SPI-05 PA2=%d PA1=%d PC0=%d PD2=%d PASS/FAIL |

#### UT-SPI-06 flash_to_fram_seq() 転送確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-SPI-06 |
| 試験名 | SPI Flash → FRAM 256Bチャンク転送 |
| 対応仕様 | iap_design §2（flash_to_fram_seq） |
| 前提条件 | UT-SPI-03・04 PASS。SPI FlashのFLASH_APP_BASE(0x098000)に既知データが書き込まれていること。 |
| 試験手順 | 1. flash_to_fram_seq(0x098000, 0x00030000, 256)を呼び出す<br>2. fram_read(0x00030000, readbuf, 256)で読み戻す<br>3. SPI Flashの同一アドレスから読んだ256Bと比較する |
| 試験データ | fsrc=0x098000, fdst=0x00030000, size=256 |
| 期待値 | memcmp(flash_data, fram_data, 256) == 0 |
| 合格基準 | 全256バイト一致すること |
| 環境 | E+R |
| 取得方法 | flash_read(0x008000,flash_buf,256)、flash_to_fram_seq(0x008000,0x00020200,256)、fram_read(0x00020200,fram_buf,256)、memcmp(flash_buf,fram_buf,256) |
| 判定方法 | memcmp==0 |
| ログ出力形式 | RC:UT-SPI-06 memcmp=%d PASS/FAIL |

---

### 1.2 FRAMアクセス単体試験 (UT-FRAM)

#### UT-FRAM-01 管理領域アクセス

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FRAM-01 |
| 試験名 | FRAM管理領域（0x00000〜0x01FFF）書き込み・読み出し |
| 対応仕様 | CAR-01_common_program_spec §7（管理領域8KB） |
| 前提条件 | UT-SPI-03 PASS |
| 試験手順 | 1. テストパターン32バイト（0xAA繰り返し）をFRAM 0x00000に書き込む<br>2. 同アドレスから32バイト読み出す<br>3. 書き込み値と一致するか確認する<br>4. 元のデータを復元するために0xFF×32バイトで上書きする |
| 試験データ | addr=0x00000, data[32]={0xAA...}, len=32 |
| 期待値 | 全32バイト0xAAで読み出せること |
| 合格基準 | memcmp一致 |
| 環境 | E+R |
| 取得方法 | 32Bバッファを0xAAで埋めてfram_write(0x00000,buf,32)、fram_read(0x00000,rbuf,32)、memcmp(buf,rbuf,32) |
| 判定方法 | memcmp==0 |
| ログ出力形式 | RC:UT-FRAM-01 memcmp=%d PASS/FAIL |

#### UT-FRAM-02 セーブデータ領域アクセス

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FRAM-02 |
| 試験名 | FRAMセーブデータ領域（0x02000〜0x05FFF）16KB境界確認 |
| 対応仕様 | CAR-01_common_program_spec §7（セーブデータ16KB） |
| 前提条件 | UT-SPI-03 PASS |
| 試験手順 | 1. 0x02000（先頭）・0x03FFE（中間）・0x05FFE（末尾-1）の3アドレスに各2バイトの識別データを書き込む<br>2. 各アドレスから読み戻して確認する |
| 試験データ | addr1=0x02000 val=0x1234, addr2=0x03FFE val=0x5678, addr3=0x05FFE val=0x9ABC |
| 期待値 | 各アドレスの読み出し値が書き込み値と一致すること |
| 合格基準 | 全3点一致 |
| 環境 | E+R |
| 取得方法 | 0x02000・0x03FFF・0x05FFF の3点でfram_write_byte(addr,test_val)後にfram_read_byte(addr)の戻り値 |
| 判定方法 | 各点で読み出し値==書き込み値 |
| ログ出力形式 | RC:UT-FRAM-02 p1=0x%02X p2=0x%02X p3=0x%02X PASS/FAIL |

#### UT-FRAM-03 コンテキストスタック領域アクセス

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FRAM-03 |
| 試験名 | FRAMコンテキストスタック領域（0x10000〜0x13880）書き込み・読み出し |
| 対応仕様 | CAR-01_common_program_spec §7確定版・iap_context_switch_variable_detail_design.md §2.2（CTX_STACK_BASE） |
| 前提条件 | UT-SPI-03 PASS |
| 試験手順 | 1. CTX_STACK_BASE(0x10000)に16バイトのヘッダパターン（深さカウンタ=0x01, 予備=0x00×15）を書き込む<br>2. 同アドレスから16バイト読み戻す<br>3. 0x10000+16（ContextEntry[0]先頭）に4バイト書き込み・読み出す<br>4. フォント領域先頭(0x13880)に4バイト書き込み・読み出して領域が独立していることを確認する |
| 試験データ | header[16]={0x01,0x00,...,0x00}, entry_top[4]={0xDE,0xAD,0xBE,0xEF} |
| 期待値 | 各アドレスの読み出し値が書き込み値と一致すること |
| 合格基準 | 全一致 |
| 環境 | E+R |
| 取得方法 | 0x10000・0x13880 の2点でfram_write_byte(addr,test_val)後にfram_read_byte(addr)の戻り値 |
| 判定方法 | 各点で読み出し値==書き込み値 |
| ログ出力形式 | RC:UT-FRAM-03 p1=0x%02X p2=0x%02X PASS/FAIL |

#### UT-FRAM-04 フォント展開領域アクセス（境界確認）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FRAM-04 |
| 試験名 | FRAMフォント領域（0x13880〜0x23880）先頭・末尾アクセス |
| 対応仕様 | CAR-01_common_program_spec §7確定版（FONT_FRAM_ADDR=0x13880・64KB確保） |
| 前提条件 | UT-SPI-03 PASS |
| 試験手順 | 1. FONT_FRAM_ADDR(0x13880)に4バイト書き込む<br>2. フォント領域末尾-4(0x2387C)に4バイト書き込む<br>3. FRAM_META_BASE(0x23880)に4バイト書き込む<br>4. 各アドレスから読み戻して値を確認する |
| 試験データ | addr_top=0x13880 val=0x11223344, addr_end=0x2387C val=0x55667788, addr_meta=0x23880 val=0x99AABBCC |
| 期待値 | 各アドレスの読み出し値が書き込み値と一致すること（アドレス間で値の混在なし） |
| 合格基準 | 全3点一致・値の混在なし |
| 環境 | E+R |
| 取得方法 | 0x13880・0x1C080・0x23878 の3点に異なる値（0xAA・0xBB・0xCC）を書いて読み返す |
| 判定方法 | 各点で読み出し値==書き込み値かつ値の混在なし |
| ログ出力形式 | RC:UT-FRAM-04 p1=0x%02X p2=0x%02X p3=0x%02X PASS/FAIL |

---

### 1.3 SPI Flash単体試験 (UT-FLASH)

#### UT-FLASH-01 カタログ領域読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-01 |
| 試験名 | SPI Flashカタログ領域（0x000000）先頭32バイト読み出し |
| 対応仕様 | CAR-01_common_program_spec §8（カタログテーブル32KB） |
| 前提条件 | UT-SPI-04 PASS |
| 試験手順 | 1. flash_read(0x000000, buf, 32)を呼び出す<br>2. 読み出しがハング・フォルトなく完了することを確認する<br>3. SPI Flash書き込み済みの場合はvalid(先頭1B)が0x01・disp_type(2B目)が0x01または0x02であることを確認する |
| 試験データ | addr=0x000000, len=32 |
| 期待値 | 読み出し完了（値は状態依存：書き込み済み=0x01...、未書き込み=0xFF...） |
| 合格基準 | flash_read()がハング・フォルトなく完了すること |
| 環境 | E+R |
| 取得方法 | flash_read(0x000000,buf,32)を呼ぶ |
| 判定方法 | 関数が戻ればPASS（buf[0]の値も記録） |
| ログ出力形式 | RC:UT-FLASH-01 buf0=0x%02X PASS/FAIL |

#### UT-FLASH-02 書き込みデータ一致確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-02 |
| 試験名 | SPI Flash 0x008000 書き込みデータ一致確認 |
| 対応仕様 | CAR-01_common_program_spec §8（恵梨沙フォント64KB固定） |
| 前提条件 | UT-SPI-04 PASS。エミュレータはspi_flash.binに既知パターンが書き込まれていること |
| 試験手順 | 1. 既知パターン（例: 0x01,0x02,...,0x08）をspi_flash.binの0x008000に書き込む（Pythonスクリプトで事前準備）<br>2. flash_read(0x008000, buf, 8)を呼び出す<br>3. 読み出し値が書き込んだパターンと一致することを確認する |
| 試験データ | addr=0x008000, len=8, expected={0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08} |
| 期待値 | buf[0]==0x01, buf[1]==0x02, ..., buf[7]==0x08 |
| 合格基準 | `memcmp(buf, expected, 8) == 0` |
| 環境 | E（spi_flash.binロード必須）/ R（Phase 2.5完了後） |
| 取得方法 | flash_read(0x008000,buf,8)を呼ぶ |
| 判定方法 | memcmp(buf, expected, 8) == 0 |
| ログ出力形式 | RC:UT-FLASH-02 buf=%02X%02X%02X%02X match=%d PASS/FAIL |

#### UT-FLASH-03 アプリ領域書き込み・読み出し（セクタ消去）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-03 |
| 試験名 | SPI Flashアプリ領域セクタ消去後の0xFF確認 |
| 対応仕様 | CAR-01_common_program_spec §8（アプリ領域0x098000〜）・test_environment_spec §4（SF-4） |
| 前提条件 | UT-SPI-04 PASS。flash_erase_sector()相当の手順が利用可能なこと。 |
| 試験手順 | 1. 0x098000の4KBセクタを消去する（spi_flash_mgrコマンド or エミュレータ直接操作）<br>2. flash_read(0x098000, buf, 16)で読み出す<br>3. 全バイトが0xFFであることを確認する |
| 試験データ | addr=0x098000, sector_size=4096 |
| 期待値 | buf[0]〜buf[15]がすべて0xFF |
| 合格基準 | 全16バイト0xFF |
| 環境 | R（実機のみ）<br>理由: エミュレータはSPI Flash書き換えをエミュレートしない |
| 取得方法 | iap.hのspi_erase_sector(0x09C000)相当の消去処理後にflash_read(0x09C000,buf,16) |
| 判定方法 | buf[0]〜buf[15]が全て0xFF |
| ログ出力形式 | RC:UT-FLASH-03 all_ff=%d PASS/FAIL |

#### UT-FLASH-04 アプリ領域 4KBブロック書き込み・読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-04 |
| 試験名 | SPI Flashアプリ領域への4KBデータ書き込みと検証 |
| 対応仕様 | CAR-01_common_program_spec §8・test_environment_spec §4（SF-3） |
| 前提条件 | UT-FLASH-03 PASS（消去済み） |
| 試験手順 | 1. 4KBのテストパターン（0x00〜0xFF繰り返し）を準備する<br>2. spi_flash_mgr経由（実機）またはエミュレータ直接でspi_flash.binの0x098000に書き込む<br>3. flash_read(0x098000, readbuf, 256)で先頭256Bを読み戻す<br>4. テストパターン先頭256Bと比較する |
| 試験データ | addr=0x098000, pattern=0x00〜0xFF×16回, verify_len=256 |
| 期待値 | memcmp(pattern, readbuf, 256) == 0 |
| 合格基準 | 256バイト完全一致 |
| 環境 | R（実機のみ）<br>理由: エミュレータはSPI Flash書き換えをエミュレートしない |
| 取得方法 | 256Bパターンを書き込んでflash_readで読み返す。memcmp(pattern,readbuf,256) |
| 判定方法 | memcmp==0 |
| ログ出力形式 | RC:UT-FLASH-04 memcmp=%d PASS/FAIL |

#### UT-FLASH-05 FLASH_APP_BASE定数確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-05 |
| 試験名 | FLASH_APP_BASE(0x098000)境界読み出し確認 |
| 対応仕様 | CAR-01_common_program_spec §8（アプリ領域先頭0x098000）・iap_design §1 |
| 前提条件 | UT-SPI-04 PASS |
| 試験手順 | 1. flash_read(FLASH_APP_BASE - 1, buf, 2)を呼び出す（システム予約領域末尾〜アプリ領域先頭をまたぐ） <br>2. flash_read(FLASH_APP_BASE, buf, 4)を呼び出す<br>3. いずれもハング・フォルトなく完了することを確認する |
| 試験データ | addr1=0x097FFF, len=2 / addr2=0x098000, len=4 |
| 期待値 | 両方の読み出しがハングなく完了すること |
| 合格基準 | ハング・フォルトなし |
| 環境 | E+R |
| 取得方法 | flash_read(0x097FFC,buf,4)とflash_read(0x098000,buf2,4)を呼ぶ |
| 判定方法 | 両関数が戻ればPASS |
| ログ出力形式 | RC:UT-FLASH-05 before=0x%08X after=0x%08X PASS/FAIL |

---

### 1.4 TFTドライバ単体試験 (UT-TFT)

#### UT-TFT-01 TFT初期化

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-01 |
| 試験名 | tft_init() 初期化シーケンス完了確認 |
| 対応仕様 | CAR-01_common_program_spec §5.1・§5.2（disp_init）・tft_oled_design §2 |
| 前提条件 | UT-SPI-01 PASS。CS=PC0、DC=PC4、RST=PD5接続済み。 |
| 試験手順 | 1. tft_init()を呼び出す<br>2. 関数がハング・フォルトなく完了することを確認する<br>3. エミュレータ：panel.htmlにTFT初期画像（白画面）が表示されることを目視確認する<br>4. 実機：TFT画面が白くなることを目視確認する |
| 試験データ | 入力なし |
| 期待値 | tft_init()完了・TFT画面が白色 |
| 合格基準 | 完了かつ画面が白色表示されること（目視） |
| 環境 | E+R |
| 取得方法 | tft_init()呼び出し後にログ出力。SDL2ウィンドウで目視確認 |
| 判定方法 | 関数が戻りかつ画面が白色（目視） |
| ログ出力形式 | RC:UT-TFT-01 init=done 目視確認要 |

#### UT-TFT-02 全画面塗りつぶし（単色）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-02 |
| 試験名 | tft_fill() 赤・緑・青の全画面塗りつぶし |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_fill） |
| 前提条件 | UT-TFT-01 PASS |
| 試験手順 | 1. tft_fill(TFT_RED)を呼び出す → 赤画面を目視確認する<br>2. tft_fill(TFT_GREEN)を呼び出す → 緑画面を目視確認する<br>3. tft_fill(TFT_BLUE)を呼び出す → 青画面を目視確認する<br>4. 各呼び出し後にDelay_Ms(500)を挿入して視認しやすくする |
| 試験データ | color: TFT_RED=0xF800・TFT_GREEN=0x07E0・TFT_BLUE=0x001F |
| 期待値 | 各呼び出し後に画面全体が指定色に変わること |
| 合格基準 | 3色とも全画面が正しい色で表示されること（目視） |
| 環境 | E+R |
| 取得方法 | tft_fill(DISP_RED)・tft_fill(DISP_GREEN)・tft_fill(DISP_BLUE)を順に呼ぶ。SDL2ウィンドウで目視確認 |
| 判定方法 | 3色とも全画面正しい色（目視） |
| ログ出力形式 | RC:UT-TFT-02 RED=done GREEN=done BLUE=done 目視確認要 |

#### UT-TFT-03 描画ウィンドウ設定

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-03 |
| 試験名 | tft_set_window() + tft_write_pixel() 矩形領域描画確認 |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_set_window / disp_write_pixel） |
| 前提条件 | UT-TFT-02 PASS |
| 試験手順 | 1. tft_fill(TFT_BLACK)で全画面を黒にする<br>2. tft_set_window(10, 10, 49, 49)で40×40のウィンドウを設定する<br>3. tft_write_pixel()を40×40=1,600回呼び出して赤ピクセルを書く<br>4. 10,10〜49,49の範囲のみ赤い矩形が描かれていることを目視確認する |
| 試験データ | x0=10, y0=10, x1=49, y1=49, color=TFT_RED |
| 期待値 | 40×40ピクセルの赤い矩形が左上(10,10)に描画されること |
| 合格基準 | 矩形の範囲・位置・色が目視で正しいこと |
| 環境 | E+R |
| 取得方法 | tft_set_window(10,10,49,49)後にDISP_REDでtft_write_pixel()を1600回呼ぶ。SDL2ウィンドウで目視確認 |
| 判定方法 | 左上(10,10)に40×40赤矩形（目視） |
| ログ出力形式 | RC:UT-TFT-03 pixels=1600 目視確認要 |

#### UT-TFT-04 日本語文字描画

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-04 |
| 試験名 | tft_draw_char_elysia() 日本語文字「テ」1文字描画 |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_draw_string）・tft_oled_design §2 |
| 前提条件 | UT-TFT-01 PASS。FRAMに恵梨沙フォントが展開済みであること（UT-FONT-01相当の前処理）。 |
| 試験手順 | 1. 恵梨沙フォントインデックス 329（「テ」）を使用する<br>2. tft_draw_char_elysia(329, 0, 0, TFT_WHITE, TFT_BLACK)を呼び出す<br>3. 座標(0,0)に8×8の白い「テ」が描画されることを目視確認する |
| 試験データ | font_idx=329（「テ」）, x=0, y=0, fg=TFT_WHITE, bg=TFT_BLACK |
| 期待値 | 「テ」の字形が座標(0,0)に8×8pxで描画されること |
| 合格基準 | 字形が目視で認識できること |
| 環境 | E+R（フォントロード後） |
| 取得方法 | 恵梨沙フォントのインデックスを指定してtft_draw_char_elysia(idx,0,0,DISP_WHITE,DISP_BLACK)。SDL2ウィンドウで目視確認。使用するインデックス値をRC:xxxログで出力すること |
| 判定方法 | 指定文字が目視で認識できる |
| ログ出力形式 | RC:UT-TFT-04 idx=%d 目視確認要 |

#### UT-TFT-05 日本語文字列描画

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-05 |
| 試験名 | disp_draw_string() 22文字日本語文字列描画 |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_draw_string） |
| 前提条件 | UT-TFT-04 PASS |
| 試験手順 | 1. 22文字の日本語文字列に対応する恵梨沙フォントインデックス配列を準備する<br>2. disp_draw_string(indices, 22, 0, 0, TFT_WHITE, TFT_BLACK)を呼び出す<br>3. 座標(0,0)から22文字の日本語文字列が描画されることを目視確認する |
| 試験データ | indices[22]（22文字日本語文字列の各インデックス）, len=22, x=0, y=0 |
| 期待値 | 22文字の日本語文字列が描画されること |
| 合格基準 | 文字列が目視で認識できること |
| 環境 | E+R |
| 取得方法 | 22文字分のインデックス配列を用意してdisp_draw_string(indices,22,0,0,DISP_WHITE,DISP_BLACK)。SDL2ウィンドウで目視確認 |
| 判定方法 | 22文字が目視認識できる |
| ログ出力形式 | RC:UT-TFT-05 len=22 目視確認要 |

---

### 1.5 キースキャン単体試験 (UT-KEY)

#### UT-KEY-01 キースキャン初期化

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-01 |
| 試験名 | keyscan_init() 正常完了確認 |
| 対応仕様 | keyscan_design §2・cartridge_master §1.5（SDA=PC1・SCL=PC2） |
| 前提条件 | I2Cバス(PC1/PC2)接続済み。開発段階：seesaw(ADA-5743, I2C 0x50)接続済み。 |
| 試験手順 | 1. keyscan_init()を呼び出す<br>2. 関数がハング・フォルトなく完了することを確認する<br>3. エミュレータ：ATtiny1604 I2Cスタブ(A-2-6)が応答すること |
| 試験データ | 入力なし |
| 期待値 | keyscan_init()完了（ハングなし）・I2C ACK受信 |
| 合格基準 | 完了・ハングなし |
| 環境 | E+R |
| 取得方法 | keyscan_init()呼び出し後にログ出力 |
| 判定方法 | 関数が戻ればPASS（ハングなし） |
| ログ出力形式 | RC:UT-KEY-01 init=done PASS/FAIL |

#### UT-KEY-02 ボタン入力読み取り（5方向スティック）

> **注記（2026-07-07）:** ボタン構成は boards_D_board_hardware §4.3（SKRHAAE010 5方向スティック）を正とする。
> アクションボタン（A/B/X/Y 等）の追加はD基板MCU見直し後（ATtiny1604→AVR64DD28 TSSOP 検討中）に本節を改定予定。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-02 |
| 試験名 | keyscan_get() 全ボタン入力確認（5方向スティック） |
| 対応仕様 | boards_D_board_hardware §4.3（SKRHAAE010）・keyscan_design §2 |
| 前提条件 | UT-KEY-01 PASS |
| 試験手順 | 1. 各方向（UP/DOWN/LEFT/RIGHT/CENTER）を順番に押し、keyscan_get()を呼び出す<br>2. 各押下に対して対応ビットが立っていることを確認する<br>3. ボタン未押下時は0x00が返ることを確認する |
| 試験データ | UP→0x01, DOWN→0x02, LEFT→0x04, RIGHT→0x08, CENTER→0x10 |
| 期待値 | 各ボタン押下時に対応するビットのみが立った値が返ること |
| 合格基準 | 5入力すべてで期待値と一致すること |
| 環境 | E+R |
| 取得方法 | エミュレータ: input_state.bin に各値を書いてkeyscan_get()を呼び戻り値を記録<br>実機: 各方向を手動操作してシリアルログを確認 |
| 判定方法 | keyscan_get() == 0x01/0x02/0x04/0x08/0x10（各入力で対応値） |
| ログ出力形式 | RC:UT-KEY-02 up=%d dn=%d lt=%d rt=%d ctr=%d PASS/FAIL |

| ボタン | ビットマスク | 試験操作 |
|---|---|---|
| UP | 0x01 | ジョイスティック上方向 |
| DOWN | 0x02 | ジョイスティック下方向 |
| LEFT | 0x04 | ジョイスティック左方向 |
| RIGHT | 0x08 | ジョイスティック右方向 |
| CENTER(BTN_OK) | 0x10 | ジョイスティック押し込み |

#### UT-KEY-03 keyscan_get() 無押下状態確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-03 |
| 試験名 | keyscan_get() 無押下時の戻り値確認 |
| 対応仕様 | keyscan_design §2 |
| 前提条件 | UT-KEY-01 PASS。ボタン未押下状態。 |
| 試験手順 | 1. ボタンを何も押さない状態でkeyscan_get()を呼び出す<br>2. 戻り値を確認する |
| 試験データ | 入力なし（ボタン未押下） |
| 期待値 | keyscan_get()戻り値 == 0x00 |
| 合格基準 | 戻り値が0x00 |
| 環境 | E+R |
| 取得方法 | input_state.binに0x00を書いた状態でkeyscan_get()を呼び戻り値を取得 |
| 判定方法 | 戻り値 == 0x00 |
| ログ出力形式 | RC:UT-KEY-03 val=0x%02X PASS/FAIL |

#### UT-KEY-04 keyscan_get_button() 単一ボタン確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-04 |
| 試験名 | keyscan_get_button() BTN_OKの押下・未押下判定 |
| 対応仕様 | keyscan_design §2（keyscan_get_button） |
| 前提条件 | UT-KEY-01 PASS |
| 試験手順 | 1. ボタン未押下状態でkeyscan_get_button(BTN_OK)を呼び出す → falseであることを確認する<br>2. エミュレータ：BTN_OKに相当するseesaw GPIOをLOWに設定してkeyscan_get_button(BTN_OK)を呼び出す → trueであることを確認する |
| 試験データ | btn_mask=BTN_OK(0x10) |
| 期待値 | 未押下=false・押下=true |
| 合格基準 | 期待値通りであること |
| 環境 | E+R |
| 取得方法 | ①input_state.binに0x00を書いてkeyscan_get_button(BTN_OK)を呼ぶ ②input_state.binに0x10を書いてkeyscan_get_button(BTN_OK)を呼ぶ |
| 判定方法 | ①==false ②==true |
| ログ出力形式 | RC:UT-KEY-04 no_press=%d press=%d PASS/FAIL |

#### UT-KEY-05 keyscan_wait() タイムアウト確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-05 |
| 試験名 | keyscan_wait(mask) mask=0で任意ボタン待機・BTN_OK押下で復帰 |
| 対応仕様 | keyscan_design §2（keyscan_wait(uint8_t mask)・mask=0で後方互換） |
| 前提条件 | UT-KEY-01 PASS |
| 試験手順 | 1. 別タスク（またはエミュレータ制御）で500ms後にBTN_OKを押す設定をする<br>2. keyscan_wait(0)を呼び出す（無期限待機・mask=0=任意ボタンで返る）<br>3. 戻り値がBTN_OK(0x10)であることを確認する |
| 試験データ | mask=0（任意ボタン）・押下ボタン=BTN_OK(0x10) |
| 期待値 | 戻り値 == 0x10 |
| 合格基準 | 戻り値が0x10 |
| 環境 | E+R |
| 取得方法 | input_state.binに0x10を書いた状態でkeyscan_wait(0)を呼び戻り値を取得 |
| 判定方法 | 戻り値 == 0x10 |
| ログ出力形式 | RC:UT-KEY-05 ret=0x%02X PASS/FAIL |

---

### 1.6 SysTick単体試験 (UT-TIM)

#### UT-TIM-01 SysTick動作確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TIM-01 |
| 試験名 | SysTick CNTレジスタのカウントアップ確認（時間計測基盤） |
| 対応仕様 | CAR-01_emulator_dev §17 A-1-6（SysTick 0xE000F000）・test_environment_spec §4b |
| 前提条件 | エミュレータ：SysTickスタブ(A-1-6)が実装済みであること。実機：CH32V003起動済み。 |
| 試験手順 | 1. SysTick->CNT（または相当レジスタ）を読み出してt0に保存する<br>2. Delay_Ms(1)を呼び出す<br>3. SysTick->CNTを再度読み出してt1に保存する<br>4. (t0 - t1)を計算してサイクル数を求める（CH32V003 ダウンカウンタ）<br>5. 48,000サイクル（=1ms@48MHz）に近い値であることを確認する（許容誤差±10%） |
| 試験データ | delay=1ms, クロック=48MHz |
| 期待値 | |t0 - t1| ≒ 48,000サイクル（実機: 43,200〜52,800 / エミュレータ: SysTickスタブの読み出しごと+48000のため144,000前後が観測される） |
| 合格基準 | 実機: 43,200〜52,800 / エミュレータ: 43,200〜200,000（スタブ特性を考慮した拡張範囲） |
| 環境 | E（スタブ動作確認）/ R（実測値記録） |
| 取得方法 | SysTick->CNT（0xE000F00C）を連続2回読んで差分を計算: t0=*(volatile uint32_t*)0xE000F00C、t1=*(volatile uint32_t*)0xE000F00C、diff=t1-t0 |
| 判定方法 | 43200 <= diff <= 200000（エミュレータ）/ 43200 <= diff <= 52800（実機） |
| ログ出力形式 | RC:UT-TIM-01 t0=%u t1=%u diff=%u PASS/FAIL |

---

### 1.7 カタログ系単体試験 (UT-CATALOG)

対象ファイル: `src/UIAPduino/common_prog/app_loader.c`  
試験環境: L2（x86 Unity）

#### UT-CATALOG-01 有効カタログエントリ読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-01 |
| 試験名 | valid=0x01 エントリ2件の正常読み出し |
| 対応仕様 | CAR-01_common_program_spec §7（FRAMカタログ構造） |
| 前提条件 | FRAMモックに valid=0x01 のエントリ2件が配置されていること |
| 試験手順 | 1. FRAMモックに valid=0x01 のカタログエントリ2件を設定する<br>2. read_app_list()を呼び出す<br>3. 2件が正しく読み出されることを確認する |
| 試験データ | エントリ1: valid=0x01, エントリ2: valid=0x01, 終端: valid=0xFF |
| 期待値 | 2件のエントリが読み出されること |
| 合格基準 | 読み出し件数==2 かつ entries[0].flash_addr==0x00098000・entries[1].flash_addr==0x00099000 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL(2, count) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-02 空カタログ（先頭終端）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-02 |
| 試験名 | 空カタログ（先頭 valid=0xFF）→ count==0 |
| 対応仕様 | CAR-01_common_program_spec §7（終端マーカー0xFF） |
| 前提条件 | FRAMモックの先頭エントリが valid=0xFF（未書き込みまたは全件削除後の状態）であること |
| 試験手順 | 1. FRAMモックの先頭26Bを 0xFF で埋める<br>2. read_app_list()を呼び出す<br>3. 0件で即終了することを確認する |
| 試験データ | FRAM先頭: 0xFF×26B |
| 期待値 | count==0・即終了 |
| 合格基準 | 読み出し件数==0 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_INT(0, count) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-03 削除済みエントリスキップ

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-03 |
| 試験名 | valid=0x00 削除済みエントリのスキップ |
| 対応仕様 | CAR-01_common_program_spec §7（削除フラグ0x00） |
| 前提条件 | FRAMモックに valid=0x00（削除済み）と valid=0x01（有効）のエントリが混在すること |
| 試験手順 | 1. valid=0x00, valid=0x01, valid=0xFF の順にFRAMモックを設定する<br>2. read_app_list()を呼び出す<br>3. valid=0x00 がスキップされ、valid=0x01 の1件のみ読み出されることを確認する |
| 試験データ | エントリ1: valid=0x00, エントリ2: valid=0x01, 終端: valid=0xFF |
| 期待値 | 1件のみ読み出し |
| 合格基準 | 読み出し件数==1 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL(1, count) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-04 タイトルフォントインデックス列の2B LE読み出し確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-04 |
| 試験名 | タイトルフォントインデックス列（uint16_t×8）の2B LE読み出し確認 |
| 対応仕様 | CAR-01_common_program_spec §7（`uint16_t title[8]` 恵梨沙フォントインデックス列） |
| 前提条件 | FRAMモックに既知フォントインデックス値が title フィールドとして配置されていること |
| 試験手順 | 1. title[0]=0x0149（「テ」）, title[7]=0x00FF など既知インデックスをFRAMモックに設定する<br>2. read_app_list()を呼び出す<br>3. entries[0].title[0]==0x0149、entries[0].title[7]==0x00FF であることを確認する |
| 試験データ | title[0]=0x0149, title[1]=0x0001, ..., title[7]=0x00FF |
| 期待値 | entries[0].title[0]==0x0149、entries[0].title[7]==0x00FF |
| 合格基準 | 全アサート一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_HEX16(0x0149, entries[0].title[0]) 等 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-05 途中終端でスキャン打ち切り

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-05 |
| 試験名 | 途中終端（valid=0xFF）でスキャンが打ち切られること |
| 対応仕様 | CAR-01_common_program_spec §7（終端マーカー0xFF） |
| 前提条件 | FRAMモックに valid=0x01 の有効エントリ1件 + valid=0xFF 終端が配置されていること |
| 試験手順 | 1. FRAMモックに有効エントリ1件を設定し、その直後に0xFF終端を設定する<br>2. read_app_list()を呼び出す<br>3. 1件で走査が止まることを確認する（終端以降は読まない） |
| 試験データ | エントリ0: valid=0x01, flash_addr=0x00098000 / エントリ1: valid=0xFF |
| 期待値 | count==1・entries[0].flash_addr==0x00098000 |
| 合格基準 | 読み出し件数==1 かつ flash_addr一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_INT(1, count) / TEST_ASSERT_EQUAL_UINT32(0x00098000UL, entries[0].flash_addr) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-06 全件削除済みで count==0

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-06 |
| 試験名 | 全件削除済み（valid=0x00×N件 + valid=0xFF終端）→ count==0 |
| 対応仕様 | CAR-01_common_program_spec §7（削除フラグ0x00・終端マーカー0xFF） |
| 前提条件 | FRAMモックに valid=0x00（削除済み）のエントリが複数件あり、末尾に0xFF終端が配置されていること |
| 試験手順 | 1. FRAMモックに valid=0x00 のエントリ2件を設定し、直後に0xFF終端を設定する<br>2. read_app_list()を呼び出す<br>3. 削除済みエントリをすべてスキップして count==0 となることを確認する |
| 試験データ | エントリ0: valid=0x00 / エントリ1: valid=0x00 / エントリ2: valid=0xFF |
| 期待値 | count==0 |
| 合格基準 | 読み出し件数==0 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_INT(0, count) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-07 max_entries 上限到達で打ち切り

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-07 |
| 試験名 | max_entries 上限到達でスキャンを打ち切ること |
| 対応仕様 | CAR-01_common_program_spec §7（read_app_list max_count引数） |
| 前提条件 | FRAMモックに有効エントリ3件が配置されていること |
| 試験手順 | 1. FRAMモックに有効エントリ3件 + 0xFF終端を設定する<br>2. read_app_list(entries, 2)（max_count=2）を呼び出す<br>3. 2件で打ち切られ、3件目が読まれないことを確認する |
| 試験データ | 有効エントリ3件（flash_addr: 0x98000/0x99000/0x9A000）、max_count=2 |
| 期待値 | count==2・entries[0].flash_addr==0x00098000・entries[1].flash_addr==0x00099000 |
| 合格基準 | 読み出し件数==2 かつ 各flash_addr一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_INT(2, count) / TEST_ASSERT_EQUAL_UINT32(0x00098000UL, ...) 等 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-CATALOG-08 disp_type フィルタ（2026-07-18定義追加・悉皆点検D）

> **経緯**: 本試験は tests/l2/test_catalog.c に実装され結果表に PASS（2026-07-10）が記録されて
> いたが、**設計書に定義が存在しないまま**だった（IT-IAP-03/04 の鏡像＝「定義なしで実施・記録」
> という逆方向の追跡漏れ）。悉皆点検D（設計⇔結果表の全ID突合）で検出し、実装から定義を復元した。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-CATALOG-08 |
| 試験名 | ハードプロファイル disp_type によるカタログエントリのフィルタ |
| 対応仕様 | CAR-01_common_program_spec §7（read_app_list・機種別表示フィルタ） |
| 前提条件 | RC_HW_PROFILE->disp_type=2（TFT）のモックプロファイル |
| 試験手順 | 1. disp_type=0（全機種）/2（TFT）/1（e-ink）の3エントリ+終端を配置する<br>2. read_app_list() を呼び出す<br>3. 全機種+TFTの2件のみ返り、e-ink専用が除外されることを確認する |
| 試験データ | 有効エントリ3件（disp_type=0x00/0x02/0x01） |
| 期待値 | count==2（disp_type=0と2のみ・1は除外） |
| 合格基準 | 読み出し件数==2 かつ 除外判定一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_INT(2, count) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

---

### 1.8 リソースロード系単体試験 (UT-RESOURCE / UT-LOADRES)

対象ファイル: `src/UIAPduino/common_prog/load_resource.c`

> **⚠ 再設計待ち（悉皆点検C・2026-07-18）**: UT-RESOURCE-01〜09 が検証していた
> **ディレクトリ線形サーチ・res_type_idビット分解・block_cellアドレス計算・size 3B LE
> デコード**のロジックは、Phase 2（2026-07-13）で `load_resource.c` から `iap.c` の
> **`find_resource()`** へ共通化・移動された。現行の `load_resource()` は
> `find_resource()` を呼ぶだけの薄いラッパになっており、`test_resource.c`（`load_resource.c`
> を直接インクルードして内部サーチを検証する構成）は **`find_resource` 未定義でリンク不能**。
> 旧PASS（2026-07-05〜08）は**移動前コードに対する記録**であり、現行コードの合否ではない。
>
> **再設計の2案**（方針未決）:
> 1. **find_resource() 抽出**: `iap.c` から `find_resource()` を独立TUへ切り出し、
>    Unity から直接リンクして 01〜09 相当のケースを検証する。
> 2. **Spike移行**: 実FRAMメタ+SPI Flashディレクトリを配置し、`load_resource()` の
>    end-to-end を Spike で検証する（UT-FONT-04 と同方式）。
>
> どちらも「検証対象が `find_resource()` へ移った」現実に追随する。決定まで
> `tests/l2/Makefile` の既定ビルドから `test_resource` を除外している。

#### UT-RESOURCE-01 正常リソースロード (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-01 |
| 試験名 | 正常リソースのロード（end-to-end） |
| 対応仕様 | CAR-01_common_program_spec §8（リソースロードシーケンス） |
| 前提条件 | Flashモック・FRAMモックに既知リソースデータが配置されていること |
| 試験手順 | 1. FlashモックにリソースディレクトリとデータをL2モックで設定する<br>2. load_resource()を呼び出す<br>3. FRAMモックに正しいデータが転送されていることを確認する |
| 試験データ | res_type_id=既知値, Flashデータ=既知パターン |
| 期待値 | FRAMに期待データが転送されること |
| 合格基準 | FRAM転送データが期待値と一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity memcmp(expected, fram_mock, len)==0 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-02 res_type_id不一致時の非転送 (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-02 |
| 試験名 | res_type_id不一致→転送なし |
| 対応仕様 | CAR-01_common_program_spec §8（リソース種別照合） |
| 前提条件 | Flashモックに異なるres_type_idのリソースが配置されていること |
| 試験手順 | 1. Flashモックに異なるres_type_idのエントリを配置する<br>2. 指定res_type_idでload_resource()を呼び出す<br>3. FRAMに転送が行われないことを確認する |
| 試験データ | 指定res_type_id≠Flash上のres_type_id |
| 期待値 | FRAM転送なし（エラー/0件返却） |
| 合格基準 | FRAM未更新 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EACH_EQUAL_UINT8(0xFF, got, 4)（FRAM転送先が初期値0xFFのまま） |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-03 ディレクトリ終端での即終了 (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-03 |
| 試験名 | ディレクトリ終端 0xFFFF で即終了 |
| 対応仕様 | CAR-01_common_program_spec §8（ディレクトリ終端マーカー） |
| 前提条件 | Flashモックにディレクトリ終端（0xFFFF）のみが配置されていること |
| 試験手順 | 1. Flashモックの先頭に0xFFFF終端を設定する<br>2. load_resource()を呼び出す<br>3. 転送なしで即終了することを確認する |
| 試験データ | ディレクトリ先頭: 0xFFFF |
| 期待値 | 即終了・FRAM転送なし |
| 合格基準 | FRAM未更新・ハングなし |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity ハングなし + TEST_ASSERT_EACH_EQUAL_UINT8(0xFF, got, 4)（FRAM転送先が初期値0xFFのまま） |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-04 res_type_id の type フィールド抽出確認 (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-04 |
| 試験名 | res_type_id bits[15:12] による type ディレクトリ参照確認 |
| 対応仕様 | CAR-01_common_program_spec §8（ResourceEntry res_type_id[15:12]=種別） |
| 前提条件 | FRAMモックに type=1 の type_start のみ設定されていること |
| 試験手順 | 1. type=1 の type_start を設定し、res_type_id=0x1ABC のエントリをFlashに配置する<br>2. load_resource(0x1ABC) を呼び出す → type=1 → マッチ → FRAM転送されること<br>3. load_resource(0x2ABC) を呼び出す → type=2 → 未登録(0xFFFF) → 転送なしで即returnすること |
| 試験データ | res_type_id: 0x1ABC（一致）/ 0x2ABC（type違い） |
| 期待値 | 0x1ABC: FRAM転送あり / 0x2ABC: 転送なし |
| 合格基準 | 両ケースのアサート一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_UINT8(0x12, got[0]) / TEST_ASSERT_EACH_EQUAL_UINT8(0xFF, ...) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-05 block_cell の cell フィールド→Flashアドレス算術確認 (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-05 |
| 試験名 | block_cell bits[3:0]=cell による Flash アドレスオフセット確認（非ゼロ cell） |
| 対応仕様 | CAR-01_common_program_spec §8（`res_addr = FLASH_APP_BASE + blk*BLOCK_SIZE + cell*CELL_SIZE`） |
| 前提条件 | Flashモックに cell=1（+256B オフセット）位置のリソースデータが配置されていること |
| 試験手順 | 1. block_cell=0x0021（blk=2, cell=1）のエントリを Flash に配置する<br>2. リソースデータを Flash 0x9A100（FLASH_APP_BASE+2×4096+1×256）に配置する<br>3. load_resource() を呼び出す<br>4. FRAM転送先のデータが 0x9A100 のデータと一致することを確認する |
| 試験データ | block_cell=0x0021, res_data={0xCA,0xFE,0xBA,0xBE} at 0x9A100 |
| 期待値 | FRAM転送先[0..3]=={0xCA,0xFE,0xBA,0xBE} |
| 合格基準 | 全バイト一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_UINT8(0xCA, got[0]) 等 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-06 リソースサイズ 3バイト LE デコード確認（256バイト超） (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-06 |
| 試験名 | リソースサイズ 3バイト LE デコード確認（size[1] 非ゼロ・258バイト） |
| 対応仕様 | CAR-01_common_program_spec §8（ResourceEntry size[3] バイト LE） |
| 前提条件 | Flashモックに 258バイトのパターンデータが配置されていること |
| 試験手順 | 1. size={0x02,0x01,0x00}=258バイト のエントリを Flash に配置する<br>2. 258バイトのパターンデータ（byte[i]=i&0xFF）を Flash に配置する<br>3. load_resource() を呼び出す<br>4. FRAM 転送先の byte[255], byte[256], byte[257] が正しいことを確認する（size[1]=0x01 が有効な証明） |
| 試験データ | size={0x02,0x01,0x00}（=258）, パターン: byte[i]=i&0xFF |
| 期待値 | got[255]=0xFF, got[256]=0x00, got[257]=0x01 |
| 合格基準 | 全アサート一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_UINT8() 4点確認 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-07 線形サーチで2件目以降にマッチ (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-07 |
| 試験名 | 線形サーチで先頭エントリをスキップして2件目にマッチ→転送 |
| 対応仕様 | CAR-01_common_program_spec §8（リソースディレクトリ線形サーチ） |
| 前提条件 | Flashモックのディレクトリ先頭に不一致エントリ、2件目に一致エントリが配置されていること |
| 試験手順 | 1. Flashモックのディレクトリ先頭に res_type_id=0x1002（不一致）を設定する<br>2. 2件目に res_type_id=0x1001（一致）を設定する<br>3. load_resource(0x1001)を呼び出す<br>4. 先頭エントリをスキップして2件目にマッチし、FRAM転送が行われることを確認する |
| 試験データ | dir[0]: res_type_id=0x1002 / dir[1]: res_type_id=0x1001, block_cell=0x0020, size=4 |
| 期待値 | FRAM転送先[0..3]=={0xDE,0xAD,0xBE,0xEF} |
| 合格基準 | 全バイト一致 |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_UINT8(0xDE, got[0]) 等 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-08 dir_cell 非ゼロでの dir_addr 計算確認 (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-08 |
| 試験名 | type_start bits[3:0]=dir_cell による dir_addr オフセット確認（非ゼロ dir_cell） |
| 対応仕様 | CAR-01_common_program_spec §8（`dir_addr = FLASH_APP_BASE + dir_blk*BLOCK_SIZE + dir_cell*CELL_SIZE`） |
| 前提条件 | Flashモックの dir_cell=1 位置（0x99100）にリソースディレクトリが配置されていること |
| 試験手順 | 1. type_start=0x0011（dir_blk=1, dir_cell=1 → dir_addr=0x99100）を設定する<br>2. 0x99100 にリソースエントリを設定する<br>3. load_resource()を呼び出す<br>4. dir_addr が 0x99100 に正しく解決されてリソースが転送されることを確認する |
| 試験データ | type_start=0x0011, エントリ at 0x99100: res_type_id=0x1001, block_cell=0x0020 |
| 期待値 | FRAM転送先[0..3]=={0x11,0x22,0x33,0x44} |
| 合格基準 | 全バイト一致（dir_cell=0 の 0x99000 ではなく 0x99100 が参照されること） |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EQUAL_UINT8(0x11, got[0]) 等 |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-RESOURCE-09 リソースサイズ=0 のケース (L2)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-RESOURCE-09 |
| 試験名 | リソースサイズ=0 のとき FRAM 転送が発生しないこと |
| 対応仕様 | CAR-01_common_program_spec §8（ResourceEntry size[3] バイト LE） |
| 前提条件 | Flashモックに size={0,0,0} のリソースエントリが配置されていること |
| 試験手順 | 1. size=0 のリソースエントリを Flash に設定する<br>2. load_resource()を呼び出す<br>3. flash_to_fram_seq() に len=0 が渡され、FRAM 転送が発生しないことを確認する |
| 試験データ | size={0,0,0}（=0バイト） |
| 期待値 | FRAM転送先は初期値0xFFのまま（転送なし） |
| 合格基準 | FRAM未更新・ハングなし |
| 環境 | E（x86 Unity） |
| 取得方法 | x86 上でUnityテストランナーを実行し、stdout の PASS/FAIL を確認する |
| 判定方法 | Unity TEST_ASSERT_EACH_EQUAL_UINT8(0xFF, got, 4) |
| ログ出力形式 | Unity出力（PASS/FAIL） |

#### UT-LOADRES-01 load_resource() 正常系 (L3)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-LOADRES-01 |
| 試験名 | load_resource() 正常系（Flashスタブに既知ディレクトリ配置） |
| 対応仕様 | CAR-01_common_program_spec §8（リソースロードシーケンス） |
| 前提条件 | spi_flash.binに既知リソースディレクトリとデータが配置されていること（setup_wave2_stubs.pyで自動配置） |
| 試験手順 | 1. run_test_ut.shがsetup_wave2_stubs.pyでspi_flash.binを設定する<br>2. test_app_ut.cのUT-LOADRES-01がload_resource()を呼び出す<br>3. FRAM転送が正常に完了しRC:UT-LOADRES-01 PASSを出力することを確認する |
| 試験データ | spi_flash.binの既知ディレクトリエントリ・既知リソースデータ |
| 期待値 | load_resource()が正常完了・FRAM転送OK |
| 合格基準 | RC:UT-LOADRES-01 PASS |
| 環境 | E（Spike + car01_plugin） |
| 取得方法 | run_test_ut.sh を実行し、ログファイルの RC:UT-LOADRES-01 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-LOADRES-01 PASSが出力されること |
| ログ出力形式 | RC:UT-LOADRES-01 PASS/FAIL |

#### UT-LOADRES-02 load_resource() 未登録種別 (L3)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-LOADRES-02 |
| 試験名 | load_resource() 未登録種別（type_start=0xFFFF） |
| 対応仕様 | CAR-01_common_program_spec §8（リソース種別照合） |
| 前提条件 | UT-LOADRES-01 PASS |
| 試験手順 | 1. type_start=0xFFFF（未登録）でload_resource()を呼び出す<br>2. FRAM転送なしで正常終了することを確認する |
| 試験データ | type_start=0xFFFF |
| 期待値 | 転送なし・正常終了 |
| 合格基準 | RC:UT-LOADRES-02 PASS（転送なし確認） |
| 環境 | E（Spike） |
| 取得方法 | run_test_ut.sh を実行し、ログファイルの RC:UT-LOADRES-02 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-LOADRES-02 PASSが出力されること |
| ログ出力形式 | RC:UT-LOADRES-02 PASS/FAIL |

#### UT-LOADRES-03 load_resource() エントリ未発見 (L3)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-LOADRES-03 |
| 試験名 | load_resource() エントリ未発見（ディレクトリ終端到達） |
| 対応仕様 | CAR-01_common_program_spec §8（ディレクトリ終端マーカー） |
| 前提条件 | UT-LOADRES-01 PASS |
| 試験手順 | 1. Flashスタブに対象エントリなしのディレクトリを配置する<br>2. load_resource()を呼び出す<br>3. 終端到達で正常終了することを確認する |
| 試験データ | spi_flash.binのディレクトリ: 対象エントリなし |
| 期待値 | 終端到達・転送なし・正常終了 |
| 合格基準 | RC:UT-LOADRES-03 PASS |
| 環境 | E（Spike） |
| 取得方法 | run_test_ut.sh を実行し、ログファイルの RC:UT-LOADRES-03 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-LOADRES-03 PASSが出力されること |
| ログ出力形式 | RC:UT-LOADRES-03 PASS/FAIL |

---

### 1.9 フォントロード系単体試験 (UT-FONT)

対象ファイル: `src/UIAPduino/common_prog/boot.c` / `flash_to_fram_seq()`

#### UT-FONT-01 Flash→FRAM転送整合性 (L3)

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FONT-01 |
| 試験名 | Flash→FRAM転送の整合性（512Bパターン・先頭・末尾・中間） |
| 対応仕様 | boot_design §3（フォントロードシーケンス）・CAR-01_common_program_spec §7 |
| 前提条件 | spi_flash.binに既知パターン（512B）が書き込まれていること |
| 試験手順 | 1. spi_flash.binの既知パターンをflash_to_fram_seq()で転送する<br>2. FRAM上のデータがFlashの既知パターンと一致することを確認する（先頭・末尾・中間） |
| 試験データ | 512Bの既知パターン（0x00〜0xFF繰り返し） |
| 期待値 | FRAM転送データがFlashパターンと完全一致 |
| 合格基準 | RC:UT-FONT-01 PASS（先頭・末尾・中間3点確認） |
| 環境 | E（Spike + car01_plugin） |
| 取得方法 | run_test_ut.sh を実行し、ログファイルの RC:UT-FONT-01 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-FONT-01 PASSが出力されること |
| ログ出力形式 | RC:UT-FONT-01 PASS/FAIL |

#### UT-FONT-02 フォント領域オフセット計算 (L2)

> ★**2026-07-31 注記**: 下の「96KB×4スロット」は**廃止された旧構成**。確定済みの正は
> 2ビット4階調2サイズ（0x024000 小144KB・16B/字 ／ 0x048000 大320KB・36B/字・
> 文字集合＝JIS X 0213:2004 第1面 ≈8,797字）＝`memory_map_canonical` §SPI Flash。
> 本試験は **(B2) 実装前の暫定実装**を検証しており、(B2) 実装時に基準値を更新する。
> 以下の改訂記録は当時の事実として保全する。

**2026-07-18改訂**: フォントのSPI Flash直読み化（96KB×4スロット・cartridge_master Rev.1.8）に伴い、
基準を旧FRAMアドレス（ELYSIA_FONT_BASE 0x13880＝旧々マップ）から FONT_FLASH_BASE（0x008000）へ是正。
実装（test_app_l2.c）は是正済みだったが本定義が旧記述のまま乖離していた（悉皆点検C該当・本改訂で解消）。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FONT-02 |
| 試験名 | フォント領域オフセット計算（フォントインデックス→SPI Flashアドレス） |
| 対応仕様 | CAR-01_common_program_spec §8（フォントアドレス計算・SPI Flash直読み） |
| 前提条件 | なし（純粋な算術試験） |
| 試験手順 | 1. 文字コード（インデックス）からSPI Flashアドレスを計算する<br>2. 計算結果が期待値と一致することを確認する |
| 試験データ | idx = 0 / 1 / 255 |
| 期待値 | FONT_FLASH_BASE + idx * 8（idx=0→0x008000, idx=1→0x008008, idx=255→0x0087F8） |
| 合格基準 | 全ケースで計算結果が期待値と一致 |
| 環境 | E（Spike + car01_plugin・run_test_l2.sh） |
| 取得方法 | run_test_l2.sh を実行し、ログの RC:UT-FONT-02 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-FONT-02 PASSが出力されること |
| ログ出力形式 | RC:UT-FONT-02 PASS/FAIL |

#### UT-FONT-04 字体スロット切替（2026-07-18新設・絨毯爆撃④）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FONT-04 |
| 試験名 | 字体スロット選択（font_slot_init/font_slot_base）の実経路検証 |
| 対応仕様 | `memory_map_canonical` §SPI Flash（**字体データは 0x024000 小144KB／0x048000 大320KB・2ビット4階調2サイズ・2026-07-19確定**）・統一インデックス空間・FRAM_FONT_SLOT_SEL(0x0A8F0)。★本試験が現在検証しているのは **(B2) 実装前の暫定実装**（FONT_FLASH_BASE 0x008000・8B/字）＝(B2) 実装時に基準値を確定マップへ更新すること。旧「96KB×4スロット」は廃止（2026-07-31 是正） |
| 前提条件 | font_slot.c（common_prog実物）を test_app_l2 に直接リンク（ADDITIONAL_C_FILES）。FRAM実書き込み経路を使用 |
| 試験手順 | 1. FRAM_FONT_SLOT_SEL へ選択値を実書き込み<br>2. font_slot_init() を呼ぶ<br>3. font_slot_base() の返却値を期待値と比較（6ケース繰り返し） |
| 試験データ | sel = 0x00 / 0x01 / 0x02 / 0x03（正常系）・0x04 / 0xFF（範囲外） |
| 期待値 | slot0〜3 → 0x008000 / 0x020000 / 0x038000 / 0x050000。範囲外 → 0x008000（slot0フォールバック・fail-safe） |
| 合格基準 | 全6ケースで返却値が期待値と一致 |
| 環境 | E（Spike + car01_plugin・run_test_l2.sh） |
| 取得方法 | run_test_l2.sh を実行し、ログの RC:UT-FONT-04 PASS/FAIL を確認する |
| 判定方法 | ログにRC:UT-FONT-04 PASSが出力されること |
| ログ出力形式 | RC:UT-FONT-04 sel=/base= 各ケース + RC:UT-FONT-04 PASS/FAIL |

**位置づけ**: 式の再計算（UT-FONT-02方式）ではなく、**common_prog の font_slot.c 実物オブジェクトを
リンクして実FRAM読み書き→初期化→取得の実経路を踏む**。UT-FONT-02 の「※字体切替の検証は
IT-FONT-02再定義で扱う」とした宿題のうち、スロット選択ロジック部分を UT として先行決着させた
（描画までの結合検証は IT-FONT-02 再定義に残る）。

---

### 1.10 アプリ一覧系試験 (UT-APPLIST)

対象ファイル: `src/UIAPduino/common_prog/app_loader.c`  
試験環境: L3（Spike・目視確認）

#### UT-APPLIST-01 read_app_list() FRAM経由読み取り

| 項目 | 内容 |
|---|---|
| 試験ID | UT-APPLIST-01 |
| 試験名 | read_app_list() FRAMスタブ経由の読み取り |
| 対応仕様 | CAR-01_common_program_spec §7（アプリ一覧表示） |
| 前提条件 | fram.binに有効カタログエントリが配置されていること |
| 試験手順 | 1. fram.binにカタログエントリを設定する<br>2. test_app_l2バイナリ（Spike）でread_app_list()を実行する<br>3. アプリ一覧がTFTに表示されることを目視確認する |
| 試験データ | fram.binのカタログエントリ（有効データ） |
| 期待値 | アプリ一覧がTFTに表示されること |
| 合格基準 | 目視確認でアプリ一覧が表示されること |
| 環境 | E（Spike・目視確認） |
| 取得方法 | run_test_l2.sh --manual を実行し、Spike上のTFT表示を目視確認する |
| 判定方法 | 目視確認 |
| ログ出力形式 | RC:UT-APPLIST-01 目視確認要 |

#### UT-APPLIST-02 カタログ0件時の動作

| 項目 | 内容 |
|---|---|
| 試験ID | UT-APPLIST-02 |
| 試験名 | カタログ0件時の動作（0xFF終端） |
| 対応仕様 | CAR-01_common_program_spec §7（空カタログ時の表示） |
| 前提条件 | fram.binのカタログ先頭が0xFF（終端）であること |
| 試験手順 | 1. fram.binのカタログ先頭を0xFF終端に設定する<br>2. test_app_l2バイナリ（Spike）でread_app_list()を実行する<br>3. アプリなし表示（または空一覧）が出ることを目視確認する |
| 試験データ | fram.binのカタログ先頭: 0xFF |
| 期待値 | 空カタログの表示（アプリなし） |
| 合格基準 | 目視確認でアプリなし/空一覧が表示されること |
| 環境 | E（Spike・目視確認） |
| 取得方法 | run_test_l2.sh --manual を実行し、Spike上のTFT表示を目視確認する |
| 判定方法 | 目視確認 |
| ログ出力形式 | RC:UT-APPLIST-02 目視確認要 |

---

## 2. L2 結合試験

### 2.0 IAP基本機能試験（最優先）(UT-IAP)

> **優先度: 最高** — IAP機構はSRAM・Flash・FRAM・コンテキストスイッチを横断する共通機能の中核であり、
> 上位のIT-IAP試験の前提となる。L1試験（単一モジュール）ではなくL2（複数モジュール連携）に分類する。

#### UT-IAP-01 iap_run() パターンB ソフトリセット

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-01 |
| 試験名 | iap_run() SPI FlashからApp Areaへ書き込みとソフトリセット |
| 対応仕様 | CAR-01_common_program_spec §5.3（パターンB）・iap_design §2 |
| 前提条件 | UT-SPI-06 PASS。SPI Flash FLASH_APP_BASE(0x098000)に起動可能な試験バイナリが書き込まれていること。 |
| 試験手順 | 1. iap_run(FLASH_APP_BASE)を呼び出す<br>2. 関数は戻らない（ソフトリセット発動）<br>3. リセット後に試験バイナリが起動し、TFTに「IAP-B OK」を表示することを確認する |
| 試験データ | flash_addr=0x098000 |
| 期待値 | iap_run()呼び出し後にソフトリセット発動・試験バイナリが起動すること |
| 合格基準 | TFTに「IAP-B OK」が表示されること（または試験バイナリのLED点滅確認） |
| 環境 | E+R |
| 取得方法 | iap_run()呼び出し→リセット後に起動したバイナリがRC:UT-IAP-01 PASSをログ出力 |
| 判定方法 | リセット後のログにPASSが出る |
| ログ出力形式 | RC:UT-IAP-01 boot_from_iap=1 PASS |

#### UT-IAP-02 iap_call() パターンA-1 同一アプリ内呼び出し

> **2026-07-11更新:** 可変長ContextEntry化（`iap_context_switch_variable_design.md`）に伴い、本試験は可変長版のContextEntryを対象とする。外部から見た合格基準（g_test_val復元）は固定版と同一。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-02 |
| 試験名 | iap_call()によるモジュール起動と戻り確認（可変長ContextEntry版） |
| 対応仕様 | iap_context_switch_variable_design.md §2.6・iap_design §2 |
| 前提条件 | UT-IAP-01 PASS。SPI Flash上に「iap_return()を呼び出す」試験モジュールが格納されていること。 |
| 試験手順 | 1. SRAM上のグローバル変数g_test_val=0x1234を設定する<br>2. iap_call(module_addr, MODULE_ID_TEST, IAP_CALL_INTERNAL)を呼び出す<br>3. 試験モジュール内でiap_return()を呼び出す<br>4. 呼び出し元に戻ったことを確認する<br>5. g_test_valが0x1234のまま復元されていることを確認する（SRAMイメージ復元の確認） |
| 試験データ | module_addr=SPI Flash内試験モジュールアドレス, call_type=IAP_CALL_INTERNAL(0) |
| 期待値 | iap_return()後に呼び出し元が再開・g_test_val==0x1234 |
| 合格基準 | 呼び出し元への復帰・SRAMの値が復元されること |
| 環境 | E+R（**実機でのiap_ctx.S検証PENDING #3**） |
| 取得方法 | iap_call()でサブモジュールを起動、iap_return()後にg_test_val（SRAM上の変数）をRC:xxxログで出力 |
| 判定方法 | g_test_val == 0x1234 |
| ログ出力形式 | RC:UT-IAP-02 g_test_val=0x%04X PASS/FAIL |

#### UT-IAP-03 iap_return() SRAMイメージ復元確認

> **2026-07-11更新:** 可変長ContextEntry化に伴い、本試験は「Zone C退避範囲内」の復元を確認する（固定2KB全体ではなく、呼び出し先flash_sizeから算出されるsram_size分のみが退避・復元される）。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-03 |
| 試験名 | iap_call()→iap_return()でZone Cが正しく復元されることの確認（可変長版） |
| 対応仕様 | iap_context_switch_variable_design.md §2.2・§2.6 |
| 前提条件 | UT-IAP-02 PASS |
| 試験手順 | 1. Zone C内（試験モジュールのsram_size範囲内）に既知パターン（0xDEADBEEF等）を複数箇所に配置する<br>2. iap_call()でモジュールを呼び出す（モジュールはZone C内を書き換えてiap_return()） <br>3. iap_return()後に各箇所のパターンが元に戻っていることを確認する |
| 試験データ | pattern: Zone C先頭+0x00=0xDEAD, +0x20=0xBEEF, +(sram_size-4)=0xCAFE |
| 期待値 | 各パターンがiap_return()後に復元されていること |
| 合格基準 | 全パターン一致 |
| 環境 | E+R（実機検証PENDING） |
| 取得方法 | SRAM上の複数アドレスにパターンを書いてiap_call→iap_return後に読み返す。mismatch_count=不一致バイト数 |
| 判定方法 | mismatch_count == 0 |
| ログ出力形式 | RC:UT-IAP-03 mismatch=%d PASS/FAIL |

#### UT-IAP-04 iap_restore_from_fram() 自己復元確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-04 |
| 試験名 | iap_restore_from_fram() FRAMバックアップからApp Areaへの書き戻し |
| 対応仕様 | iap_design §2（iap_restore_from_fram）・boot_design §3（バックアップ） |
| 前提条件 | UT-SPI-06 PASS。FRAM_BACKUP_ADDR(0x06000)に起動可能なバイナリが存在すること。 |
| 試験手順 | 1. App Areaを既知のパターンで上書きする（破損状態を模擬）<br>2. iap_restore_from_fram(FRAM_BACKUP_ADDR, APP_BASE, BACKUP_SIZE)を呼び出す<br>3. 関数は戻らない（ソフトリセット発動）<br>4. リセット後にFRAMバックアップのバイナリが起動することを確認する |
| 試験データ | fram_src=0x06000, app_base=0x2000, app_size=8192 |
| 期待値 | FRAMバックアップから起動すること |
| 合格基準 | バックアップバイナリが起動すること |
| 環境 | E+R |
| 取得方法 | FRAM_BACKUP_ADDR(0x06000)に試験バイナリを書いてiap_restore_from_fram()呼び出し→リセット後に起動したバイナリがRC:UT-IAP-04 PASSをログ出力 |
| 判定方法 | リセット後のログにPASSが出る |
| ログ出力形式 | RC:UT-IAP-04 boot_from_fram=1 PASS |

---

### 2.0b IAP可変サイズコンテキストスイッチ試験（UT-IAP-05〜10）

> **新設（2026-07-11）:** `iap_context_switch_variable_design.md`に基づく可変長ContextEntry化の試験項目。UT-IAP-02/03（本節冒頭）はこの節の前提として先にPASSしていること。

#### UT-IAP-05 SRAM退避サイズ計算式の検証

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-05 |
| 試験名 | calc_sram_size()が退避サイズ式通りの値を返すことの確認 |
| 対応仕様 | iap_context_switch_variable_design.md §2.3 |
| 前提条件 | なし（単体関数試験） |
| 試験手順 | 1. flash_size = 256, 512, 1024, 2048 の4パターンでcalc_sram_size()を呼ぶ<br>2. 戻り値を期待値と比較する |
| 試験データ | 256→64（最小値適用）, 512→64（最小値適用）, 1024→128, 2048→256 |
| 期待値 | 全4パターンで`max(64, ceil(flash_size/8/32)×32)`と一致すること |
| 合格基準 | 4パターン全て一致 |
| 環境 | E+R |
| 取得方法 | 4パターンの計算結果をRC:xxxログで出力 |
| 判定方法 | 4パターン全て期待値と一致 |
| ログ出力形式 | RC:UT-IAP-05 sram_size[4]=%d,%d,%d,%d PASS/FAIL |

#### UT-IAP-06 複数サイズでのiap_call/iap_return動作確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-06 |
| 試験名 | 256B/1KB/2KB相当の異なるサイズの試験モジュールでiap_call/iap_returnが正常動作することの確認 |
| 対応仕様 | iap_context_switch_variable_design.md §2.2・§2.6 |
| 前提条件 | UT-IAP-02/03 PASS |
| 試験手順 | 1. flash_size=256B/1KB/2KB相当の3種類の試験モジュールを用意する<br>2. それぞれについてiap_call→iap_returnを実行し、g_test_val復元とZone Cパターン復元を確認する |
| 試験データ | module_256, module_1k, module_2k（いずれもg_test_val=0x1234設定） |
| 期待値 | 3パターンともg_test_val==0x1234・Zone Cパターン一致 |
| 合格基準 | 3パターン全てPASS |
| 環境 | E+R（実機検証PENDING #3） |
| 取得方法 | 3パターンそれぞれの結果をRC:xxxログで出力 |
| 判定方法 | 3パターン全て一致 |
| ログ出力形式 | RC:UT-IAP-06 size=%d g_test_val=0x%04X PASS/FAIL |

#### UT-IAP-07 IapCallStatus正常系

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-07 |
| 試験名 | 正常呼び出し時にiap_call()がstatus=0(OK)を返すことの確認 |
| 対応仕様 | iap_context_switch_variable_design.md §2.5 |
| 前提条件 | UT-IAP-02 PASS |
| 試験手順 | 1. Zone C・FRAM CTXスタックに十分な空きがある状態でiap_call()を呼ぶ<br>2. 戻り値（a0）のstatusフィールドを確認する |
| 試験データ | 通常状態（空きあり） |
| 期待値 | status == 0 |
| 合格基準 | status == 0 |
| 環境 | E+R |
| 取得方法 | iap_call()戻り値をRC:xxxログで出力 |
| 判定方法 | status == 0 |
| ログ出力形式 | RC:UT-IAP-07 status=%d PASS/FAIL |

#### UT-IAP-08 ERR_SRAM_FULL（callee単体サイズ超過拒否）

> **2026-07-11修正:** Zone Cアドレスモデル確定（単一ウィンドウ再利用モデル・iap_context_switch_variable_design.md §2.2）により、Zone Cは累積使用量ではなく「常に1コンテキスト分のみ使用」となったため、ERR_SRAM_FULLは「ネストの積み重ねによる枯渇」ではなく「callee単体のsram_sizeが1024Bを超える」場合のみ発生する異常系ガードに再定義する。

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-08 |
| 試験名 | callee単体のsram_sizeがZone Cサイズ(1024B)を超える場合にERR_SRAM_FULLで拒否されることの確認 |
| 対応仕様 | iap_context_switch_variable_design.md §2.2・§2.5 |
| 前提条件 | UT-IAP-06 PASS |
| 試験手順 | 1. flash_sizeが calc_sram_size(flash_size) > 1024B となるような大きなモジュール（例: 32KB相当）を疑似的に指定してiap_call()を呼ぶ<br>2. 戻り値を確認する |
| 試験データ | flash_size = 32768（sram_size計算上1024Bを超える値） |
| 期待値 | status == -1 (ERR_SRAM_FULL)・呼び出しは拒否される（Flash書き換え・リセットが発生しない） |
| 合格基準 | status == -1 かつ拒否後も呼び出し元が正常継続 |
| 環境 | E+R |
| 取得方法 | iap_call()戻り値をRC:xxxログで出力 |
| 判定方法 | status == -1 |
| ログ出力形式 | RC:UT-IAP-08 status=%d PASS/FAIL |

#### UT-IAP-09 WARN_NEAR_FULL / ERR_FRAM_FULL（FRAM使用量閾値）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-09 |
| 試験名 | FRAM CTXスタック使用量が12KB超でWARN、16KB超でERR_FRAM_FULLとなることの確認 |
| 対応仕様 | iap_context_switch_variable_design.md §2.5 |
| 前提条件 | UT-IAP-07 PASS |
| 試験手順 | 1. FRAM CTXスタック使用量が12KBを超えるまでiap_call()をネストする<br>2. その時点のstatusを確認する（WARN_NEAR_FULL期待）<br>3. さらに16KBを超えるまでネストし、拒否されることを確認する |
| 試験データ | FRAM使用量が12KB/16KBを跨ぐネスト段数（可変長のため段数はモジュールサイズ依存） |
| 期待値 | 12KB超過時 status==1(WARN_NEAR_FULL)、16KB超過見込み時 status==-2(ERR_FRAM_FULL) |
| 合格基準 | 両条件とも期待通りのstatusが返ること |
| 環境 | E+R |
| 取得方法 | 各段のiap_call()戻り値・fram_freeをRC:xxxログで出力 |
| 判定方法 | 12KB超でstatus==1、16KB超でstatus==-2 |
| ログ出力形式 | RC:UT-IAP-09 depth=%d status=%d fram_free=%d PASS/FAIL |

#### UT-IAP-10 パッチフィールドの256Bチャンク境界跨ぎ（キャリー経路直接検証・2026-07-15追加）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-10 |
| 試験名 | 被パッチ4Bフィールドが256Bチャンク境界を跨ぐ場合のキャリー処理検証 |
| 対応仕様 | iap_context_switch_variable_detail_design.md §7.7（境界跨ぎキャリー・Phase 2b 4b実装） |
| 前提条件 | UT-IAP-06/07 PASS |
| 試験手順 | 1. R_RISCV_32の被パッチワードをモジュール内オフセット≡254 (mod 256)へ決定的配置したcallee（p14・fixture_carry.S）を用意する<br>2. caller（p15）からiap_callし、calleeがdelta≠0のセルへload_and_patchされることでキャリー経路（現チャンク2B書込→次チャンク先頭2B上書き）を必ず通す<br>3. callee自身が「パッチ済みワード==fixture_targetの実行時アドレス」を照合する<br>4. iap_returnで復帰しstatus==0を確認する |
| 試験データ | fixture_carry.S: .balign 256 + .space 254 + .word fixture_target（254+4=258>256で跨ぎ確定）。R32パッチはold+deltaで全delta≠0において値が変化するため「無変化で偶然PASS」がない |
| 期待値 | marker==expected（キャリー2B+2Bが正しく合成される）・復帰status==0 |
| 合格基準 | marker照合PASSかつ復帰status=0 |
| 環境 | E+R |
| 取得方法 | calleeがRC:xxxログでmarker/expected（下位16bit）を出力 |
| 判定方法 | RC:UT-IAP-10 marker=... PASS と RC:UT-IAP-10 ret status=0 PASS の両方 |
| ログ出力形式 | RC:UT-IAP-10 marker=%d expected=%d PASS/FAIL ／ RC:UT-IAP-10 ret status=%d PASS/FAIL |

> **備考（フィクスチャ製作時の副次的発見・2026-07-15）**: `patch_table_gen.py`が
> リロケーションのaddendを無視しており、`シンボル+オフセット`参照（配列添字・
> 構造体フィールド等）でMISMATCHのfail-closedビルドエラーになる潜在バグを発見・
> 是正した（target=シンボル値+addendへ修正）。誤パッチではなくビルド停止側に
> 倒れていたため実害はなかった。是正後もp12のパッチテーブルはバイト一致（無回帰）。

#### UT-IAP-11 A-2（別アプリ間呼び出し）・ensure順序検証（2026-07-16追加・Task5②）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-11 |
| 試験名 | 別カタログのアプリ内サブプログラム呼び出し（A-2）の往復と、A-2分岐のensure順序検証 |
| 対応仕様 | iap_context_switch_variable_detail_design.md §7.10.2（A-2メタ入替）・§7.11.4（ensure順序潜在バグ是正） |
| 前提条件 | UT-IAP-06/07 PASS |
| 試験手順 | 1. caller（p16・catalog 0）と別アプリB（catalog 1）内のcallee（p12）を、カタログキャッシュ（0x1A000）にentry0/entry1をvalid配置してSPI Flashへ用意する<br>2. callerが`iap_call(res2, catalog_idx=1)`でA-2呼び出しする<br>3. `iap_call_impl`が`catalog_flash_addr(1)`→`a2_meta_swap_in`でapp Bメタを入替え、calleeをロード・ジャンプ<br>4. calleeがZone Cを書き換えた後`iap_return`する（復帰側もcaller=catalog 0のメタを書き戻す＝A-2の呼出/復帰両経路をカバー）<br>5. 復帰後にstatus==0とcaller側のg_test_val/pattern（Zone C）復元を確認する |
| 試験データ | 結合イメージは0x0600窓にbootuiバイト列を持ち、FRAM在席テーブル（0x1A800）はクリア済み。よって初回A-2で`ensure(A2)`がbootuiを追い出してA2をロードしてから`catalog_flash_addr`（.pool_2_a2内）を呼ぶ。**ネガティブコントロール（2026-07-16実施）**: `catalog_flash_addr`を`ensure`より前に呼ぶ順序バグを一時再導入するとUT-IAP-11がFAIL・他項目（05〜10）はPASSのまま＝本試験が順序バグを確実に捕捉することを実証済み |
| 期待値 | status==0かつg_test_val/pattern復元（A-2往復成立・ensure順序正常） |
| 合格基準 | RC:UT-IAP-11 status=0 PASS かつ RC:UT-IAP-11 a2 g_test_val=0x1234 PASS の両方 |
| 環境 | E+R |
| 取得方法 | callerがRC:xxxログでstatusとg_test_val復元可否を出力 |
| 判定方法 | 上記2ログの両PASS |
| ログ出力形式 | RC:UT-IAP-11 status=0 PASS ／ RC:UT-IAP-11 a2 g_test_val=0x1234 PASS/FAIL |

---

#### UT-IAP-12 常駐サブプログラム起動時ロード（§7.11・2026-07-16追加・Task5③）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-12 |
| 試験名 | アプリ起動時の常駐サブプログラム（オーケストレータ相当）を常駐領域0x1E00へdelta=-512でload+patchする経路の検証 |
| 対応仕様 | iap_context_switch_variable_detail_design.md §7.11.1（resident_load）・§7.11.4（resident_res_idガード先行判定）・§3.4.5（常駐領域0x1E00） |
| 前提条件 | UT-IAP-06/07 PASS（load_and_patch本経路が健全であること） |
| 試験手順 | 1. trigger app（p18）が`iap_run(0x098000)`を呼ぶ<br>2. `iap_run`が`flash_addr+0x2000`のメタ8KBを`FRAM_META_BASE`へ転送し、メタ+0x167Cの`resident_res_id`（=5）を読む<br>3. ≠0xFFFFのため`pool_ensure_loaded(A2)`→`resident_load(5)`を1回呼ぶ<br>4. `resident_load`が`find_resource(CODE\|5)`/`find_resource(PATCH\|5)`で常駐モジュール実体（p17・60B）とパッチテーブルを引き当て、`load_and_patch(master, 0x1E00, -512, size, patch)`で常駐領域へ配置<br>5. `iap_run`がApp Area書込後`soft_reset`。plugin が常駐領域0x1E00（512B）を`resident_result.bin`へ保存<br>6. フィクスチャ`resident_marker`の被パッチ語＝`resident_target`の実行時アドレスを照合 |
| 試験データ | フィクスチャ`fixture_resident.S`（p17）: `resident_marker`（モジュール内オフセット4）に自己参照語`.word resident_target`（R_RISCV_32）を置く。単一ビルド（0x2000基準）のリンク時値は`resident_target`のリンクアドレス0x2008だが、delta=-512のパッチ後は「`resident_target`の実行時アドレス」=RESIDENT_BASE(0x1E00)+モジュール内オフセット8=**0x1E08**になる。delta≠0のため値が必ず変化し「無変化で偶然PASS」がない。**ネガティブコントロール（対照・2026-07-16実施）**: `resident_res_id=0xFFFF`（常駐なし）で再実行すると`iap_run`のガードでload経路がスキップされ、0x1E00は未書込（0xFFFF_FFFF）のまま＝ガードが常駐指定なしアプリのプール非依存動作を保証することを実証 |
| 期待値 | 正例: `resident_result.bin`のオフセット4のLEワード==0x00001E08（=`resident_target`実行時アドレス）／負例: 同ワード==0xFFFFFFFF（ロード非発生） |
| 合格基準 | 正例PASS（0x1E08一致）かつ負例PASS（0xFFFFFFFF一致）の両方 |
| 環境 | E+R |
| 取得方法 | plugin が`soft_reset`検出時に0x1E00（512B）を`resident_result.bin`へダンプ。run_test_iap_var.shがnmで期待値を算出し照合 |
| 判定方法 | 正例/負例の両照合PASS |
| ログ出力形式 | UT-IAP-12 pos resident_result[4]=0x%08x expected=0x%08x PASS/FAIL ／ UT-IAP-12 neg resident_result[4]=0x%08x expected=0xffffffff PASS/FAIL |

#### UT-IAP-13 申告サイズ全振り幅スイープ（2026-07-18新設）

> **新設の背景**: 機構の分岐はほぼすべて申告サイズに従属する（size_cells 1〜32 → sram_size 64〜1024 → CTXエントリ長 98〜1058 → Zone Cウィンドウ → アロケータ窓幅）にもかかわらず、既存試験は cells=1,2,3,5,8 の5点しか踏んでいなかった（カバレッジ 5/32 ≈ 16%・docs/test_coverage_20260718.md §1.1）。**本スイープが初回実行で gp ABI 実バグ（gp_abi_design.md）を検出した。**

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-13 |
| 試験名 | 申告サイズ全振り幅スイープ（A-1往復・Zone Cクラス境界跨ぎ） |
| 対応仕様 | CAR-01_common_program_spec §5.4・iap_context_switch_variable_design §2.5/§7.6・gp_abi_design.md |
| 前提条件 | UT-IAP-06/07 PASS。同一ソース（phase23）を `-DSWEEP_SIZE` と Zone Cクラス別 ld で6点ビルド。 |
| 試験手順 | 各申告サイズ（1024/2048/4096/8192B）で: 1. caller が Zone C の静的変数へ既知パターンを設定<br>2. `iap_call(res2, 0xFF, 0)` で A-1 往復<br>3. 復帰後に status==OK・静的変数の復元・スタック退避値の生存を確認する |
| 試験データ | 申告 1024B(4セル/sram128/slot128)・2048B(8セル/sram256/slot256)・4096B(16セル/sram512/**slot512**=初使用)・**8192B(32セル/sram1024/slot1024=新設・ZONE_C_SIZE上限ちょうど・CTXエントリ最大1,058B)**。※256/512Bは試験アプリ実体(約600B)が申告を超え過小申告=setupがfail-closedするため対象外（cells=1,2の経路はcallee側p12/p20が既に踏んでいる） |
| 期待値 | 全点で `RC:UT-IAP-13 size=<N> ... PASS` |
| 合格基準 | 実行4点すべてPASS |
| 環境 | E（論理確認）/ **R（必須）** |

**本試験が実証したこと（2026-07-18）**: ①**cells=32（App Area全域申告）の正常系往復**が初めて検証された（Zone C上限ちょうど・`>`判定で通る境界）。②caller全域占有時は `iap_call` の最中に実行中の caller 自身が追い出されるが、**復帰時の sticky再ロードで自己回復する**（要: 本体の CODE/PATCH 登録＝§8.7規約3。ハーネス側 `--test iap_var` に res1 登録を追加）。③1024/2048B の当初FAILは**gp ABI 実バグの検出**だった（機構の静的変数30箇所がアプリの gp でずれ Zone C を無言破壊。詳細・修正は gp_abi_design.md）。

#### UT-IAP-14 callee セル境界のオフバイワン（2026-07-18新設・絨毯爆撃②）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-14 |
| 試験名 | callee 申告 flash_size のセル境界（255/256/257・511/512/513）往復 |
| 対応仕様 | iap_context_switch_variable_design §7.6（アロケータ）・load_and_patch のチャンク処理 |
| 前提条件 | UT-IAP-13 PASS。callee(p12・実体216B)の申告 flash_size のみを `--callee-decl-size` で変え、実体は0xFFパディング。 |
| 試験手順 | caller(p23_1024)が callee を A-1 呼び出し。callee 申告を 255/256/257/511/512/513B の6点で往復させる |
| 試験データ | `callee_cells=ceil(size/256)` と load_and_patch のチャンク数が 255→256→257 で 1→1→2 と遷移する境界 |
| 期待値 | 全6点で status=OK・caller の Zone C 復元 PASS |
| 合格基準 | 6点すべてPASS |
| 環境 | E（論理確認）/ **R（必須）** |

**踏む境界**: `callee_cells`（配置状態表の占有セル数）と `load_and_patch` の256Bチャンク数が、ちょうど256B単位で1つ増える点。現状の試験は p12(216B)・p14(1140B) など中途半端な実サイズしか踏んでおらず、`ceil()` の境界（256/512 ちょうど・その±1）が未検証だった。2026-07-18に6点すべてPASSを確認。

#### UT-IAP-15 過小申告→自己破壊のネガティブコントロール（2026-07-18新設・絨毯爆撃③）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-15 |
| 試験名 | caller 過小申告による自己破壊の実証（ネガティブコントロール） |
| 対応仕様 | iap_context_switch_variable_design §7.6（アロケータは申告セル数のみを信じる＝GIGO）・§8.7規約1（PC側ツールが申告≥実体を保証） |
| 前提条件 | setup の過小申告ガードを `--force-undersize` で**意図的にバイパス**する（本試験専用フラグ） |
| 試験手順 | caller実体260B（2セル・cell1先頭0x2100に番兵0xC0DEFACE）を申告256B（1セル）で登録。callerがcallee(p12)をA-1呼び出しし、復帰後に番兵を読む |
| 試験データ | 機構はcell1を空きと誤認→calleeをcell1へロード→番兵破壊。sticky再ロードはcaller_size_cells=1分（cell0のみ）しか復元しない |
| 期待値 | **番兵が破壊されていること**（`sentinel_destroyed PASS`）。無傷なら機構が何らか保護している＝理解と食い違うためFAIL |
| 合格基準 | 呼び出し前=0xC0DEFACE かつ 復帰後≠0xC0DEFACE |
| 環境 | E（論理確認）/ R不要（実機で自己破壊を起こす価値なし） |

**位置づけ**: 「破壊されたらPASS」の逆説試験。機構は申告≥実体を**検証しない**設計（検証には実体サイズ情報が必要だがディレクトリの申告値しか持たない）であり、安全網はPC側カタログツールの申告≥実体保証（§8.7規約1）と試験ハーネスのsetupガードのみ。本試験はその無防備さが**実際に自己破壊を招く**ことを実証し、ガードの必要性を実データで裏付ける。2026-07-18にPASS（番兵破壊を確認）。

---

### 2.1 起動シーケンス結合試験 (IT-BOOT)

#### IT-BOOT-01 通常起動フロー（App Areaあり）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-01 |
| 試験名 | boot_main() 通常起動：フォントロード→CART_READY→App Area起動 |
| 対応仕様 | boot_design §3（起動シーケンス）・CAR-01_common_program_spec §4 |
| 前提条件 | common_prog書き込み済み。App Area(0x2000)に試験バイナリ書き込み済み。GPIO2(PA2)=LOW。 |
| 試験手順 | 1. 電源ONまたはリセット<br>2. 200ms待機が完了することを確認する（ログ: MODULE_BOOT / EVENT_INIT_DONE後200ms）<br>3. GPIO2=LOWを確認する（通常起動フロー）<br>4. 恵梨沙フォント転送完了を確認する（ログ: MODULE_BOOT / 0x11）<br>5. CART_READY(PA1)がHIGHになることを確認する（実機：テスタ；エミュレータ：GPIO観測）<br>6. App Areaの試験バイナリが起動することを確認する |
| 試験データ | GPIO2=LOW（外部からPULLDOWN） |
| 期待値 | 200ms待機→フォントロード完了→PA1=HIGH→試験バイナリ起動（順序通り） |
| 合格基準 | 全シーケンスが順序通り完了すること |
| 環境 | E+R |

#### IT-BOOT-02 外部通信モード起動（GPIO2=HIGH）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-02 |
| 試験名 | boot_main() GPIO2=HIGH時の外部通信モード分岐確認 |
| 対応仕様 | boot_design §3・CAR-01_common_program_spec §5.5 |
| 前提条件 | common_prog書き込み済み。FRAM_SPI_MGR_ADDR(0x08000)にspi_flash_mgrバイナリが格納されていること。 |
| 試験手順 | 1. PA2（GPIO2）をHIGHにして電源ON<br>2. boot_main()が外部通信モードに分岐することを確認する<br>3. CART_READY(PA1)がHIGHになることを確認する<br>4. iap_restore_from_fram(FRAM_SPI_MGR_ADDR, ...)が呼び出されることを確認する（ログ） |
| 試験データ | GPIO2=HIGH（PA2プルアップ） |
| 期待値 | 通常起動フローに進まずspi_flash_mgr起動 |
| 合格基準 | spi_flash_mgr起動・PA1=HIGH |
| 環境 | R（実機のみ）<br>理由: エミュレータはGPIO INDRが常に0返しのためGPIO2=HIGH分岐を再現できない |

#### IT-BOOT-03 上下キー同時押しでブートローダセルフアップデートモード

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-03 |
| 試験名 | boot_main() BTN_UP+BTN_DOWN同時押しでセルフアップデートモード分岐 |
| 対応仕様 | boot_design §3・CAR-01_common_program_spec §5.6 |
| 前提条件 | common_prog書き込み済み。SPI Flash 0x018000にブートローダアップデートバイナリが格納されていること。 |
| 試験手順 | 1. 電源ON直後にBTN_UP(0x01)とBTN_DOWN(0x02)を同時押しの状態にする（エミュレータ：GPIO設定；実機：ジョイスティック操作）<br>2. TFTが赤画面（TFT_RED）になることを確認する<br>3. iap_run(BOOTLOADER_UPDATE_BLOCK_ADDR=0x018000)が呼び出されることを確認する（ログ） |
| 試験データ | keyscan_get()戻り値 = BTN_UP\|BTN_DOWN = 0x03 |
| 期待値 | TFT赤画面表示後にiap_run(0x018000)呼び出し |
| 合格基準 | TFT赤画面表示・iap_run()呼び出しのログ確認 |
| 環境 | E+R |

#### IT-BOOT-04 App Areaなし・FRAMバックアップあり（自己復元）＋バックアップ生成の往復

> **2026-07-17改訂（往復化・§8.4の解消）:**
> - **アドレス是正**: FRAMバックアップは `0x06000` ではなく **`FRAM_BACKUP_ADDR = 0x18000`**（8KB）。旧記述はFRAMマップ改訂（2026-07-13）以前の値。
> - **往復化**: 可変設計書§8.4が求める「バックアップ**生成**→復元の往復」を本試験で満たす（新IDは起こさない）。従来の定義は復元側のみで、生成側（`boot.c` `backup_app_area_to_fram`＝圧縮施策5の1トランザクション`fram_write`）が未検証だった。UT-IAP-04はsetupが事前に置いたバックアップからの復元しか見ていない。
> - **判定方法**: 「呼び出されることを確認」ではなく**成果物の完全一致で判定する**。`boot_puts()`は`#ifdef BOOT_LOG`で通常ビルドではno-opに展開されるため、ログ文字列は判定条件にできない。

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-04 |
| 試験名 | App Area→FRAMバックアップ生成と、App Areaが空の場合の自己復元（往復） |
| 対応仕様 | boot_design §3（バックアップ有効性判定）・iap_context_switch_variable_design §6.7 施策5・§8.4 |
| 前提条件 | IT-BOOT-01 PASS。`fram.bin`が2実行間で永続すること（Phase 1が`soft_reset`→`exit(0)`で正常終了しデストラクタが走ること）。 |
| 試験手順 | **Phase 1（生成）**: 1. App Areaにp1を置いて起動<br>2. bootが`app_area_has_program()`→`backup_app_area_to_fram()`でApp Area(0x2000・8KB)をFRAM 0x18000へ1トランザクション書き込み<br>3. p1が`iap_run()`→`soft_reset`→PficStubが`exit(0)`（`fram.bin`同期）<br>**Phase 2（復元）**: 4. App Areaを0xFFで結合し、**setupを走らせずPhase 1の`fram.bin`を引き継ぐ**<br>5. bootが`app_area_has_program()`=false→`fram_backup_valid()`=true→`iap_restore_from_fram(FRAM_BACKUP_ADDR, 0x2000, 8KB)`→`soft_reset` |
| 試験データ | App Area先頭4B=0xFFFFFFFF（Phase 2）・FRAM 0x18000先頭4B≠0xFFFFFFFF |
| 期待値 | ①`fram.bin[0x18000..+8KB]` == p1（0xFFパディング込み・**生成**）<br>②`iap_result.bin` == p1（同上・**復元**） |
| 合格基準 | ①②の両方が完全一致すること |
| 環境 | E+R |

**判定が成果物のみで成立する理由**: Phase 2はApp Areaを0xFFで結合しているため、**`iap_result.bin`がp1と完全一致する経路は「`fram_backup_valid()`→`iap_restore_from_fram()`でFRAMバックアップをApp Areaへ書き戻し→`soft_reset`」以外に存在しない**。したがって成果物自体が8bルート通過の証明になる。

**ネガティブコントロール（2026-07-17実施）**: FRAMバックアップの先頭4Bを0xFFへ書き換える（`fram_backup_valid()`をfalseにする）とPhase 2で`iap_result.bin`が生成されない＝復元経路に入らないことを確認済み。本試験がバックアップ有効性判定と復元経路を実際に見ていることを実証している。

---

### 2.2 フォントロード結合試験 (IT-FONT)

#### IT-FONT-01 SPI Flash→FRAMフォント転送

| 項目 | 内容 |
|---|---|
| 試験ID | IT-FONT-01 |
| 試験名 | 恵梨沙フォント SPI Flash(0x008000)→FRAM(0x13880) 55KB転送 |
| 対応仕様 | CAR-01_common_program_spec §7・§8（恵梨沙フォント）・boot_design §3 |
| 前提条件 | SPI Flash 0x008000に恵梨沙フォントが書き込まれていること。UT-SPI-06 PASS。 |
| 試験手順 | 1. boot_main()内のfont_load_from_flash()相当処理を実行する（または直接呼び出す）<br>2. 処理時間を計測する（SysTick使用）<br>3. FRAM 0x13880から先頭8バイトを読み出す<br>4. SPI Flash 0x008000の先頭8バイトと一致することを確認する |
| 試験データ | src=0x008000, dst=FONT_FRAM_ADDR=0x13880, size=55×1024=56,320B |
| 期待値 | FRAM 0x13880の内容がSPI Flash 0x008000の内容と一致すること<br>転送時間≒18ms（設計値） |
| 合格基準 | 先頭8バイト一致・転送時間の記録 |
| 環境 | E+R |

#### IT-FONT-02 フォントインデックスからFRAMアドレス算出確認

| 項目 | 内容 |
|---|---|
| 試験ID | IT-FONT-02 |
| 試験名 | フォントインデックスidxのFRAMアドレス = FONT_FRAM_ADDR + idx×8 の確認 |
| 対応仕様 | tft_oled_design §2（恵梨沙フォント描画・FONT_FRAM_ADDR=0x13880） |
| 前提条件 | IT-FONT-01 PASS |
| 試験手順 | 1. idx=0のFRAMアドレス(0x13880+0)から8バイト読み出す<br>2. idx=100のFRAMアドレス(0x13880+800)から8バイト読み出す<br>3. idx=6876（最大）のFRAMアドレス(0x13880+55008)から8バイト読み出す<br>4. 各読み出しがSPI Flash同一インデックスの値と一致することを確認する |
| 試験データ | idx=0, 100, 6876 |
| 期待値 | 各インデックスのFRAM読み出し値 = SPI Flash同インデックス値 |
| 合格基準 | 全3点一致 |
| 環境 | E+R |

#### IT-FONT-03 日本語文字TFT表示（フォントロード→描画エンドツーエンド）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-FONT-03 |
| 試験名 | フォントロード完了後に日本語文字をTFTに表示（エンドツーエンド） |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_draw_string）・tft_oled_design §2 |
| 前提条件 | IT-FONT-01 PASS・UT-TFT-01 PASS |
| 試験手順 | 1. 「ラテカ」（3文字）の恵梨沙フォントインデックスを取得する<br>2. tft_draw_string_elysia(indices, 3, 10, 10, TFT_WHITE, TFT_BLACK)を呼び出す<br>3. TFTに「ラテカ」が表示されることを目視確認する |
| 試験データ | text=「ラテカ」のインデックス配列[3] |
| 期待値 | TFTに「ラテカ」が正しく表示されること（目視） |
| 合格基準 | 3文字が目視で認識できること |
| 環境 | E+R |

---

### 2.3 リソースロード結合試験 (IT-RSRC)

#### IT-RSRC-01 load_resource() 基本動作（画像）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-RSRC-01 |
| 試験名 | load_resource() 画像リソース(res_type=0x1)ロード |
| 対応仕様 | CAR-01_common_program_spec §5.2（load_resource）・cartridge_master §1.8 |
| 前提条件 | IT-FONT-01 PASS（FRAM_META_BASEにメタ情報転送済み）。SPI Flashに試験用画像リソースが書き込まれていること。 |
| 試験手順 | 1. load_resource(0x1001, FRAM_META_BASE + 0x4000)を呼び出す（画像種別0x1・ID=001）<br>2. 関数がハングなく完了することを確認する<br>3. 転送先FRAMアドレスから先頭4バイトを読み出し、リソースデータと一致することを確認する |
| 試験データ | res_type_id=0x1001, fram_dest_addr=FRAM_META_BASE+0x4000 |
| 期待値 | FRAM転送先の先頭4バイト = SPI Flash上の同リソース先頭4バイト |
| 合格基準 | 4バイト一致・ハングなし |
| 環境 | E+R |

#### IT-RSRC-02 load_resource() type_start[] テーブル参照確認

| 項目 | 内容 |
|---|---|
| 試験ID | IT-RSRC-02 |
| 試験名 | FRAM_META_BASE + META_TYPE_START_OFF から type_start[type] を正しく読み出せることの確認 |
| 対応仕様 | iap_design §1（META_TYPE_START_OFF=0x1000）・load_resource_design §3 |
| 前提条件 | IT-RSRC-01 前提（FRAM_META_BASE+0x1000にtype_start[]が転送済み） |
| 試験手順 | 1. fram_read(FRAM_META_BASE + 0x1000, buf, 32)を呼び出す（16種別×2B）<br>2. buf[0x00:0x01]（種別0x0・プログラム）の値を確認する<br>3. buf[0x02:0x03]（種別0x1・画像）の値を確認する<br>4. 値がSPI Flash上のメタ情報領域の対応オフセット（0x1000）と一致することを確認する |
| 試験データ | addr=FRAM_META_BASE+0x1000=0x24880, len=32 |
| 期待値 | 16種別×2B=32Bの値がSPI Flash上の値と一致すること |
| 合格基準 | 32バイト一致 |
| 環境 | E+R |

#### IT-RSRC-03 load_resource() ディレクトリ拡張対応

| 項目 | 内容 |
|---|---|
| 試験ID | IT-RSRC-03 |
| 試験名 | 512エントリ超のアプリでディレクトリ拡張ブロック(res_type=0xE)から検索できることの確認 |
| 対応仕様 | cartridge_master §1.8（ディレクトリ拡張） |
| 前提条件 | IT-RSRC-01 PASS。SPI FlashにブロックN+1以降のリソースが存在するダミーアプリが格納されていること。 |
| 試験手順 | 1. res_type_id=0x1_200（ID=0x200=512番目の画像）でload_resource()を呼び出す<br>2. 拡張ブロック(res_type=0xE)を辿って正しいリソースに到達することを確認する<br>3. 転送先FRAMに正しいデータが書き込まれることを確認する |
| 試験データ | res_type_id=0x1200, fram_dest_addr=FRAM_META_BASE+0x5000 |
| 期待値 | 拡張ディレクトリを辿ってリソースが正しく転送されること |
| 合格基準 | 転送データが期待値と一致すること |
| 環境 | E+R（SPI Flashに拡張ダミーデータが必要） |

---

### 2.4 IAPコンテキストスイッチ結合試験 (IT-IAP)

> **注意：IT-IAP-01〜04は実機での実施が必須。iap_ctx.S（RISC-Vアセンブリスタブ）の実機動作が未検証（CAR-01_common_program_spec §10 PENDING #3）。エミュレータでの論理確認後、実機で必ず追確認すること。**

#### IT-IAP-01 iap_call() → iap_return() 1段ネスト

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-01 |
| 試験名 | iap_call()でモジュール起動し iap_return() で呼び出し元に戻るエンドツーエンド |
| 対応仕様 | CAR-01_common_program_spec §5.4（パターンA-1）・iap_design §2 |
| 前提条件 | UT-IAP-02 PASS。SPI Flash上に試験モジュールが格納済み。 |
| 試験手順 | 1. 呼び出し元プログラム内でグローバル変数g_call_count=0に初期化する<br>2. iap_call(module_addr, 0xFE, IAP_CALL_INTERNAL)を呼び出す<br>3. 試験モジュールがg_call_count++してiap_return()を呼び出す<br>4. 呼び出し元でg_call_countが1であることを確認する<br>5. FRAMのコンテキストスタック深さカウンタ(CTX_STACK_DEPTH_ADDR=0x10003・可変長版で変更後のアドレス)が0に戻っていることを確認する |
| 試験データ | g_call_count初期値=0, IAP_CALL_INTERNAL=0 |
| 期待値 | g_call_count==1・CTX深さ==0（復帰後） |
| 合格基準 | 全条件一致 |
| 環境 | E（論理確認）/ **R（必須・iap_ctx.S実機検証）** |

#### IT-IAP-02 iap_call() 多重ネスト確認

> **2026-07-11更新:** 「最大7段」は固定2KB版のFRAM容量律速による固定値であり、可変長版ではCTX_STACK_MAXという固定上限自体が廃止され、FRAM 16KB残量に対する動的チェックに置き換わっている（`iap_context_switch_variable_design.md` §2.5）。本試験のネスト段数は固定7段ではなく、試験モジュールのsram_size次第で変わる（同一サイズが並ぶ場合の理論上限は同§2.5の容量ワークスルー表を参照）。段数の具体的な選定は試験実施時に確定する。

> **2026-07-16更新（Task5④・Phase 2準拠へ再設計）:**
> - **用語**: 「深さ（ネスト段数）」を**同時呼び出しサブプログラム数（多重度）**へ改める。`CTX_STACK_DEPTH_ADDR`が数えているのは関数コールスタックの深さではなく、Zone Cスナップショットを FRAM に退避したまま復帰を待っている**中断中サブプログラムの数**であり、後者が機構の実態に忠実なため（ユーザー指摘）。
> - **段数=10に確定**: 律速は FRAM 容量（16KB・1エントリ≈98B＝理論上100段超）ではなく **App Area のセル数（32セル）**。実測でモジュール(p20)=472B（申告512B=2セル）・L0(p19)=748B（申告768B=3セル）となり、**2×10+3=23セル ≤ 32セル**で10多重度が成立する（実測値・2026-07-16）。
> - **方式**: 本試験は**初版（41795fb・2026-07-07）の原案＝「自分自身を再帰呼び出しする試験モジュール」方式へ回帰**したもの。2026-07-10の実装（p7/p8/p9の3バイナリ・リセット連鎖・2段）は原案から逸れており、かつ Phase 2 の直接ジャンプ方式（単一Spike実行）と非整合だったため廃止した（逸脱の理由は記録に残っていない）。
> - **10重登録の原理**: 配置状態表の検索キーは`(catalog_idx, res_id)`の組（`iap.c` `plc_find`）。したがって**同一のCODE実体・パッチテーブルを res_id 2〜11 の10組で登録する**と、各 res_id が別リージョンとして別セルへ配置され、10個が同時在席する。パッチテーブルはモジュール内オフセット基準で位置独立のため共用でき、Flash追加コストはディレクトリ20エントリのみ。各段は FRAM の多重度を読んで`next = RES_NEST_BASE + 多重度`を決めるため、バイナリは1本で足りる（Zone Cに状態を持たない＝A-2ステートレス規約 案A）。

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-02 |
| 試験名 | IAPコンテキストスタック多重ネストと順次復帰（可変長版・Phase 2準拠） |
| 対応仕様 | CAR-01_common_program_spec §5.4・iap_context_switch_variable_design.md §2.5・§7.6（アロケータ） |
| 前提条件 | IT-IAP-01 PASS。SPI Flash上に自分自身を再帰呼び出しする試験モジュール（p20）が格納され、res_id 2〜11 の10組でディレクトリ登録済み（全エントリが同一実体を指す）。L0（p19）は res1・cell0 に事前登録。 |
| 試験手順 | 1. L0（多重度0）が iap_call() を呼ぶ<br>2. 各段が`CTX_STACK_DEPTH_ADDR`(0x22003)を読み、多重度が1,2,…と増加することを確認する<br>3. 多重度が目標値に達した段（最奥）は iap_call せず iap_return する<br>4. 各段が復帰後に多重度が自段の値へ戻ったこと・status=OK を確認して iap_return する<br>5. L0まで戻り多重度=0であることを確認する<br>6. **L0が配置状態表（`PLACEMENT_ENTRY_BASE`）をダンプし、在席リージョンを出力する** |
| 試験データ | 目標同時呼び出しサブプログラム数=10（res2〜11）。L0=res1・実測748B/申告768B=3セル。モジュール=実測472B/申告512B=2セル |
| 期待値 | ①入場トレース: 多重度 0→10 の単調増加<br>②復帰: 10行すべて PASS（多重度1〜9のモジュール9本＋L0 1本。最奥段は iap_call しないため復帰行なし）<br>③配置状態表: **11リージョン（res1〜11）が重複なく在席**・count=11 |
| 合格基準 | ①②③すべて成立すること |
| 環境 | E（論理確認）/ **R（必須）** |

**判定②が本質である理由:** リージョンは iap_return では削除されず、新規配置時の重複追い出し（`plc_evict`）でしか消えない。したがって全段復帰後の表に11個が非重複で残っていることは、「ピーク時に10個が**同時に**在席し、1個も追い出されなかった」ことの証明になる。トレースだけでは「10回呼んだ」ことしか示せず、同時在席の証明にならない。本判定により、`allocate()`が実行中・中断中モジュールのセルを追い出さないこと（`iap.c`の caller touch）も10個が競合する状態で実経路検証される。

**判定にシリアル出力を用いる理由（試験環境上の制約）:** FRAMスタブは`sync()`をデストラクタでのみ呼ぶ（`fram_stub.cc`）。本試験は最終段がハルトし`timeout`でSpikeを終了させるためデストラクタが走らず、**実行後の`fram.bin`には書き戻されない**（setup時の状態のまま残る）。したがって配置状態表は実行後のファイルからではなく、アプリ自身がシリアルへダンプしたものを判定する。UT-IAP-12が`resident_result.bin`で判定できるのは`iap_run`→soft_reset→`exit(0)`でプラグインの保存経路を通るためであり、経路が異なる。

#### IT-IAP-03 iap_call() IAP引数・戻り値エリア確認（A-1）

> **2026-07-16注記（追跡漏れの発覚）:** 本試験と IT-IAP-04 は、設計書初出（41795fb・2026-07-07）から IT-IAP-01/02 と**同時に定義されていたにもかかわらず、試験結果報告書に一度も記載されていなかった**（PASS表・未実施試験セクション・履歴行のいずれにも行がない）。実装（`run_test_it.sh`）も IT-BOOT-01/03・IT-IAP-01/02 の4項目のみで、除外を判断した記録はどこにも残っていない。**定義と追跡台帳が最初から接続されていなかった**ことが原因と判断し、Task5 の残タスクへ復帰させた（ユーザー承認・2026-07-16）。

> **2026-07-16改訂（Phase 2準拠＋引数エリア移設）:**
> - **アドレス**: 引数エリアは**メタ領域内（+0x1680/+0x16C0）から FRAM管理領域（0x1A900/0x1AA00・各256B）へ移設**された。理由は`iap_design.md`「IAP引数・戻り値エリア」参照（メタ領域は`a2_meta_swap_in`がFlash原本から再ロードするキャッシュであり、アプリが書き込む場所ではないため）。
> - **シグネチャ**: 旧記述の`iap_call(module_addr, 0xFE, IAP_CALL_INTERNAL)`はPhase 1の**アドレス指定**方式。Phase 2は`iap_call(res_id, catalog_idx, call_type)`のリソースID指定へ移行済み。A-1は`catalog_idx=0xFF`（自アプリ）で表す。

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-03 |
| 試験名 | IAP A-1引数・戻り値エリア（FRAM管理領域）経由のモジュール間データ受け渡し |
| 対応仕様 | iap_design「IAP引数・戻り値エリア」・iap_context_switch_variable_design §7.1（FRAMレイアウト） |
| 前提条件 | IT-IAP-01 PASS。FRAM_META_BASEにメタ情報転送済み。 |
| 試験手順 | 1. caller が `FRAM_IAP_A1_ARG`(0x1A900) に64バイトの試験データを書き込む<br>2. `iap_call(res_id, 0xFF, 0)` でA-1呼び出しする<br>3. callee が `FRAM_IAP_A1_ARG` から64バイトを読み出し、各バイトを加工して `FRAM_IAP_A1_RET`(0x1AA00) へ書き込み、`iap_return()` する<br>4. 復帰後に caller が `FRAM_IAP_A1_RET` の64バイトを読み、期待値と照合する |
| 試験データ | 引数: 64バイト固定パターン（`i ^ 0x5A`）／期待戻り値: calleeが各バイトに`+1`した値 |
| 期待値 | 戻り値エリアの64バイトが期待値と完全一致すること |
| 合格基準 | 64バイト全一致（`RC:IT-IAP-03 a1_arg ret=OK`） |
| 環境 | E（論理確認）/ **R（必須）** |

**本試験が検証する機構的性質**: 引数・戻り値エリアはcommon_progが一切参照しない領域（純粋な取り決め）であり、機構は「触らないこと」で正しさを担保している。本試験は、A-1のコンテキストスイッチ（Zone C退避/復元・calleeのload_and_patch・配置状態表更新）を跨いで**管理領域の内容が保存されること**を実経路で確認する。加工（+1）を挟むのは、calleeが実際に読んで書いたことを保証するため（無変化のまま通る偶然のPASSを排除する）。

#### IT-IAP-04 iap_call() A-2 別アプリ間呼び出し（引数受け渡し・メタ復元）

> **2026-07-16改訂（Phase 2準拠＋引数エリア移設）:** 旧定義はPhase 1のままで、**現行機構では成立しない**内容だった。是正点は3つ:
> 1. **A-2引数エリアがメタ内では成立しない**（本試験の核心）。旧手順「アプリBが`FRAM_META_BASE+0x1700`を確認」は不可能だった。`a2_meta_swap_in`が呼出時にアプリBのメタ8KBをFlash原本から転送し、引数を塗り潰すため。戻り値（+0x1740）も復帰時のメタ書き戻しで同様に破壊される。→ 引数エリアを**FRAM管理領域（A-2引数0x1AB00 / A-2戻り値0x1AC00・各256B）へ移設**して成立させた。
> 2. **シグネチャ**: 旧記述`iap_call(0x09C000, ID_APP_A, IAP_CALL_EXTERNAL)`はアドレス指定（Phase 1）。Phase 2は`iap_call(res_id, catalog_idx, call_type)`。
> 3. **A-2の判定条件**: Phase 2では`call_type`ではなく **`eff_catalog != cur.catalog_idx`**（`iap.c:1045`）でA-2へ分岐する。旧試験データ`call_type=IAP_CALL_EXTERNAL(1)`は機構の実態と無関係（UT-IAP-11は`call_type=0`のままA-2経路を通っている）。
> 4. **期待値**: Phase 2に`caller_flash_addr`は存在しない（`ContextEntry v3`の`caller_catalog_idx`＋カタログ逆引きに置換）。→「呼び出し元アプリのメタが復元されたこと」を直接観測する形へ改める。

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-04 |
| 試験名 | IAP A-2（別アプリ間呼び出し）の引数・戻り値受け渡しと呼び出し元メタの復元 |
| 対応仕様 | CAR-01_common_program_spec §5.4（パターンA-2）・iap_context_switch_variable_design §7.10.2（A-2機構）・iap_design「IAP引数・戻り値エリア」 |
| 前提条件 | IT-IAP-01・UT-IAP-11 PASS。SPI Flash上にアプリA（catalog 0・0x098000）とアプリB（catalog 1・0x0A0000）が格納され、カタログキャッシュにentry0/1がvalid配置済み。 |
| 試験手順 | 1. アプリA（caller）が `FRAM_IAP_A2_ARG`(0x1AB00) に64バイトの試験データを書き込む<br>2. `iap_call(res_id, catalog_idx=1, 0)` でA-2呼び出しする（`iap_call_impl`が`catalog_flash_addr(1)`→`a2_meta_swap_in`でアプリBのメタへ入替え）<br>3. アプリB内のcalleeが `FRAM_IAP_A2_ARG` から64バイトを読み、加工して `FRAM_IAP_A2_RET`(0x1AC00) へ書き、`iap_return()` する<br>4. 復帰時に`prep`がアプリAのメタを書き戻す<br>5. 復帰後にアプリAが ①`FRAM_IAP_A2_RET` の64バイトを期待値と照合 ②**`FRAM_META_BASE+0x1000`の`type_start[0]`がアプリA自身の値に戻っていること**を確認 |
| 試験データ | caller=アプリA(catalog 0)／callee=アプリB(catalog 1)内のサブプログラム。引数: 64バイト固定パターン（`i ^ 0x5A`）／期待戻り値: calleeが各バイトに`+1`した値。メタ判定値: アプリAの`type_start[0]`=0x0020（アプリBは0x00E0） |
| 期待値 | ①戻り値64バイトが期待値と完全一致 ②`type_start[0]`==0x0020（アプリAのメタへ復元済み） |
| 合格基準 | ①②の両方が成立すること |
| 環境 | E（論理確認）/ **R（必須・iap_ctx.S実機検証）** |

**判定②を設けた理由（UT-IAP-11との差分）**: UT-IAP-11（A-2往復・ensure順序）はA-2の呼出/復帰の両経路を通すが、検証しているのは`status==0`とcaller側Zone Cの復元であり、**呼び出し元のメタが実際に書き戻されたことを観測していない**。メタが戻っていなければ、復帰後のアプリAは`find_resource`でアプリBのディレクトリを引いてしまう（自分のリソースを解決できない）。`type_start[0]`はアプリA/Bで異なる値を持つため、この1点の照合で復元を決定的に判定できる。**この観測点があるため、IT-IAP-04はUT-IAP-11の証跡代替では閉じられない**（2026-07-16・当初は証跡代替可と見立てたが誤りだった）。

**本試験が守る実利用者**: 日本語入力Wnn（`CAR-01_japanese_input`）はA-2公開サービスとして設計されており、A-2引数エリアでひらがな文字列を受け取る（実測見積もり約40B）。旧配置のままではWnn呼び出しは機構が引数を破壊するため成立しなかった。本試験はその受け渡し経路そのものを検証する。

#### IT-IAP-05 App Area溢れ時のLRU追い出しと復帰時sticky再ロード

> **2026-07-16新設（Task5⑥）:** IT-IAP-02（多重度10・23セル＝追い出しなし）の対になる試験。**App Areaを意図的に溢れさせ**、`allocate()`のLRU追い出しと、`sram_restore_from_fram_prep`の**sticky再ロード**（`iap.c:1236-1252`）を実経路で検証する。IT-IAP-02とはハーネス・試験モジュールを共用し、`NEST_TARGET`（`-D`で上書き）と`--nest-target/--allow-cell-overflow`のみが異なる。

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-05 |
| 試験名 | App Area溢れ時のLRU追い出しと復帰時sticky再ロード |
| 対応仕様 | iap_context_switch_variable_design.md §7.6（アロケータ）・§3.5.6（表更新順序）・CAR-01_common_program_spec §5.4 |
| 前提条件 | IT-IAP-02 PASS。IT-IAP-02と同一のハーネス（`--test nest`）で、目標多重度=15・`--allow-cell-overflow`を指定。**L0(res1)がCODE/PATCHディレクトリに登録済みであること**（下記「本試験が要求する規約」参照） |
| 試験手順 | 1. 多重度15まで iap_call() をネストさせる（**2×15+3=33セル > 32セル**でApp Areaが必ず溢れる）<br>2. 各段の入場・復帰は IT-IAP-02 と同一<br>3. L0まで復帰後、L0が配置状態表をダンプする |
| 試験データ | 目標同時呼び出しサブプログラム数=15（res2〜16）。L0=res1・実測748B/申告768B=3セル。モジュール=実測472B/申告512B=2セル → 必要33セル > 32セル |
| 期待値 | ①入場トレース: 多重度 0→15 の単調増加（溢れても呼び出し自体は成功する）<br>②復帰: 15行すべて PASS<br>③配置状態表: **count < 16（＝追い出しが発生した証明）**・重複なし・**res1(L0)が在席**・使用セル数 ≤ 32 |
| 合格基準 | ①②③すべて成立すること |
| 環境 | E（論理確認）/ **R（必須）** |

**判定の論理:** 16リージョン（L0+モジュール15個）は33セルを要し32セルに物理的に収まらないため、**`count < 16` は追い出しが起きたことの算術的な証明**になる。その上で全段が正しく復帰し、かつL0が自身の出力を出せていることが、**追い出されたモジュールがsticky再ロードで復元されたことの証明**になる（復元されていなければ、そのセルには別モジュールのコードが乗ったままで、戻った瞬間に別物を実行するため正しい出力を出せない）。

**追い出しの犠牲者がL0になる理由（実測で確認済み）:** `iap_call_impl`は **callee配置（`plc_register`でseq N採番）→ caller touch（seq N+1）** の順に採番する（`iap.c:1108-1111` → `1150-1155`）。L0は最初の1回だけtouchされてから中断し続けるため、`lru_seq`が全体最小（=3）となり、`allocate()`のコスト最小窓としてL0のセルが選ばれる。

**本試験が要求する規約（＝実機で顕在化しうる要件・ユーザー決定 案A・2026-07-16）:**
sticky再ロードは `find_resource(CODE|caller_res_id)` と `find_resource(PATCH|caller_res_id)` を引き、**いずれか不在なら無限ループでハングする**（`iap.c:1241` / `1246`）。CODE側はアプリ本体が「自アプリCODEディレクトリの ResourceEntry[0]」である前提（`iap_placement_init`・`iap.c:1296-1297`）により保証される。**しかしアプリ本体のパッチテーブル（種別`0x6000|main_res_id`）の存在を保証する仕組みは無い。** 本体はcell0・delta=0で動作するため通常運用ではパッチテーブルを必要としないが、**追い出されて再ロードされる瞬間にだけ要求される**。したがって「**App Areaを使い切る可能性のあるアプリは、本体にもパッチテーブルを登録すること**」がPC側カタログ生成ツールの要件となる（パッチ内容はdelta=0で実質no-opだが、テーブルの存在自体が必要）。§8.7の開発者ガイド転記項目。

**実測結果（2026-07-16・log_it_20260717_084457.txt）:** 予測通りの経過を確認した。①res16配置時に31セル使用済み・空き1セルでL0(seq最小)が追い出される → ②復帰時にL0が不在と判定されcell0へsticky再ロード → ③その再ロード（3セル）が今度はres16を追い出す → **最終状態: res1在席・res16のみ消失・count=15・使用31/32セル・重複なし**。

---

## 3. L3 システム試験

### 3.1 表示システム試験 (ST-DISP)

試験プログラム: test_app_phase2（CAR-01_emulator_dev §17 A-2-9参照）

#### T-1〜T-5 TFT基本表示試験

| 試験ID | 試験名 | 手順概要 | 期待値 | 合格基準 |
|---|---|---|---|---|
| ST-DISP-T-1 | 初期化→白画面 | tft_init()後に画面確認 | TFT全面白 | 目視：全面白色 |
| ST-DISP-T-2 | 単色塗りつぶし（赤/緑/青） | tft_fill(RED/GREEN/BLUE)各500ms表示 | 各色で全画面塗りつぶし | 目視：3色正確 |
| ST-DISP-T-3 | 文字描画（英数字） | 「HELLO WORLD」をtft_draw_string_elysiaで表示 | 座標(10,10)に英数文字列 | 目視：文字が読める |
| ST-DISP-T-4 | 日本語表示（恵梨沙フォント） | 「ラテカ」等の日本語をtft_draw_string_elysiaで表示 | 座標(10,30)に日本語文字列 | 目視：文字が読める |
| ST-DISP-T-5 | ウィンドウ描画（矩形） | tft_set_window(20,20,100,100)+tft_write_pixel()で矩形 | 指定矩形のみ描画 | 目視：矩形の範囲・位置が正確 |

#### B-1〜B-8 TFT描画速度ベンチマーク

各ベンチマークでFPS・ms/frame・合計時間をTFT右上に常時表示し、FRAMログに記録する（RC:xxx形式またはFRAM 2号ログ）。

| 試験ID | 試験名 | 計測内容 | 参考目標値 |
|---|---|---|---|
| ST-DISP-B-1 | 全画面塗りつぶし速度 | tft_fill()のFPS計測（100回） | 記録のみ（参考値なし・初回計測） |
| ST-DISP-B-2 | 矩形連続描画 | 大200×200/中100×100/小32×32 各100回のms/回 | 記録のみ |
| ST-DISP-B-3 | zlib圧縮画像展開・転送速度 | 展開ms + 全画面転送ms + KB/s | 記録のみ |
| ST-DISP-B-4 | RGB565全画面直接転送速度 | 240×240×2B=115,200Bの転送ms | 記録のみ |
| ST-DISP-B-5 | テキスト縦スクロール（英数） | 1行/フレームのFPS | 記録のみ |
| ST-DISP-B-6 | テキスト縦スクロール（日本語） | 恵梨沙フォント・1行/フレームのFPS | 記録のみ |
| ST-DISP-B-7 | FPSゲーム模擬描画 | レイキャスト風垂直線240本のFPS | 記録のみ |
| ST-DISP-B-8 | シューティング模擬描画 | 背景スクロール+敵10+弾20のFPS | 記録のみ |

**合格基準:** 各ベンチマークが完了し、数値がFRAMログに記録されていること（ハングなし）。数値の妥当性は初回実機計測後に基準値を設定する。

---

### 3.2 ストレージ速度計測試験 (ST-STOR)

#### P-1〜P-7 メモリ間スループット計測

SysTick(0xE000F000)を使用してサイクル数計測（48MHz = 1サイクル ≒ 20.83ns）。

| 試験ID | 転送経路 | 計測サイズ | 期待スループット（参考） |
|---|---|---|---|
| ST-STOR-P-1 | USB→SPI Flash（HID経由） | 4KB（1セクタ） | 〜36KB/s（minichlink実績値） |
| ST-STOR-P-2 | SPI Flash→FRAM | 1KB / 4KB / 16KB | 24MHz SPI理論値の実測 |
| ST-STOR-P-3 | SPI Flash→内蔵Flash（IAP） | 9KB（App Area） | 〜117ms（設計値） |
| ST-STOR-P-4 | SPI Flash→内蔵SRAM | 256B / 1KB / 2KB | 24MHz SPI実測 |
| ST-STOR-P-5 | FRAM→内蔵Flash（IAP） | 9KB | 〜105ms（Flash消去含む） |
| ST-STOR-P-6 | FRAM→内蔵SRAM | 256B / 1KB / 2KB | 24MHz SPI実測 |
| ST-STOR-P-7 | 内蔵Flash→内蔵SRAM（memcpy） | 256B / 1KB | 48MHz CPU実測 |

**合格基準:** 全計測が完了し数値がFRAMログに記録されること。

---

### 3.3 IAPコンテキストスイッチ速度計測試験 (ST-IAP)

#### CS-1〜CS-8 コンテキストスイッチ所要時間計測

| 試験ID | 計測項目 | 設計見込み | 合格基準 |
|---|---|---|---|
| ST-IAP-CS-1 | iap_call()総所要時間（呼び出し〜新モジュール起動） | ≒117ms | 計測完了・記録 |
| ST-IAP-CS-2 | SRAM退避時間（2KB→FRAM・sram_save_to_fram） | ≒0.7ms | 計測完了・記録 |
| ST-IAP-CS-3 | 内蔵Flash消去時間（9KB・64Bセクタ×144回） | ≒80ms | 計測完了・記録 |
| ST-IAP-CS-4 | 内蔵Flash書き込み時間（SPI Flash→内蔵Flash 9KB） | ≒26ms | 計測完了・記録 |
| ST-IAP-CS-5 | iap_return()総所要時間（戻り処理〜呼び出し元再開） | ≒115ms | 計測完了・記録 |
| ST-IAP-CS-6 | App Area書き戻し時間（SPI Flash→内蔵Flash 9KB） | ≒26ms | 計測完了・記録 |
| ST-IAP-CS-7 | メタ情報復元時間（SPI Flash→FRAM 6KB） | ≒2ms | 計測完了・記録 |
| ST-IAP-CS-8 | SRAM復元時間（FRAM→内蔵SRAM 2KB・sram_restore_from_fram） | ≒0.7ms | 計測完了・記録 |

**合格基準:** 各計測が完了しFRAMログに記録されること。ボトルネック（CS-3 Flash消去）が80ms以内であること（最悪値は仕様書§5.3の290ms以内）。

---

## 4. 試験ログ設計

### 4.1 LogEntry構造体（test_environment_spec §5準拠）

```c
typedef struct {
    uint32_t timestamp_ms;  // 起動からの経過ms（SysTick由来）
    uint8_t  module_id;     // モジュールID（下記定義参照）
    uint8_t  event_id;      // イベントID（下記定義参照）
    uint8_t  payload[10];   // 任意データ（アドレス・値・計測結果等）
} LogEntry;  // 計16B

// FRAM 2号（CS: PD6）先頭から格納
// 256KB / 16B = 16,384エントリ蓄積可能
```

### 4.2 モジュールID定義（test_environment_spec §5.2準拠）

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
| 0xFE | 試験アプリ（test_app） |
| 0xFF | システム（boot_main等） |

### 4.3 イベントID定義（test_environment_spec §5.3準拠）

| イベントID | 内容 | payload内容 |
|---|---|---|
| 0x00 | 初期化開始 | なし |
| 0x01 | 初期化完了 | payload[0]=所要ms(下位8bit) |
| 0x10 | 読み出し開始 | payload[0..3]=アドレス |
| 0x11 | 読み出し完了 | payload[0..1]=サイズ, payload[2..3]=所要us |
| 0x20 | 書き込み開始 | payload[0..3]=アドレス |
| 0x21 | 書き込み完了 | payload[0..1]=サイズ, payload[2..3]=所要us |
| 0xE0 | エラー発生 | payload[0]=エラーコード |
| 0xF0 | 試験ステップ開始 | payload[0..1]=試験ID（ASCII 2文字） |
| 0xF1 | 試験ステップ PASS | payload[0..1]=試験ID |
| 0xF2 | 試験ステップ FAIL | payload[0..1]=試験ID, payload[2]=失敗箇所コード |
| 0xF8 | 計測値記録 | payload[0..1]=試験ID, payload[2..5]=計測値(us or cycle) |

### 4.4 試験開始・中間・終了ログフォーマット

```c
// 試験開始ログ（各L1/L2試験の先頭で出力）
void log_test_start(uint8_t module_id, const char test_id[2]) {
    LogEntry e = {
        .timestamp_ms = systick_ms(),
        .module_id    = 0xFE,  // 試験アプリ
        .event_id     = 0xF0,
        .payload      = {test_id[0], test_id[1], module_id, 0, ...}
    };
    log_write(&e);
}

// 中間状態ログ（転送・計測の節目で出力）
void log_step_perf(const char test_id[2], uint32_t elapsed_us) {
    LogEntry e = {
        .timestamp_ms = systick_ms(),
        .module_id    = 0xFE,
        .event_id     = 0xF8,
        .payload      = {test_id[0], test_id[1],
                         (elapsed_us >> 24) & 0xFF,
                         (elapsed_us >> 16) & 0xFF,
                         (elapsed_us >> 8)  & 0xFF,
                         elapsed_us & 0xFF, 0, 0, 0, 0}
    };
    log_write(&e);
}

// 合格ログ
void log_test_pass(const char test_id[2]) {
    LogEntry e = {
        .timestamp_ms = systick_ms(),
        .module_id    = 0xFE,
        .event_id     = 0xF1,
        .payload      = {test_id[0], test_id[1], 0, ...}
    };
    log_write(&e);
}

// 失敗ログ
void log_test_fail(const char test_id[2], uint8_t fail_point) {
    LogEntry e = {
        .timestamp_ms = systick_ms(),
        .module_id    = 0xFE,
        .event_id     = 0xF2,
        .payload      = {test_id[0], test_id[1], fail_point, 0, ...}
    };
    log_write(&e);
}
```

### 4.5 RC:xxx形式ログ出力フォーマット（USB HID / FRAM 2号共通）

```
RC:TST START UT-SPI-01
RC:TST DONE  UT-SPI-01 PASS 0us
RC:TST DONE  UT-SPI-03 PASS 125us
RC:TST START IT-FONT-01
RC:PERF P2 FLASH-TO-FRAM 4096B 1850us
RC:TST DONE  IT-FONT-01 PASS 18200us
RC:TST DONE  UT-KEY-02 FAIL point=02
RC:BENCH B1-FPS 23.4fps
RC:BENCH CS3-ERASE 78500us
```

---

## 5. 試験環境

### 5.1 エミュレータ環境セットアップ

**前提:** CAR-01_emulator_dev_20260622.md Phase 0〜2 完了済み（A-0-1〜A-2-9）

```bash
# Phase 0: Spike + car01_plugin ビルド確認
spike --isa=rv32ec \
  -m0x0:0x4000,0x20000000:0x800 \
  --extlib=emulator/car01_plugin/build/libcar01_plugin.so \
  --extension=car01_plugin \
  src/UIAPduino/common_prog/common_prog.elf

# Phase 1: SPI Flash / FRAMファイル確認
ls -la spi_flash.bin   # 16MB（プロジェクトルートに配置）
ls -la fram.bin        # 512KB（プロジェクトルートに配置）

# Phase 2: panel.html確認
python3 emulator/panel/serial_monitor.py \
  --frame tft_frame.bin --port 8765 &
# ブラウザで emulator/panel/panel.html を開いて
# TFT表示・ジョイスティックUIを確認

# 試験用spi_flash.bin作成（恵梨沙フォント・ダミーカタログ・試験バイナリ含む）
python3 tools/dummy_data_gen.py \
  --app src/UIAPduino/test_app_phase2/test_app.bin \
  --output spi_flash.bin

# 恵梨沙フォント・画像リソースの書き込みは別途 tools/image_convert.py 等で実施
```

**試験用spi_flash.bin構成（2026-06-25確定）：**
生成ツール：tools/dummy_data_gen.py

アドレス配置（アプリブロック16KB単位）：
```
0x008000: 恵梨沙フォント（64KB確保）
0x098000: test_app.bin（16KB枠）
0x09C000: 画像1（アニメ）29ブロック=116KB
0x0B8000: 画像2（写真）29ブロック=116KB
0x0D4000: 画像3（パワポ）29ブロック=116KB
0x0F0000: 画像4（書籍）29ブロック=116KB
```

設計方針：
試験リソースもアプリブロック単位で管理する。
試験アプリだけ例外扱いしない。
カタログ管理方式（IT-RSRC）の試験完了後に
正式なカタログエントリ登録に移行する予定。

**エミュレータ試験実行手順:**

```bash
# 試験アプリをcommon_prog内蔵Flash起動で実行
spike --isa=rv32ec \
  -m0x0:0x4000,0x20000000:0x800 \
  --extlib=emulator/car01_plugin/build/libcar01_plugin.so \
  --extension=car01_plugin \
  src/UIAPduino/common_prog/common_prog.elf \
  2>&1 | grep "^RC:"

# ログをファイルに保存
spike --isa=rv32ec \
  -m0x0:0x4000,0x20000000:0x800 \
  --extlib=emulator/car01_plugin/build/libcar01_plugin.so \
  --extension=car01_plugin \
  src/UIAPduino/common_prog/common_prog.elf \
  2>&1 | grep "^RC:" | tee log_emulator_$(date +%Y%m%d_%H%M%S).txt

# ログ解析
python3 tools/log_viewer.py --input log_emulator_*.txt
```

**ITEMリスト（エミュレータで実施可能な試験）:**

| 試験群 | 試験ID | エミュレータ実施可否 |
|---|---|---|
| UT-SPI | 01〜06 | ✅ Phase 1完了後 |
| UT-FRAM | 01〜04 | ✅ Phase 1完了後 |
| UT-FLASH | 01〜05 | ✅ Phase 1完了後 |
| UT-TFT | 01〜05 | ✅ Phase 2完了後（panel.html目視） |
| UT-KEY | 01・03〜05 | ✅ Phase 2完了後（A-2-6スタブ） |
| UT-KEY-02 | — | TBD（ボタン構成未確定） |
| UT-TIM | 01 | ✅ Phase 1完了後（A-1-6 SysTickスタブ） |
| UT-IAP | 01〜04 | ✅ Phase 1完了後（論理確認） |
| IT-BOOT | 01〜04 | ✅ Phase 2完了後 |
| IT-FONT | 01〜03 | ✅ Phase 2完了後 |
| IT-RSRC | 01〜03 | ✅ Phase 2完了後 |
| IT-IAP | 01〜04 | ✅ 論理確認のみ（実機追確認必須） |
| ST-DISP T/B | 全 | ✅ Phase 2完了後（FPS値は参考） |
| ST-STOR P | 全 | ⚠️ エミュレータ値は参考のみ |
| ST-IAP CS | 全 | ⚠️ 論理確認のみ（実測値は実機で取得） |

---

### 5.2 実機環境セットアップ

**必要ハードウェア（test_environment_spec §1.1準拠）:**

```
□ UIAPduino Pro Micro CH32V003 V1.4
□ SPI FRAM 1号（MB85RS4MTYPF 256KB or 512KB）CS=PA1
□ SPI FRAM 2号（ログ用・試験環境専用）CS=PD6
□ SPI Flash W25Q128JV 16MB CS=PA2
□ TFT液晶 ST7789 1.3inch 240×240px CS=PC0 DC=PC4 RST=PD5
□ キー入力デバイス（ATtiny1604 / I2C、boards_D §4.3）
□ ブレッドボード・電源モジュール（3.3V）
□ USB-Cケーブル（データ通信対応・書き込み専用）
□ Windows PC（pip install hid済み）
```

**配線確認チェックリスト（test_environment_spec §1.2準拠）:**

```
□ SPI バス: PC5(SCK) PC6(MOSI) PC7(MISO) 共通配線
□ CS_EINK/TFT: PC0
□ CS_YMF: PD2
□ CS_FLASH: PA2  ← 旧PD3から変更（2026-06-09確定）
□ CS_FRAM（1号）: PA1  ← 旧PD4から変更（2026-06-11確定）
□ CS_FRAM（2号・ログ用）: PD6
□ TFT DC: PC4  ← 旧PC3から変更
□ TFT RST: PD5  ← 旧PC4から変更
□ I2C SDA: PC1 / SCL: PC2（キー入力デバイス接続）
□ CART_READY(PA1): テスタで測定可能な状態
□ GPIO2(PA2=PA2兼用: CS_FLASH共用注意）: ← PA2はCS_FLASHと同じピン
```

> **注意：PA2はCS_FLASHとGPIO2を兼用するピン。boot_main()の200ms待機中のみGPIO2として使用し、その後CS_FLASHとして使用する。試験時は200ms以内にGPIO2確認が終わることを前提とする。**

**実機セットアップ手順（概略）:**

```bash
# Step 1: ブートローダ書き換え確認
python3 tools/write_program.py --file bootloader_confirm.bin
# → オレンジLEDが100ms点滅することを目視確認

# Step 2: 共通プログラム書き込み
python3 tools/write_program.py --file common_prog.bin

# Step 3: SPI Flash試験資材書き込み（Phase 2.5）
# GPIO2=HIGH（PA2をVCCに接続）して電源ON → spi_flash_mgr起動
python3 tools/dummy_data_gen.py --with-font --output spi_flash.bin
python3 tools/write_program.py --spi-flash --addr 0x000000 --file spi_flash.bin
# GPIO2=LOWに戻して再起動 → アプリ選択UI確認

# Step 4: 試験アプリ書き込み
python3 tools/write_program.py --file test_app_phase2.bin --addr 0x2000

# Step 5: 試験実行（電源はブレッドボード電源に切り替え）
# TFT表示・ログを目視確認

# Step 6: ログ吸い上げ
python3 tools/log_capture.py --output log_$(date +%Y%m%d_%H%M%S).json

# Step 7: 解析
python3 tools/log_viewer.py --input log_*.json
```

**実機で実施必須の試験（エミュレータでは論理確認のみ）:**

| 試験ID | 理由 |
|---|---|
| IT-IAP-01〜04 | iap_ctx.S（RISC-Vアセンブリスタブ）の実機動作が未検証（PENDING #3） |
| ST-STOR P-1〜P-7 | 実測スループット値の取得（エミュレータ値は参考のみ） |
| ST-IAP CS-1〜CS-8 | 実際のFlash消去時間など実機固有の計測 |

---

---

## 3. WDT・ハードウェアプロファイル試験

> 対応設計書: `common_prog/wdt_hwprofile_design.md`

### 3.0 WDT集中管理機能試験 (UT-WDT)

#### UT-WDT-01 IWDG 初期化確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-WDT-01 |
| 試験名 | boot_main() でのIWDG初期化（プリスケーラ・リロード値設定）確認 |
| 対応仕様 | wdt_hwprofile_design §1.2 |
| 前提条件 | common_prog.bin がエミュレータで起動できること |
| 試験手順 | 1. Spike起動（SpikeにIWDGスタブを追加し PSCR/RLDR/CTLR への書き込みを記録）<br>2. boot_main()実行<br>3. IWDG_PSCR = 6、IWDG_RLDR = 0x4DF、CTLR = 0xCCCC（起動）の書き込みシーケンスを確認 |
| 試験データ | 期待PSCR=6、期待RLDR=0x4DF |
| 期待値 | boot_main()先頭でIWDGが正しいパラメータで初期化されること |
| 合格基準 | PSCR/RLDR/CTLRへの正しい書き込みシーケンスが確認できること |
| 環境 | E（Spike・IWDGスタブ要追加） |
| 取得方法 | IWDGスタブに書き込みログ機能を追加し、car01_pluginのログとして出力 |
| 判定方法 | ログに `[iwdg] PSCR=6 RLDR=0x4DF START` が出ること |
| ログ出力形式 | `[iwdg] PSCR=N RLDR=0xNNN START` |

#### UT-WDT-02 rtc_system_tick() レジスタ書き込み確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-WDT-02 |
| 試験名 | rtc_system_tick() が IWDG->CTLR = 0xAAAA を書き込むことの確認 |
| 対応仕様 | wdt_hwprofile_design §1.3 |
| 前提条件 | なし |
| 試験手順 | 1. L2テスト（Unity）でIWDG->CTLR をモック変数に差し替える<br>2. rtc_system_tick() を呼び出す<br>3. モック変数に 0xAAAA が書き込まれたことを確認する |
| 試験データ | なし |
| 期待値 | IWDG->CTLR == 0xAAAA |
| 合格基準 | モック変数の値が 0xAAAA であること |
| 環境 | E（L2 Unity）|
| 取得方法 | Unity テストの ASSERT_EQUAL_HEX32(0xAAAA, mock_iwdg_ctlr) |
| 判定方法 | アサーション通過 |
| ログ出力形式 | `UT-WDT-02 PASS` |

#### UT-WDT-03 IWDG リセット後のシステムメニュー強制帰還確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-WDT-03 |
| 試験名 | RCC->RSTSCKR IWDGRSTF bit セット時に app_loader_run() へ遷移することの確認 |
| 対応仕様 | wdt_hwprofile_design §2.2 |
| 前提条件 | UT-WDT-01 PASS。App Areaに起動可能なバイナリが存在すること |
| 試験手順 | 1. SpikeにRCCスタブを追加し、RCC->RSTSCKR の初期値に IWDGRSTF(bit28)=1 を設定<br>2. boot_main()実行<br>3. App Area のバイナリが起動せず、app_loader_run()（アプリ選択UI）が表示されることを確認する |
| 試験データ | RCC->RSTSCKR 初期値 = 0x10000000（IWDGRSTF bit28 セット） |
| 期待値 | App Area バイナリが起動しない。アプリ選択UIのログが出ること |
| 合格基準 | "RC:UT-WDT-03 iwdg_reset=1 PASS" がログに出ること |
| 環境 | E（Spike・RCCスタブ要追加） |
| 取得方法 | 試験アプリをApp Areaに配置（起動すれば"FAIL"を出力）。app_loader_run()に試験用ログ出力を追加 |
| 判定方法 | App Area バイナリの "FAIL" が出ず、システムメニュー側のログが出ること |
| ログ出力形式 | `RC:UT-WDT-03 iwdg_reset=1 PASS` |

#### UT-WDT-04 長時間処理ループ内でのWDTクリア呼び出し確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-WDT-04 |
| 試験名 | keyscan_wait()・ram_flash_write_chunk() 内でrtc_system_tick()が呼ばれることの確認 |
| 対応仕様 | wdt_hwprofile_design §1.4 |
| 前提条件 | UT-WDT-02 PASS |
| 試験手順 | 1. L2テスト（Unity）でIWDG->CTLRをモック変数に差し替える<br>2a. keyscan_wait()を呼び出し、1回ボタン押下を模擬して返す<br>2b. ram_flash_write_chunk()を呼び出す<br>3. 各呼び出し中にモック変数への 0xAAAA 書き込みが発生したことを確認する |
| 試験データ | ボタン押下模擬: keyscan_get()スタブが1回後にBTN_OKを返す |
| 期待値 | keyscan_wait() / ram_flash_write_chunk() 実行中にIWDG->CTLR = 0xAAAA が書き込まれること |
| 合格基準 | 各関数実行後にモック呼び出しカウント >= 1 |
| 環境 | E（L2 Unity）|
| 取得方法 | Unityテスト内で mock_iwdg_write_count を確認 |
| 判定方法 | mock_iwdg_write_count >= 1 |
| ログ出力形式 | `UT-WDT-04 PASS` |

---

### 3.1 ハードウェアプロファイル試験 (UT-PROFILE)

#### UT-PROFILE-01 RtcHardwareProfile 固定アドレス配置確認

| 項目 | 内容 |
|---|---|
| 試験ID | UT-PROFILE-01 |
| 試験名 | RtcHardwareProfile が 0x08D8 に配置されていることの確認 |
| 対応仕様 | wdt_hwprofile_design §3.1 |
| 前提条件 | common_prog.bin がビルドされていること |
| 試験手順 | 1. common_prog.elf の 0x08D8 のシンボルを確認（`nm` または `objdump`）<br>2. Spike実行中に試験アプリが 0x08D8 を読み出してログ出力する<br>3. 値が 0xFFFFFFFF でないことを確認する |
| 試験データ | アドレス = 0x08D8 |
| 期待値 | 0x08D8 に rtc_hw_profile シンボルが配置されており、値が 0xFFFFFFFF でないこと |
| 合格基準 | `RC:UT-PROFILE-01 word0=0xNNNNNNNN (not 0xFFFFFFFF) PASS` |
| 環境 | E（Spike）|
| 取得方法 | 試験アプリで `*(uint32_t*)0x08D8` を読み出してUSARTログ出力 |
| 判定方法 | word0 != 0xFFFFFFFF |
| ログ出力形式 | `RC:UT-PROFILE-01 word0=0xNNNNNNNN PASS` |

#### UT-PROFILE-02 RtcHardwareProfile ビットフィールド値確認（CAR-01 TFT機種）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-PROFILE-02 |
| 試験名 | RtcHardwareProfile の各フィールドが CAR-01 UIAPduino TFT 機種の正しい値であることの確認 |
| 対応仕様 | wdt_hwprofile_design §3.4 |
| 前提条件 | UT-PROFILE-01 PASS |
| 試験手順 | 1. 試験アプリが 0x08D8 を `RtcHardwareProfile*` にキャストして各フィールドを読み出す<br>2. 期待値と照合する |
| 試験データ | 期待値: device_class=2, profile_ver=1, model_code=1, hw_revision=1, disp_type=2, disp_width=240, disp_height=240, key_type=1, audio_type=2, storage_size=1 |
| 期待値 | 全フィールドが上記期待値と一致すること |
| 合格基準 | `RC:UT-PROFILE-02 mismatch=0 PASS` |
| 環境 | E（Spike）|
| 取得方法 | 試験アプリで各ビットフィールドを読み出して期待値と比較、不一致数をログ出力 |
| 判定方法 | mismatch == 0 |
| ログ出力形式 | `RC:UT-PROFILE-02 mismatch=N PASS/FAIL` |

---

## 6. 変更履歴

| 日付 | 内容 |
|---|---|
| 2026-06-24 | 初版作成 |
| 2026-06-25 | 試験実施順序追加（UT-KEY-03〜05を最優先）・全L1試験項目に取得方法・判定方法・ログ出力形式を追加・UT-FLASH-03/04・IT-BOOT-02をR（実機のみ）に変更・SPI1 CTLR1エミュレート追加に伴いUT-SPI-01をE+Rに変更 |
| 2026-07-07 | Wave 2 試験項目（§1.8〜§1.11）追加・UT-TFT-04/05実装内容反映・UT-FLASH-02判定基準修正・UT-TIM-01エミュレータ特性注記追加 |
| 2026-07-07 | UT-CATALOG-04・UT-RESOURCE-04〜06 追加（第4部乖離解消: フォントインデックスLE読み出し・typeビットフィールド抽出・cellオフセット算術・サイズ3B LEデコード） |
| 2026-07-07 | UT-CATALOG-02を「空カタログ（先頭0xFF→count==0）」に修正・UT-CATALOG-05〜07追加（途中終端打ち切り・全件削除・上限打ち切り） |
| 2026-07-07 | UT-RESOURCE-07〜09追加（線形サーチ2件目マッチ・dir_cellオフセット・size=0）・UT-FONT-02定数名修正（FRAM側）・試験実施順序にWave 2試験群を追記・§1.8〜1.11に取得方法フィールド追加 |
| 2026-07-07 | UT-KEY-02をTBDから5方向スティック正式定義に更新（boards_D §4.3準拠）・D基板MCU見直し注記追加・UT-IAP-01〜04をL1§1.7からL2§2.0（最優先）に移設・§1.8〜1.11を§1.7〜1.10に繰り上げ・実施順序の12番をL2 IAP最優先表記に更新 |
| 2026-07-10 | §3（WDT・ハードウェアプロファイル試験）追加: UT-WDT-01〜04・UT-PROFILE-01〜02（計6件）。目論見書レビュー結果を反映し、アドレス0x08D8・挿入箇所6個所・RCC_RSSTCKRレジスタ名を正式定義 |
