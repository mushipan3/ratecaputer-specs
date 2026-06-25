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
| ジョイスティック（開発段階暫定） | seesaw ADA-5743 / I2C 0x50 | test_environment_spec §9 |
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
8. UT-IAP-01〜04（IAP）
9. L2結合試験（IT-BOOT〜IT-IAP）
10. L3システム試験（ST-DISP〜ST-IAP）

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
| 対応仕様 | CAR-01_common_program_spec §7確定版・iap_design §3（CTX_STACK_BASE） |
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

#### UT-FLASH-02 フォント領域読み出し

| 項目 | 内容 |
|---|---|
| 試験ID | UT-FLASH-02 |
| 試験名 | SPI Flash恵梨沙フォント領域（0x008000）先頭8バイト読み出し |
| 対応仕様 | CAR-01_common_program_spec §8（恵梨沙フォント64KB固定） |
| 前提条件 | UT-SPI-04 PASS。SPI Flashに恵梨沙フォントが書き込まれていること（エミュレータはspi_flash.binをロード済み） |
| 試験手順 | 1. flash_read(0x008000, buf, 8)を呼び出す<br>2. フォントインデックス0（文字コード最小）のビットマップ8バイトを読み出す<br>3. 値が0xFF×8（未書き込み）でないことを確認する |
| 試験データ | addr=0x008000, len=8 |
| 期待値 | 読み出し完了、buf[0]〜buf[7]が0xFF×8以外（有効フォントデータ） |
| 合格基準 | 読み出し完了かつ0xFF×8でないこと |
| 環境 | E（spi_flash.binロード必須）/ R（Phase 2.5完了後） |
| 取得方法 | flash_read(0x008000,buf,8)を呼ぶ |
| 判定方法 | buf[0]〜buf[7]が全て0xFFでない |
| ログ出力形式 | RC:UT-FLASH-02 buf=%02X%02X%02X%02X not_all_ff=%d PASS/FAIL |

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

#### UT-TFT-04 英数文字描画

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-04 |
| 試験名 | tft_draw_char_elysia() 英数文字1文字描画 |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_draw_string）・tft_oled_design §2 |
| 前提条件 | UT-TFT-01 PASS。FRAMに恵梨沙フォントが展開済みであること（UT-FONT-01相当の前処理）。 |
| 試験手順 | 1. 恵梨沙フォントの'A'のインデックスを取得する（エミュレータ：既知インデックス使用）<br>2. tft_draw_char_elysia(idx_A, 0, 0, TFT_WHITE, TFT_BLACK)を呼び出す<br>3. 座標(0,0)に8×8の白い'A'が描画されることを目視確認する |
| 試験データ | font_idx=（'A'のインデックス）, x=0, y=0, fg=TFT_WHITE, bg=TFT_BLACK |
| 期待値 | 'A'の字形が座標(0,0)に8×8pxで描画されること |
| 合格基準 | 字形が目視で認識できること |
| 環境 | E+R（フォントロード後） |
| 取得方法 | 恵梨沙フォントのインデックスを指定してtft_draw_char_elysia(idx,0,0,DISP_WHITE,DISP_BLACK)。SDL2ウィンドウで目視確認。使用するインデックス値をRC:xxxログで出力すること |
| 判定方法 | 指定文字が目視で認識できる |
| ログ出力形式 | RC:UT-TFT-04 idx=%d 目視確認要 |

#### UT-TFT-05 文字列描画

| 項目 | 内容 |
|---|---|
| 試験ID | UT-TFT-05 |
| 試験名 | tft_draw_string_elysia() ASCII文字列「HELLO」描画 |
| 対応仕様 | CAR-01_common_program_spec §5.2（disp_draw_string） |
| 前提条件 | UT-TFT-04 PASS |
| 試験手順 | 1. "HELLO"の各文字の恵梨沙フォントインデックス配列を準備する<br>2. tft_draw_string_elysia(indices, 5, 10, 10, TFT_WHITE, TFT_BLACK)を呼び出す<br>3. 座標(10,10)から「HELLO」が9px間隔（8px+1px）で描画されることを目視確認する |
| 試験データ | indices[5]={'H','E','L','L','O'の各インデックス}, len=5, x=10, y=10 |
| 期待値 | 「HELLO」が左詰め9px間隔で描画されること（合計幅=5×9-1=44px） |
| 合格基準 | 文字列が目視で認識できること |
| 環境 | E+R |
| 取得方法 | 5文字分のインデックス配列を用意してtft_draw_string_elysia(indices,5,0,0,DISP_WHITE,DISP_BLACK)。SDL2ウィンドウで目視確認 |
| 判定方法 | 5文字が9px間隔で目視認識できる |
| ログ出力形式 | RC:UT-TFT-05 len=5 目視確認要 |

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

#### UT-KEY-02 ボタン入力読み取り（TBD）

| 項目 | 内容 |
|---|---|
| 試験ID | UT-KEY-02 |
| 試験名 | keyscan_get() 全ボタン入力確認 |
| 対応仕様 | keyscan_design §2（BTN_UP/DOWN/LEFT/RIGHT/CENTER） |
| 前提条件 | UT-KEY-01 PASS |
| 試験手順 | **TBD**（ボタン構成未確定）<br>確定済み：UP(0x01)・DOWN(0x02)・LEFT(0x04)・RIGHT(0x08)・BTN_OK/CENTER(0x10)の5入力<br>追加ボタン（A・B または A・B・X・Y）はハードウェア仕様確定後に更新する |
| 試験データ | **TBD** |
| 期待値 | **TBD**（確定済み5入力のみ先行定義可） |
| 合格基準 | **TBD** |
| 環境 | **TBD** |

**暫定（確定済み5入力のみ）:**

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
| 期待値 | |t0 - t1| ≒ 48,000サイクル（43,200〜52,800の範囲） |
| 合格基準 | サイクル数が43,200〜52,800の範囲内 |
| 環境 | E（スタブ動作確認）/ R（実測値記録） |
| 取得方法 | SysTick->CNT（0xE000F00C）を連続2回読んで差分を計算: t0=*(volatile uint32_t*)0xE000F00C、t1=*(volatile uint32_t*)0xE000F00C、diff=t1-t0 |
| 判定方法 | 43200 <= diff <= 52800 |
| ログ出力形式 | RC:UT-TIM-01 t0=%u t1=%u diff=%u PASS/FAIL |

---

### 1.7 IAP単体試験 (UT-IAP)

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

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-02 |
| 試験名 | iap_call()によるモジュール起動と戻り確認 |
| 対応仕様 | CAR-01_common_program_spec §5.4（パターンA-1）・iap_design §2 |
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

| 項目 | 内容 |
|---|---|
| 試験ID | UT-IAP-03 |
| 試験名 | iap_call()→iap_return()でSRAM 2KB全体が正しく復元されることの確認 |
| 対応仕様 | CAR-01_common_program_spec §5.4（ContextEntry sram_image[2048]） |
| 前提条件 | UT-IAP-02 PASS |
| 試験手順 | 1. SRAM内に既知パターン（0xDEADBEEF等）を複数箇所に配置する<br>2. iap_call()でモジュールを呼び出す（モジュールはSRAMを書き換えてiap_return()） <br>3. iap_return()後に各箇所のパターンが元に戻っていることを確認する |
| 試験データ | pattern: SRAM[0x200]=0xDEAD, SRAM[0x400]=0xBEEF, SRAM[0x7F0]=0xCAFE |
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
| 試験データ | fram_src=0x06000, app_base=0x1E00, app_size=8192 |
| 期待値 | FRAMバックアップから起動すること |
| 合格基準 | バックアップバイナリが起動すること |
| 環境 | E+R |
| 取得方法 | FRAM_BACKUP_ADDR(0x06000)に試験バイナリを書いてiap_restore_from_fram()呼び出し→リセット後に起動したバイナリがRC:UT-IAP-04 PASSをログ出力 |
| 判定方法 | リセット後のログにPASSが出る |
| ログ出力形式 | RC:UT-IAP-04 boot_from_fram=1 PASS |

---

## 2. L2 結合試験

### 2.1 起動シーケンス結合試験 (IT-BOOT)

#### IT-BOOT-01 通常起動フロー（App Areaあり）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-01 |
| 試験名 | boot_main() 通常起動：フォントロード→CART_READY→App Area起動 |
| 対応仕様 | boot_design §3（起動シーケンス）・CAR-01_common_program_spec §4 |
| 前提条件 | common_prog書き込み済み。App Area(0x1E00)に試験バイナリ書き込み済み。GPIO2(PA2)=LOW。 |
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

#### IT-BOOT-04 App Areaなし・FRAMバックアップあり（自己復元）

| 項目 | 内容 |
|---|---|
| 試験ID | IT-BOOT-04 |
| 試験名 | App Areaが空の場合にFRAM 0x06000から自己復元 |
| 対応仕様 | boot_design §3（バックアップ有効性判定） |
| 前提条件 | App Areaが空（先頭4バイト=0xFF）。FRAM 0x06000に有効なバックアップが存在すること。 |
| 試験手順 | 1. App Areaを0xFFで上書きする（または未書き込み状態）<br>2. 電源ON<br>3. boot_main()がFRAMバックアップを検出することを確認する（先頭4Bが0xFF以外）<br>4. iap_restore_from_fram(FRAM_BACKUP_ADDR, ...)が呼び出されることを確認する |
| 試験データ | App Area先頭4B=0xFFFFFFFF, FRAM 0x06000先頭4B≠0xFFFFFFFF |
| 期待値 | FRAMバックアップから自己復元・起動 |
| 合格基準 | FRAMバックアップから起動すること |
| 環境 | E+R |

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
| 試験手順 | 1. 呼び出し元プログラム内でグローバル変数g_call_count=0に初期化する<br>2. iap_call(module_addr, 0xFE, IAP_CALL_INTERNAL)を呼び出す<br>3. 試験モジュールがg_call_count++してiap_return()を呼び出す<br>4. 呼び出し元でg_call_countが1であることを確認する<br>5. FRAMのコンテキストスタック深さカウンタ(CTX_STACK_DEPTH_ADDR=0x10010)が0に戻っていることを確認する |
| 試験データ | g_call_count初期値=0, IAP_CALL_INTERNAL=0 |
| 期待値 | g_call_count==1・CTX深さ==0（復帰後） |
| 合格基準 | 全条件一致 |
| 環境 | E（論理確認）/ **R（必須・iap_ctx.S実機検証）** |

#### IT-IAP-02 iap_call() 最大ネスト7段確認

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-02 |
| 試験名 | IAPコンテキストスタック最大7段ネストと順次復帰 |
| 対応仕様 | CAR-01_common_program_spec §5.4（最大ネスト7段） |
| 前提条件 | IT-IAP-01 PASS。SPI Flash上に自分自身を再帰呼び出しする試験モジュールが格納済み（最大7段）。 |
| 試験手順 | 1. 深さカウンタを観測しながら7段のiap_call()を呼び出す<br>2. 各段でCTX_STACK_DEPTH_ADDR(0x10010)の値が1〜7と増加することを確認する<br>3. 7段目からiap_return()を順次呼び出す<br>4. 各段でカウンタが6→0と減少することを確認する<br>5. 最終的に呼び出し元に戻り、カウンタが0であることを確認する |
| 試験データ | 最大ネスト深さ=7段 |
| 期待値 | カウンタ: 呼び出し時1→7（増加）・復帰時6→0（減少）・最終=0 |
| 合格基準 | カウンタ変化が期待通りで最終的に0に戻ること |
| 環境 | E（論理確認）/ **R（必須）** |

#### IT-IAP-03 iap_call() IAP引数・戻り値エリア確認

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-03 |
| 試験名 | IAP A-1引数エリア(FRAM_META_BASE+0x1680)経由でモジュール間データ受け渡し |
| 対応仕様 | iap_design §3（IAP引数・戻り値エリア）・cartridge_master §1.8（プログラムブロック後半メタ情報） |
| 前提条件 | IT-IAP-01 PASS。FRAM_META_BASEにメタ情報転送済み。 |
| 試験手順 | 1. FRAM_META_BASE+0x1680（IAP A-1引数エリア）に64バイトの試験データを書き込む<br>2. iap_call(module_addr, 0xFE, IAP_CALL_INTERNAL)を呼び出す<br>3. 試験モジュールがFRAM_META_BASE+0x1680から64バイトを読み出し、FRAM_META_BASE+0x16C0（戻り値エリア）に加工した値を書き込む<br>4. iap_return()後に呼び出し元がFRAM_META_BASE+0x16C0の値を確認する |
| 試験データ | 引数: 64バイト固定パターン、期待戻り値: モジュールが加工した値 |
| 期待値 | 戻り値エリアに期待値が書き込まれていること |
| 合格基準 | 戻り値一致 |
| 環境 | E（論理確認）/ **R（必須）** |

#### IT-IAP-04 iap_call() A-2 別アプリ間呼び出し

| 項目 | 内容 |
|---|---|
| 試験ID | IT-IAP-04 |
| 試験名 | IAP A-2（IAP_CALL_EXTERNAL）による別アプリ呼び出しと復帰 |
| 対応仕様 | CAR-01_common_program_spec §5.4（パターンA-2）・iap_design §2 |
| 前提条件 | IT-IAP-01 PASS。SPI Flash上に別アプリ（異なるcaller_flash_addr）として格納された試験モジュールが存在すること。 |
| 試験手順 | 1. アプリA（SPI Flash 0x098000）からiap_call(0x09C000, ID_APP_A, IAP_CALL_EXTERNAL)を呼び出す<br>2. アプリB（0x09C000）がFRAM_META_BASE+0x1700（A-2引数）を確認してiap_return()を呼び出す<br>3. アプリAに戻り、FRAMのcaller_flash_addrが0x098000に復元されていることを確認する |
| 試験データ | caller=0x098000, callee=0x09C000, call_type=IAP_CALL_EXTERNAL(1) |
| 期待値 | アプリAへ復帰・caller_flash_addr=0x098000 |
| 合格基準 | 復帰確認・caller_flash_addr一致 |
| 環境 | E（論理確認）/ **R（必須・iap_ctx.S実機検証）** |

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
□ seesaw Mini I2C Gamepad QT（ADA-5743）I2C 0x50（開発段階暫定）
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
□ I2C SDA: PC1 / SCL: PC2（seesaw ADA-5743接続）
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
python3 tools/write_program.py --file test_app_phase2.bin --addr 0x1E00

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

## 6. 変更履歴

| 日付 | 内容 |
|---|---|
| 2026-06-24 | 初版作成 |
| 2026-06-25 | 試験実施順序追加（UT-KEY-03〜05を最優先）・全L1試験項目に取得方法・判定方法・ログ出力形式を追加・UT-FLASH-03/04・IT-BOOT-02をR（実機のみ）に変更・SPI1 CTLR1エミュレート追加に伴いUT-SPI-01をE+Rに変更 |
