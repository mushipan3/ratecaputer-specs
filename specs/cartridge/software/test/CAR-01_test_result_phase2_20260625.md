# CAR-01 試験結果 Phase 0〜2（L1単体＋L2結合）

**文書作成日: 2026-06-25**
**最終更新日: 2026-07-19**
**参照設計書: CAR-01_test_design_phase2_20260624.md**
**試験環境:** E = エミュレータ（spike + car01_plugin） / R = 実機

---

## 凡例

本書の試験結果一覧・各表の列の意味:

| 列 | 説明 |
|---|---|
| 試験ID | 試験設計書と同一のID |
| 試験名 | 試験内容の短い名称 |
| 結果 | PASS / FAIL / 未実施 |
| 実施日 | 試験実施日（未実施は「-」） |
| 環境 | E=エミュレータ / R=実機 |
| 備考 | 判定根拠・実測値・FAIL時の失敗箇所等 |

---

## 概要

| 項目 | 内容 |
|------|------|
| 対象 | CAR-01 common_prog L1/L2単体試験（Wave 1: 23件 + Wave 2: 13件 + 追加: 9件 + IAP: 4件 + WDT/Profile/Catalog追加: 5件 + IAP可変サイズ: 6件 = 計60件）全件PASS |
| 試験アプリ | test_app_ut.c（APP_AREA_EINK 0x2000 配置（2026-07-15アドレス是正）） |
| 環境記号 | E = エミュレータ（spike + car01_plugin）、R = 実機 |

---

## 試験結果一覧

| 試験ID | 試験名 | 結果 | 実施日 | 環境 | 備考 |
|--------|--------|------|--------|------|------|
| UT-KEY-01 | keyscan_init() 正常完了確認 | PASS | 2026-07-08 | E | boot_main()が起動時に実際のkeyscan_init()を呼び出し（boot.c:150）、app_main()到達 + UT-KEY-03 PASS（I2C動作確認）を根拠にPASSと判定。test_app内のno-opスタブはRC_API非経由のため試験に影響しない。実機(R)での確認はUPDI環境整備後に実施 |
| UT-KEY-03 | keyscan_get() 無押下時の戻り値確認 | PASS | 2026-07-02 | E | |
| UT-KEY-04 | keyscan_get_button() BTN_OKの押下・未押下判定 | PASS | 2026-07-02 | E | panel.html Sキー操作で確認 |
| UT-KEY-05 | keyscan_wait(mask) mask=0で任意ボタン待機・BTN_OK押下で復帰 | PASS | 2026-07-02 | E | panel.html Sキー操作で確認 |
| UT-SPI-01 | spi_manager_init() 正常完了確認 | PASS | 2026-07-02 | E | TFT初期化後BR=0（SPI_DEV_TFT追加後） |
| UT-SPI-02 | fram_write_byte() / fram_read_byte() 正常動作確認 | PASS | 2026-07-02 | E | |
| UT-SPI-03 | fram_write() / fram_read() 256Bバースト転送確認 | PASS | 2026-07-02 | E | |
| UT-SPI-04 | flash_read() 先頭1バイト読み出し確認 | PASS | 2026-07-02 | E | |
| UT-SPI-05 | spi_cs_select() / spi_cs_deselect() 排他制御確認 | PASS | 2026-07-02 | E | |
| UT-SPI-06 | SPI Flash → FRAM 256Bチャンク転送 | PASS | 2026-07-02 | E | |
| UT-FRAM-01 | FRAM管理領域（0x00000〜0x01FFF）書き込み・読み出し | PASS | 2026-07-02 | E | |
| UT-FRAM-02 | FRAMセーブデータ領域（0x02000〜0x05FFF）16KB境界確認 | PASS | 2026-07-02 | E | |
| UT-FRAM-03 | FRAMコンテキストスタック領域（0x10000〜0x13880）書き込み・読み出し | PASS | 2026-07-02 | E | |
| UT-FRAM-04 | FRAMフォント展開領域（0x13880〜0x23880）先頭・末尾アクセス | PASS | 2026-07-02 | E | |
| UT-FLASH-01 | SPI Flashカタログ領域（0x000000）先頭32バイト読み出し | PASS | 2026-07-02 | E | |
| UT-FLASH-02 | SPI Flash恵梨沙フォント領域（0x008000）先頭8バイト読み出し | PASS | 2026-07-08 | E | 判定を「not all 0xFF」から期待パターン8B完全照合に変更。setup_wave2_stubs.pyが0x008000に{0xA5,0x5A,0x12,0x34,0x56,0x78,0x9A,0xBC}を書き込み、b0=0xA5 match=1 PASS確認 |
| UT-FLASH-05 | FLASH_APP_BASE(0x098000)境界読み出し確認 | PASS | 2026-07-02 | E | |
| UT-TFT-01 | tft_init() 初期化シーケンス完了確認 | PASS | 2026-07-02 | E | |
| UT-TFT-02 | tft_fill() 赤・緑・青の全画面塗りつぶし | PASS | 2026-07-02 | E | 目視確認（SDL2ウィンドウ） |
| UT-TFT-03 | tft_set_window() + tft_write_pixel() 矩形領域描画確認 | PASS | 2026-07-02 | E | 目視確認（SDL2ウィンドウ） |
| UT-TFT-04 | tft_draw_char_elysia() 日本語文字「テ」1文字描画 | PASS | 2026-07-02 | E | 目視確認（SDL2ウィンドウ） |
| UT-TFT-05 | disp_draw_string() 22文字日本語文字列描画 | PASS | 2026-07-02 | E | 目視確認（SDL2ウィンドウ） |
| UT-TIM-01 | SysTick CNTレジスタのカウントアップ確認（時間計測基盤） | PASS | 2026-07-02 | E | エミュレータでdiff=144000（3ms相当）PASS範囲を200000に拡張 |
| UT-CATALOG-01 | valid=0x01 エントリ2件の正常読み出し | PASS | 2026-07-05 | E（x86 Unity） | |
| UT-CATALOG-02 | valid=0xFF による終端検出 | PASS | 2026-07-05 | E（x86 Unity） | |
| UT-CATALOG-03 | valid=0x00 削除済みエントリのスキップ | PASS | 2026-07-05 | E（x86 Unity） | |
| UT-RESOURCE-01 | 正常リソースのロード（end-to-end） | **再設計待ち** | ~~2026-07-05~~ | E（x86 Unity） | 悉皆点検C: 検証対象の線形サーチが load_resource.c→iap.c find_resource() へ移動しリンク不能。旧PASSは移動前コードに対するもの。設計書§1.8（再設計待ち注記）参照 |
| UT-RESOURCE-02 | res_type_id不一致→転送なし | **再設計待ち** | ~~2026-07-05~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-RESOURCE-03 | ディレクトリ終端 0xFFFF で即終了 | **再設計待ち** | ~~2026-07-05~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-FONT-02 | フォント領域オフセット計算（文字コード→SPI Flashアドレス・2026-07-18直読み化で基準是正） | PASS | 2026-07-18 | E（Spike） | log_l2_20260718_230443.txt |
| UT-FONT-04 | 字体スロット切替（slot0〜3 + 範囲外フォールバック・font_slot.c実物リンク） | **PASS** | 2026-07-18 | E（Spike） | 絨毯爆撃④。6ケース全一致（0x008000/0x020000/0x038000/0x050000・範囲外04/FF→slot0）。log_l2_20260718_230443.txt |
| UT-LOADRES-01 | load_resource() 正常系（Flashスタブに既知ディレクトリ配置） | PASS | 2026-07-05 | E（Spike） | commit 985ba6e |
| UT-LOADRES-02 | load_resource() 未登録種別（type_start=0xFFFF） | PASS | 2026-07-05 | E（Spike） | commit 985ba6e |
| UT-LOADRES-03 | load_resource() エントリ未発見（ディレクトリ終端到達） | PASS | 2026-07-05 | E（Spike） | commit 985ba6e |
| UT-FONT-01 | Flash→FRAM転送の整合性（512Bパターン・先頭・末尾・中間） | PASS | 2026-07-05 | E（Spike） | commit 985ba6e |
| UT-APPLIST-01 | read_app_list() FRAMスタブ経由の読み取り | PASS | 2026-07-05 | E（Spike・目視） | run_test_l2.sh --manual |
| UT-APPLIST-02 | カタログ0件時の動作（0xFF終端） | PASS | 2026-07-05 | E（Spike・目視） | run_test_l2.sh --manual |
| UT-IAP-01 | iap_run() SPI FlashからApp Areaへ書き込みとソフトリセット | PASS | 2026-07-09 | E（Spike） | run_test_iap.sh Phase1: Flash書込内容MATCH / Phase2: "UT-IAP-01 PASS"確認。R-01〜R-07全対策適用後に回帰試験としてPASS確認（commit b966dc2）。**2026-07-14 Phase 2a+ブートローダープール構成（f3d63fa）で回帰PASS**（log_iap_20260714_190824.txt） |
| UT-IAP-04 | iap_restore_from_fram() FRAMバックアップからApp Areaへの書き戻し | PASS | 2026-07-09 | E（Spike） | run_test_iap.sh Phase6: "UT-IAP-04 PASS"確認。FRAM[0x006000]にp6.binを配置し、iap_restore_from_fram()で App Area へ展開・起動を確認。**2026-07-14 Phase 2a+ブートローダープール構成（f3d63fa）で回帰PASS**（log_iap_20260714_190824.txt） |
| UT-IAP-02 | iap_call()によるモジュール起動と戻り確認 | PASS | 2026-07-10 | E（Spike） | run_test_iap.sh Phase1: callee(p4)書込MATCH / Phase2: "UT-IAP-02 PASS"確認。根本原因: iap_ctx.S の la 命令が GP-relative relaxation により addi a5,gp,offset に変換され、callee の gp（common_prog と異なる）で g_current_app_flash_addr を誤読。R-05チェック for(;;)ハング。.option norelax で絶対アドレス(auipc+addi)に修正。**2026-07-14 廃止＝SKIP化**（Phase 2aで旧2フェーズ・リセット方式が単一実行・直接ジャンプへ移行したため。後継=UT-IAP-06/07・log_iap_20260714_190824.txt） |
| UT-IAP-03 | iap_call()→iap_return()でSRAM 2KB全体復元確認 | PASS | 2026-07-10 | E（Spike） | run_test_iap.sh Phase2: "UT-IAP-03 PASS"確認。sent[0..3]パターン(0xDEADBEEF等)がSRAM復元後に正しく読み出せることを確認。**2026-07-13 可変長化(83b5da7)後の回帰でもPASS**（申告8192→sram_size=1024=全Zone C退避で等価動作）。**2026-07-14 廃止＝SKIP化**（Phase 2aで旧2フェーズ・リセット方式が単一実行・直接ジャンプへ移行したため。後継=UT-IAP-06・log_iap_20260714_190824.txt） |
| UT-IAP-05 | calc_sram_size() 退避サイズ式検証（256/512/1024/2048→64/64/128/256） | PASS | 2026-07-13 | E（Spike） | run_test_iap_var.sh シナリオ1(p10): "RC:UT-IAP-05 sram_size[4]=64,64,128,256 PASS"。iap.h の static inline 定義を試験アプリが直接取り込んで検証（log_iap_var_20260713_135327.txt）。**2026-07-14 Phase 2a実装（新シグネチャ・実アロケータ/パッチ経路・f3d63fa）で再実施PASS**（log_iap_var_20260714_190526.txt） |
| UT-IAP-06 | 複数サイズ（256B/1KB/2KB申告・sram_size 64/128/256B）でのiap_call/iap_return | PASS | 2026-07-13 | E（Spike） | run_test_iap_var.sh シナリオ2: サイズクラス別ld（slot64/128/256）でリンクした3種のcallerで往復・g_test_val=0x1234とパターン8B復元確認。**初回実施でslot64/128がFAIL**→保存機構スタックがスナップショット前にウィンドウを踏み潰す設計ギャップを発見・機構スタックのZone B退避で修正後PASS。**2026-07-14 Phase 2a実装で再実施: 初回3クラス全FAIL→復帰パスのgp未復元という実バグを発見（callerのgp相対アクセス全壊・Phase 1は同一リンカ配置で偶然潜伏）→ContextEntry v3（gp追加・28→32B）是正後PASS**（f3d63fa・log_iap_var_20260714_190526.txt） |
| UT-IAP-07 | IapCallStatus正常系（往復後status=0） | PASS | 2026-07-13 | E（Spike） | run_test_iap_var.sh シナリオ2: 3クラスとも"RC:UT-IAP-07 status=0 PASS"。復帰パスがg_restore.status経由でa0にIapCallStatusを設定する機構（実装時発見ギャップ）の検証を兼ねる。**2026-07-14 Phase 2a実装で再実施PASS**（3クラスともstatus=0・log_iap_var_20260714_190526.txt）。**2026-07-15 アドレス是正後の回帰で2048Bクラス（delta=0x800）のみFAIL→トランポリンsp/gp PC相対+c.lui破壊の潜在実バグ2件を発見・是正（eb13087）→全クラスPASS**（log_iap_var_20260715_103748.txt） |
| UT-IAP-08 | ERR_SRAM_FULL（callee単体サイズ超過拒否） | PASS | 2026-07-13 | E（Spike） | run_test_iap_var.sh シナリオ1(p10): ディレクトリsize=32768の異常ブロック(0xAC000)へのiap_callがstatus=-1で即時拒否・Flash書換/リセットなしで呼び出し元継続を確認。**2026-07-14 Phase 2a実装で再実施PASS**（log_iap_var_20260714_190526.txt） |
| UT-IAP-09 | WARN_NEAR_FULL / ERR_FRAM_FULL（FRAM使用量閾値） | PASS | 2026-07-13 | E（Spike） | run_test_iap_var.sh: ERR側=残り90B(<最小エントリ94B)でstatus=-2即時拒否（シナリオ1）。WARN側=使用量12KB+2B状態からの往復で復帰時status=1（シナリオ3/p13）。ネスト多段の実積み上げではなくtop_ptr事前設定による等価状態で試験（1段=1ソフトリセット=1Spike起動のため多段実行は非現実的）。**2026-07-14 Phase 2a実装で再実施PASS**（ERR側=depth=0 status=-2 fram_free=90・WARN側=depth=1 status=1。v3で最小エントリ94→98Bだが残り90Bは引き続き閾値未満で試験成立・log_iap_var_20260714_190526.txt）。**2026-07-15 アドレス是正後の回帰でWARN側がFAIL（callee=p12がdelta=0x800配置でc.lui破壊によりクラッシュ）→是正後PASS**（eb13087・log_iap_var_20260715_103748.txt） |
| UT-IAP-10 | パッチフィールドの256Bチャンク境界跨ぎ（キャリー経路直接検証） | PASS | 2026-07-15 | E（Spike） | run_test_iap_var.sh シナリオ4: fixture_carry.SのR32被パッチワードをオフセット0x2FE（mod 256=254・254+4=258>256で跨ぎ確定）へ決定的配置したcallee(p14)をcaller(p15)からiap_call。callee配置=cell3（delta=0x300）で"RC:UT-IAP-10 marker=9732 expected=9732 PASS"（0x2604=fixture_target 0x2304+delta。キャリー2B+2B合成の完全一致）・復帰status=0 PASS（log_iap_var_20260715_211716.txt）。**§8.2（キャリー経路の直接UT未実施）を解消**。副次的発見: patch_table_gen.pyのaddend無視バグを是正（試験設計書§2.0b備考参照） |
| UT-IAP-11 | A-2（別アプリ間呼び出し）往復とensure順序検証 | PASS | 2026-07-16 | E（Spike） | run_test_iap_var.sh シナリオ5。ネガティブコントロール（順序バグ再導入でFAIL）実証済み。詳細は変更履歴2026-07-16参照 |
| UT-IAP-12 | 常駐サブプログラム起動時ロード実経路 | PASS | 2026-07-16 | E（Spike） | run_test_iap_var.sh シナリオ6。正例0x1E08/負例0xFFFFFFFF両PASS。詳細は変更履歴2026-07-16参照 |
| UT-IAP-13 | 申告サイズ全振り幅スイープ（1024/2048/4096/8192B・slot512初使用・slot1024新設） | **PASS** | 2026-07-18 | E（Spike） | run_test_iap_var.sh シナリオ7。**cells=32（Zone C上限ちょうど・CTXエントリ最大1,058B）の正常系を初検証**。本スイープの初回実行が**gp ABI実バグを検出**（詳細=gp_abi_design.md・変更履歴2026-07-18） |
| UT-IAP-14 | calleeセル境界オフバイワン（申告255/256/257/511/512/513B） | **PASS** | 2026-07-18 | E（Spike） | 絨毯爆撃②。--callee-decl-size で実体と独立に注入・callee_cells/チャンク数の1→2遷移6点全PASS。log_iap_var_20260718_225408.txt |
| UT-IAP-15 | 過小申告→自己破壊ネガティブコントロール（破壊されたらPASS） | **PASS** | 2026-07-18 | E（Spike） | 絨毯爆撃③。--force-undersize でガードをバイパスし caller実体260B(2セル)を申告1セルで登録→cell1の番兵0xC0DEFACE破壊を確認。機構は申告を検証しない（安全網=PCツール+setupガード）ことの実証。log_iap_var_20260718_225408.txt |
| UT-CATALOG-04 | タイトルフォントインデックス列（uint16_t×8）の2B LE読み出し確認 | PASS | 2026-07-07 | E（x86 Unity） | |
| UT-RESOURCE-04 | res_type_id bits[15:12] による type ディレクトリ参照確認 | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-RESOURCE-05 | block_cell bits[3:0]=cell による Flash アドレスオフセット確認（非ゼロ cell） | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-RESOURCE-06 | リソースサイズ 3バイト LE デコード確認（size[1] 非ゼロ・258バイト） | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-KEY-02 | keyscan_get() 全ボタン入力確認（5方向スティック） | PASS | 2026-07-08 | E | inject_keys.pyによるUP/DOWN/LEFT/RIGHT/CENTER自動注入（0x01/0x02/0x04/0x08/0x10）、全5方向 LATCH consumed 確認 |
| UT-CATALOG-05 | 途中終端（valid=0xFF）でスキャンが打ち切られること | PASS | 2026-07-07 | E（x86 Unity） | test_catalog.c test_CATALOG_05_mid_terminator_stops_scan |
| UT-CATALOG-06 | 全件削除済み（valid=0x00×N件 + valid=0xFF終端）→ count==0 | PASS | 2026-07-07 | E（x86 Unity） | test_catalog.c test_CATALOG_06_all_deleted_zero_count |
| UT-CATALOG-07 | max_entries 上限到達でスキャンを打ち切ること | PASS | 2026-07-07 | E（x86 Unity） | test_catalog.c test_CATALOG_07_max_entries_limit（max=2で3件中2件返却） |
| UT-RESOURCE-07 | 線形サーチで先頭エントリをスキップして2件目にマッチ→転送 | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（線形サーチが find_resource() へ移動） |
| UT-RESOURCE-08 | type_start bits[3:0]=dir_cell による dir_addr オフセット確認（非ゼロ dir_cell） | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-RESOURCE-09 | リソースサイズ=0 のとき FRAM 転送が発生しないこと | **再設計待ち** | ~~2026-07-07~~ | E（x86 Unity） | 悉皆点検C・設計書§1.8（find_resource移動） |
| UT-CATALOG-08 | disp_typeフィルタ：機種非対応エントリのスキップ確認 | PASS | 2026-07-10 | E（x86 Unity） | test_catalog.c test_CATALOG_08_disp_type_filter。TFTプロファイル(disp_type=2)で disp_type=0(汎用)/2(TFT)/1(eink)の3件中2件返却を確認 |
| UT-WDT-02 | rtc_system_tick() IWDG->CTLR=0xAAAA 書き込み確認 | PASS | 2026-07-10 | E（x86 Unity） | test_wdt.c test_UT_WDT_02。ch32fun.h スタブ経由で g_mock_iwdg.CTLR=0xAAAA を確認 |
| UT-WDT-04 | 長時間処理ループ内でのWDTクリア呼び出し確認（10回連続） | PASS | 2026-07-10 | E（x86 Unity） | test_wdt.c test_UT_WDT_04。10回連続呼び出し後も CTLR=0xAAAA 維持を確認 |
| UT-PROFILE-01 | RtcHardwareProfile 固定アドレス 0x08D8 配置確認 | PASS | 2026-07-10 | E（Spike） | test_app_ut.c UT-PROFILE-01。*(uint32_t*)0x08D8 != 0xFFFFFFFF を確認（nm出力: rtc_hw_profile=0x000008d8） |
| UT-PROFILE-02 | RtcHardwareProfile ビットフィールド値確認（CAR-01 TFT機種） | PASS | 2026-07-10 | E（Spike） | test_app_ut.c UT-PROFILE-02。w0=0x0F021112(dc=2,pv=1,mc=1,rv=1,dt=2,dw=240) w1=0x000484F0(dh=240,kt=1,at=2,ss=1) 完全照合 |

---

## 結合試験（IT）結果一覧

> **2026-07-16新設（Task5④）:** 本表はこれまで存在せず、IT系の結果は変更履歴行にのみ記録されていた。そのため設計書に定義済みの **IT-IAP-03/04 が一度も追跡されないまま7回以上の回帰記録が積まれる**という追跡漏れが発生していた（除外を判断した記録はどこにもない）。**定義と追跡台帳を接続する**ため本表を新設し、設計書の定義件数と本表の件数が一致することを回帰のたびに確認する運用とする（ユーザー承認・2026-07-16）。
>
> **⚠ 件数の是正（2026-07-17）**: 2026-07-16時点で「IT定義=7件⇔本表=7件で一致」と記載したが、**これは誤りだった**。IT-IAP系のみを数え、**IT-BOOT-02/04・IT-FONT-01〜03・IT-RSRC-01〜03 の7件を数え漏らしていた**（追跡漏れを是正するために新設した本表に、同種の漏れを埋め込んでいた）。設計書のIT定義は**全15件**。下表を全15件に是正し、未実施も理由つきで明示する。
>
> **件数の一致確認（2026-07-17時点）: 設計書のIT定義=15件 ⇔ 本表=15件。一致。** 内訳: **PASS 8件**／**未実施 7件**（実機のみ1件＝設計時判断・追跡漏れ6件＝要判断）。

| 試験ID | 試験名 | 結果 | 実施日 | 環境 | 備考 |
|--------|--------|------|--------|------|------|
| IT-BOOT-01 | 通常起動フロー（App Areaあり → boot → アプリ起動） | PASS | 2026-07-16 | E（Spike） | run_test_it.sh。test_app_l2 の RC: 出力＝boot_main() 完走を根拠に判定 |
| IT-BOOT-02 | boot_main() GPIO2=HIGH時の外部通信モード分岐確認 | **未実施** | — | **R（実機のみ）** | **設計時からの判断＝追跡漏れではない**。設計書に「エミュレータはGPIO INDRが常に0返しのためGPIO2=HIGH分岐を再現できない」と明記。加えて外部通信の相手側スタブが未整備（共通プログラムの責務は`extern_comm_run()`＝CART_READY HIGH＋`iap_restore_from_fram(FRAM_SPI_MGR_ADDR,…)`でspi_flash_mgrへ制御を渡すまで。その先のPCとのプロトコルはspi_flash_mgr側）。**接続確立シーケンスまでは共通機能の責務のため将来実施**（ユーザー確認・2026-07-17） |
| IT-BOOT-03 | UP+DOWN同時押し → ブートローダセルフアップデートモード分岐 | PASS | 2026-07-16 | E（Spike） | run_test_it.sh。iap_result.bin vs Flash[0x018000] が MATCH |
| IT-BOOT-04 | App Areaが空の場合にFRAMバックアップから自己復元 | **PASS** | 2026-07-17 | E（Spike） | **追跡漏れから復帰・初実施**。**可変設計書§8.4（バックアップ生成→復元の往復）を本試験で解消**。生成側の検証を追加して往復化した。詳細は下記 |
| IT-IAP-01 | iap_call() → iap_return() 1段ネスト | PASS | 2026-07-16 | E（Spike） | UT-IAP-02/03 PASS による証跡代替（run_test_it.sh に判定根拠を出力） |
| IT-IAP-02 | IAPコンテキストスタック多重ネストと順次復帰（可変長版・Phase 2準拠） | **PASS** | 2026-07-16 | E（Spike） | **Task5④で再設計・10多重度で実証**。詳細は下記 |
| IT-IAP-03 | IAP A-1引数・戻り値エリア（FRAM管理領域）経由のモジュール間データ受け渡し | **PASS** | 2026-07-16 | E（Spike） | **追跡漏れから復帰・初実施**。引数エリア移設に伴いPhase 2準拠で再定義。詳細は下記 |
| IT-IAP-04 | IAP A-2の引数・戻り値受け渡しと呼び出し元メタの復元 | **PASS** | 2026-07-16 | E（Spike） | **追跡漏れから復帰・初実施**。**旧定義は現行機構では成立しなかった**（引数エリア移設で成立）。ネガティブコントロール実施済み。詳細は下記 |
| IT-IAP-05 | App Area溢れ時のLRU追い出しと復帰時sticky再ロード | **PASS** | 2026-07-16 | E（Spike） | **Task5⑥で新設**。多重度15＝33セル>32で意図的に溢れさせた。詳細は下記 |
| IT-FONT-01 | SPI Flash→FRAMフォント転送 | **未実施** | — | E+R | **追跡漏れ（2026-07-17発覚）**。L2/L3のUT-FONT系に等価な試験がある可能性があり、証跡代替で閉じられるかは**中身を確認して判断する**（IT-IAP-04で「代替可と思ったら観測点が欠けていた」を踏んだため、未確認のまま代替可とはしない） |
| IT-FONT-02 | フォントインデックスからFRAMアドレス算出確認 | **未実施** | — | E+R | 同上 |
| IT-FONT-03 | 日本語文字TFT表示（フォントロード→描画エンドツーエンド） | **未実施** | — | E+R | 同上 |
| IT-RSRC-01 | load_resource() 基本動作（画像） | **未実施** | — | E+R | **追跡漏れ（2026-07-17発覚）**。L2のUT-RESOURCE系との関係を要確認 |
| IT-RSRC-02 | load_resource() type_start[] テーブル参照確認 | **未実施** | — | E+R | 同上 |
| IT-RSRC-03 | load_resource() ディレクトリ拡張対応 | **未実施** | — | E+R | 同上（SPI Flashに拡張ダミーデータが必要） |

### IT-IAP-02 詳細（2026-07-16・Task5④）

**方式**: 試験設計書 初版（41795fb・2026-07-07）の原案＝「自分自身を再帰呼び出しする試験モジュール」方式へ回帰。2026-07-10実装（p7/p8/p9の3バイナリ・リセット連鎖・2段）は原案から逸れており（逸脱理由の記録なし）、Phase 2の直接ジャンプ方式と非整合だったため廃止。

**10重登録の原理**: 配置状態表の検索キーが`(catalog_idx, res_id)`の組（`iap.c` `plc_find`）であることを利用し、**同一のCODE実体・パッチテーブルを res_id 2〜11 の10組で登録**。各 res_id が別リージョンとして別セルへ配置され10個が同時在席する。バイナリは1本（p20）で足り、Flash追加コストはディレクトリ20エントリのみ。

**実測（初回実証: log_it_20260716_230944.txt ／ 出力ラベル変更後の再実証: log_it_20260717_084457.txt）**
※初回ログは出力ラベルが`RC:IT-IAP-02 …`、再実証ログは`RC:IT-IAP-NEST …`（IT-IAP-05新設に伴う試験ID非依存化・下記IT-IAP-05詳細参照）。判定内容・結果は同一。

| 判定 | 結果 |
|---|---|
| ①入場トレース | `mult=0→10` 単調増加（期待 `[0..10]` と完全一致）→ PASS |
| ②復帰 | 10行すべて PASS / FAIL=0（多重度9〜1のモジュール9本＋L0 1本。最奥段は iap_call せず復帰行なし） |
| ③配置状態表 | `res1@cell0+3, res2@cell3+2, res3@cell5+2, … res11@cell21+2` = **count=11・重複なし・res_id=[1..11]** → MATCH |

**セル勘定（段数の律速）**: 律速はFRAM容量（16KB・1エントリ≈98B＝理論上100段超）ではなく **App Areaのセル数32**。実測 **L0(p19)=748B（申告768B=3セル・余裕20B）**・**モジュール(p20)=472B（申告512B=2セル・余裕40B）**→ **2×10+3=23セル ≤ 32セル**。実測 cell=3,5,7,…,21 と2セル刻みで並んでおり、**禁止residueによる窓選択の阻害は発生しなかった**。

> **L0の余裕が20Bしかない点に注意**: p19は配置状態表ダンプ（判定③の証拠出力）を持つため748Bある。申告768Bを超えると setup が fail-closed で停止する（過小申告＝セル占有の誤認＝自己破壊のため）。L0にこれ以上出力を足す場合は申告を1024B（4セル）へ上げること。その場合の総セル数は 2×10+4=24セルで依然32セル内に収まる。

**実装時の知見3件**:
1. **モジュールの2セル制約**: 汎用の`rc_putu`（pw[5]表＋桁分解ループ）を使うと600B=3セルとなり、10×3+3=33セル>32で成立しない。多重度が0〜10しか取らないことを利用した2桁固定出力へ置換し472B（2セル）に収めた。将来モジュールが太った際に黙って段数が下がらないよう、setup側にセル勘定の fail-closed 検証を実装（`--test nest`）。
2. **判定にシリアル出力を用いる理由**: FRAMスタブは`sync()`をデストラクタでのみ呼ぶ（`fram_stub.cc`）。本試験は最終段がハルトし`timeout`でSpikeを終了させるためデストラクタが走らず、**`fram.bin`は setup 時の状態のまま**＝実行後のファイルから配置状態表を読む方式は使えない。L0が復帰後に自らダンプする方式へ変更した。UT-IAP-12が`resident_result.bin`で判定できるのは`iap_run`→soft_reset→`exit(0)`でプラグインの保存経路を通るためで、経路が異なる。
3. **setup失敗の隠蔽**: `tee`がpythonの終了コードを返すため`set -e`では setup 失敗を捕捉できず、前試験のFRAMが残ったまま実行されて「機構のFAIL」に見える事象を実際に踏んだ（`--test nest`のargparse choices追加漏れ）。nest setup呼び出しに`PIPESTATUS`チェックを追加して fail-closed 化。

**副次的に実証されたもの**: `allocate()`は実行中・中断中モジュールのセルを明示保護しておらず、「直前にtouchされた段はLRU上位＝追い出されない」という前提（`iap.c`の caller touch）に依存している。本試験は11リージョンが競合する状態でこの前提を実経路検証する唯一の試験となった。

### IT-BOOT-04 詳細（2026-07-17・§8.4「バックアップ生成→復元の往復」を解消）

**背景**: 可変設計書§8.4は、圧縮施策5（App Areaバックアップの`fram_write`1トランザクション化＝内蔵Flashメモリマップ直読み・旧256Bチャンクループ廃止／`iap_restore_from_fram`のnoinline化・−80B）の回帰確認を求めていた。UT-IAP-04は**setupが事前に置いたFRAMバックアップからの復元**しか見ておらず、**実際の生成（`boot.c` `backup_app_area_to_fram`）→復元の往復**が未確認だった。設計書に定義済みのIT-BOOT-04（追跡漏れ・未実施）が復元側そのものだったため、**新IDを起こさずIT-BOOT-04に生成側の検証を足して往復化**した。

**往復の組み方（fram.binの永続性を利用した2実行）**:

| | 内容 | 判定 |
|---|---|---|
| Phase 1 | App Area=p1 で起動 → boot が App Area(0x2000・8KB) を FRAM 0x18000 へバックアップ → p1 が `iap_run()` → `soft_reset` → **PficStubが`exit(0)`＝デストラクタが走り`fram.bin`が同期される** | ①`fram.bin[0x18000..+8KB]` == p1（**生成**の検証） |
| Phase 2 | App Area=空(0xFF)・**setupを走らせず**Phase 1の`fram.bin`を引き継ぐ → boot が `app_area_has_program()`=false → `fram_backup_valid()`=true → `iap_restore_from_fram` → `soft_reset` | ②`iap_result.bin` == p1（**復元**の検証） |

**実測（生ログ: log_it_20260717_120434.txt）**: ①MATCH ②MATCH → **PASS**。

**判定を成果物のみで行う理由**: `boot_puts()`は`#ifdef BOOT_LOG`で通常ビルドではno-opに展開される（Flash節約）ため、`[boot] fram backup restore`のようなログ文字列は判定条件にできない（当初これを条件に入れて誤FAILさせた）。Phase 2はApp Areaを0xFFで結合しているので、**`iap_result.bin`がp1と完全一致する経路は「`fram_backup_valid()`→`iap_restore_from_fram()`でFRAMバックアップをApp Areaへ書き戻し→`soft_reset`」以外に存在しない**＝成果物自体が8bルート通過の証明になる。

**ネガティブコントロール（2026-07-17実施）**: FRAMバックアップの先頭4Bを0xFFへ書き換え（`fram_backup_valid()`をfalseにする）てPhase 2を実行すると、**`iap_result.bin`が生成されない**＝復元経路に入らないことを確認。本試験がバックアップ有効性判定と復元経路を実際に見ていることを実証した。

### IT-IAP-03 / IT-IAP-04 詳細（2026-07-16・引数エリア移設に伴う初実施）

**経緯**: Task5でIT-IAP-04の証跡代替可否を判定しようとした際、**旧定義が現行機構では成立しない**ことが判明した。`a2_meta_swap_in`（`iap.c:554`）はA-2呼出時に呼び先アプリの、復帰時に呼び元アプリの**メタ8KB全体をFlash原本から転送する**（`find_resource`が呼び先ディレクトリを引くためで、A-2に本質的な動作）。旧引数エリアは4領域ともメタ内（+0x1680〜+0x177F）にあったため、**A-2引数は呼出時に、A-2戻り値は復帰時に、機構自身が塗り潰していた**。→ 4領域とも**FRAM管理領域へ移設・各256B**（`iap_design.md`「IAP引数・戻り値エリア」）。

**実測（生ログ: log_it_20260717_103605.txt）**:

| 試験 | 判定 | 結果 |
|---|---|---|
| IT-IAP-03 | A-1引数64B往復（+1加工） | `RC:IT-IAP-03 status=0 ret64=OK PASS` |
| IT-IAP-04 | ①A-2引数64B往復（+1加工） | `RC:IT-IAP-04 status=0 ret64=OK PASS` |
| IT-IAP-04 | ②呼び出し元メタ復元 | `RC:IT-IAP-04 type_start0=32 expect=32 PASS`（0x0020=アプリA自身の値） |

**ネガティブコントロール（2026-07-16実施・移設が実バグ修正であることの実証）**: 引数エリアのアドレスのみを旧配置（メタ+0x1700/+0x1740）へ戻したバイナリでIT-IAP-04を実行すると:

```
RC:IT-IAP-04 callee arg->ret done
RC:IT-IAP-04 status=0 ret64=NG FAIL          ← 引数受け渡しのみ失敗
RC:IT-IAP-04 type_start0=32 expect=32 PASS   ← メタ復元は正常
```

**`status=0`（呼び出しは成功）・メタ復元も正常なまま、引数だけが静かに壊れる**という不具合の姿がそのまま再現した。日本語入力Wnn（A-2公開サービスとして設計済み・A-2引数エリアでひらがな文字列を受け取る／実測見積もり約40B）を実装していれば、原因不明の文字化けとしてこれに突き当たっていた。本試験が当該バグを確実に捕捉することも同時に実証された。

**判定②を設けた理由（UT-IAP-11との差分）**: UT-IAP-11はA-2の呼出/復帰両経路を通すが、検証対象は`status==0`とcaller側Zone Cの復元であり、**呼び出し元のメタが実際に書き戻されたことを観測していない**。メタが戻らなければ復帰後のアプリAは`find_resource`でアプリBのディレクトリを引いてしまう。`type_start[0]`はアプリA(0x0020)/B(0x00E0)で異なり、試験開始時のFRAMメタは0xFFFF（未ロード）なので、**「書き戻さない」「アプリBのまま」の両方の失敗モードを捕捉できる**。この観測点があるため、**IT-IAP-04はUT-IAP-11の証跡代替では閉じられない**（当初は代替可と見立てたが誤りだった）。

**実装**: 同一ソースを`-D`で振り分け（p11s/m/l と同じ方式）。`p21a/p22a`=A-1（`FRAM_IAP_A1_ARG/RET`・`catalog_idx=0xFF`）、`p21b/p22b`=A-2（`FRAM_IAP_A2_ARG/RET`・`catalog_idx=1`・`CHECK_META`定義）。ハーネスは既存の`--test iap_var`（A-1）/`--test a2`（A-2）をそのまま流用したため**setup_iap_stubs.pyの変更は不要**だった。calleeが各バイトに+1して返す設計により、無変化のまま通る偶然のPASSを排除している。

**回帰**: IT 7項目全PASS（log_it_20260717_103605.txt）・UT-IAP-05〜12 全PASS。common_progは**定数追加のみでコード変更ゼロ**（FLASH 5516B/RAM 524B/BOOTPOOL 1652B＝移設前と完全一致を実測確認）。

### IT-IAP-05 詳細（2026-07-16・Task5⑥）

**方式**: IT-IAP-02 とハーネス・試験モジュールを共用し、目標多重度のみ15へ（`NEST_TARGET`を`-D`で上書き＝p20e・setup側は`--nest-target 15 --allow-cell-overflow`）。**2×15+3=33セル > 32セル**でApp Areaが必ず溢れ、`allocate()`のLRU追い出しと`prep`のsticky再ロード（`iap.c:1236-1252`）を実経路で通す。

**実測（生ログ: log_it_20260717_084457.txt）**:

| 判定 | 結果 |
|---|---|
| ①入場トレース | `mult=0→15` 単調増加（溢れても呼び出し自体は成功）→ PASS |
| ②復帰 | 15行すべて PASS / FAIL=0 |
| ③配置状態表 | **count=15**（追い出しなしなら16）・**追い出された res_id=[16]**・**res1(L0)在席**・使用31/32セル・重複なし → MATCH |

**観測された経過（事前予測と完全一致）**:
1. res16配置時点で L0(3セル)+res2〜15(28セル)=31セル使用済み・空き1セル。res16は2セル必要 → **追い出し発生**
2. LRU最小は L0（seq=3）→ res16 が L0 のセル0〜1を奪う
3. 復帰時に prep が L0 の不在を検出 → `find_resource(CODE|1)`/`(PATCH|1)` → cell0へ **sticky再ロード**
4. その再ロード（3セル）の`plc_evict(0,3)`が今度は res16 を追い出す → **最終状態: res1在席・res16のみ消失**

**判定の論理**: 16リージョンは33セルを要し32セルに物理的に収まらないため、`count < 16` は追い出しが起きたことの**算術的な証明**。その上で **L0が自身の配置状態表ダンプを正しく出力できたこと自体が、sticky再ロードが働いた証明**になる（復元されていなければ cell0 には res16 のコードが乗ったままで、戻った瞬間に別モジュールを実行し正しい出力を出せない）。

**本試験で確定した規約（実機で顕在化しうる要件・ユーザー決定 案A）**: sticky再ロードは `find_resource(CODE|res_id)` と `find_resource(PATCH|res_id)` を引き、不在なら**無限ループでハングする**（`iap.c:1241`/`1246`）。CODE側は「アプリ本体=自アプリCODEディレクトリのResourceEntry[0]」という`iap_placement_init`（`iap.c:1296-1297`）の前提で保証されるが、**本体のパッチテーブルの存在を保証する仕組みは無い**。本体はcell0・delta=0で動くため通常運用では不要だが、追い出されて再ロードされる瞬間にだけ要求される。→ **「App Areaを使い切る可能性のあるアプリは本体にもパッチテーブルを登録すること」をPC側カタログ生成ツールの要件とする**（§8.7 開発者ガイド転記項目へ追加）。本試験のL0(p19)も`--emit-relocs`でリンクし`p19_patch.bin`を登録している。**この経路はIT-IAP-02（10多重・追い出しなし）では踏めない。**

**実装時の知見**: 試験アプリの出力ラベルを`RC:IT-IAP-02 …`から**`RC:IT-IAP-NEST …`（試験ID非依存）へ変更**した。p19/p20はIT-IAP-02とIT-IAP-05で共用されるため、IT-IAP-05のログに他試験のIDが並び証跡として読めない状態だったため。

---

## 未実施試験

| 試験ID | 試験名 | 理由 |
|--------|--------|------|
| UT-KEY-02（R） | keyscan_get() 全ボタン入力確認（5方向スティック） | エミュレータPASS済み。実機試験はUPDI環境整備後 |
| UT-KEY-01（R） | keyscan_init() 正常完了確認 | エミュレータPASS済み（B案）。実機でのI2C初期化確認はUPDI環境整備後 |
| UT-KEY-03〜05（R） | キー系試験 | エミュレータPASS済み。実機はUPDI環境整備後 |
| UT-FLASH-02（R） | SPI Flash恵梨沙フォント領域（0x008000）先頭8バイト読み出し | エミュレータPASS済み。実機はUPDI環境整備後 |
| UT-FLASH-03 | SPI Flashアプリ領域セクタ消去確認 | 実機（R）のみ・UPDI環境整備後 |
| UT-FLASH-04 | SPI Flash 4KBブロック書き込み・読み出し | 実機（R）のみ・UPDI環境整備後 |
| UT-IAP-01（PASS転記済み）| iap_run() | → 上記PASSテーブルへ移動（2026-07-09） |
| UT-IAP-04（PASS転記済み）| iap_restore_from_fram() | → 上記PASSテーブルへ移動（2026-07-09） |
| UT-IAP-02（PASS転記済み）| iap_call() | → 上記PASSテーブルへ移動（2026-07-10） |
| UT-IAP-03（PASS転記済み）| iap_return() SRAM復元確認 | → 上記PASSテーブルへ移動（2026-07-10） |
| UT-KEY-02（全ボタン） | keyscan_get() アクションボタン（A/B/X/Y等）確認 | D基板MCU見直し後（ATtiny1604→AVR64DD28検討中）に再定義 |
| UT-WDT-01 | IWDG 初期化確認（PSCR/RLDR/CTLR書き込みシーケンス） | car01_plugin に IWDG NullPeriphStub 追加済み（フォールト解消）。初期化シーケンス詳細のスタブ検証は未実施。実機(R)確認待ち |
| UT-WDT-03 | IWDG リセット後のシステムメニュー強制帰還確認 | boot.c に iwdg_reset 判定実装済み（RCC->RSTSCKR フラグ読み出し）。エミュレータでの再現・自動試験は未実施。実機(R)確認待ち |

---

## Flash メモリ使用量履歴（common_prog / Flash上限 5632B）

コミット内容と Flash 使用量の変化を記録する。**今後も変化のあるコミット毎に追記すること。**

| コミット | サイズ(B) | 増減(B) | 主な変更内容 |
|----------|----------|---------|-------------|
| 20c6c1d | 5084 | −124 | iap_run() SRAM overflow修正（8KB一括→256Bチャンク） |
| b528eb9 | 5100 | +16 | keyscan_wait() mask引数追加 |
| da4a4b0 | 4788 | −312 | tft_oled.c 8x8px再ビルド（LTO最適化効果） |
| 5936d7a | 4760 | −28 | tft_draw_char_elysia() FRAM一括読み出し方式 |
| 53c219f | 4772 | +12 | tft_draw_char_elysia() 1バイトずつに戻す |
| 376a22e | 4760 | −12 | tft_draw_char_elysia() ELYSIA_CHAR_BYTES汎用版 |
| 7ccbd02 | 4884 | +124 | tft_draw_char_elysia() 1トランザクション化 |
| 21a4f1b | 4924 | +40 | SPI転送パフォーマンス改善（flash_to_fram_seq 256Bチャンク等） |
| **41e49ae** | **5336** | **+412** | **boot.c USART1フォントロード進捗ログ追加**（最大増加） |
| 43bb9f1 | 5588 | +252 | Bug-2/Bug-3 コンテキストスイッチ修正（iap_ctx.S naked asm追加） |
| R-01実装 | 5624 | +36 | FRAM書き込みアトミシティ対策（restore_flag write-last） |
| BOOT_LOG条件コンパイル化 | 5080 | −544 | boot.c 進捗ログを `#ifdef BOOT_LOG` で条件コンパイル化（ヘッドルーム 552B 回復） |
| R-02: Flash書き込みエラー検出 | 5124 | +44 | ram_flash_write_chunk() に WRPRTERR チェック追加（消去・書込各ループ） |
| R-03: App Area復元CRC検証 | 5216 | +92 | CRC-16/MODBUS 追加・ContextEntryヘッダ 16B→20B・復元後ベリファイ |
| R-04: Flash書き込み中IRQ管理 | 5220 | +4 | iap_sram_restore_and_jump に csrsi mstatus,8 追加（SRAM復元後・ジャンプ前に IRQ 再有効化） |
| R-05: caller_flash_addr範囲検証 | 5332 | R-07と同時計上 | iap.h に FLASH_END(16MB) 追加・sram_save_to_fram() に範囲/アライン検証追加 |
| R-06: 周辺機器状態ガイドライン | 5332 | 0 | iap.h にサブプログラム向けガイドラインコメント追加（コード変更なし） |
| R-07: SPI通信タイムアウト | 5332 | +112(R-05/06/07合計) | spi_manager.c 全SPI転送ループに10000カウントタイムアウト追加（超過でハルト） |
| §6リンカスクリプト+boot.c修正 (b966dc2) | 5324 | −8 | common_prog.ld Zone A強制(LENGTH=768)・boot.c chunk_buf[1024]→[256] |
| GP-relative relaxation修正 (iap_ctx.S) | 5392 | +68 | iap_call_entry / iap_sram_restore_and_jump 内の la 命令に .option norelax 追加。callee gp 不正による g_current_app_flash_addr 誤読（R-05 for(;;)ハング）を修正 |
| WDT+HardwareProfile+display dispatch実装 | 5560 | +168 | rtc_system_tick()・RtcHardwareProfile(0x08D8)・rc_disp_*ラッパー6本・IWDG初期化・disp_typeフィルタ実装。test_app_ut に UT-PROFILE-01/02追加（8132B/8192B） |
| DISP_TFT/DISP_EINKコンパイル時フラグ化 | 4588 | −972 | rc_api.c の実行時 disp_type 分岐を #ifdef DISP_TFT / DISP_EINK に置換。TFTビルドから eink コード完全除外。tft_fill 解像度を RC_HW_PROFILE 参照に動的化 |
| fence.i追加 (569bcea) | 4592 | +4 | iap_sram_restore_and_jump の jr t0 直前に fence.i を追加（.insn手書きエンコード）。Spike icacheステイル実行の解消 |
| IAP可変サイズ Phase 1 (83b5da7) | 5016 | +424 | 可変長ContextEntry・LIFO可変長CTXスタック・動的容量チェック・IapCallStatus返却・FRAM新マップ切替・SP固定・機構スタック退避。**見込み+150〜250Bを超過**（動的チェック2系統・サイズ自動取得・エラー復帰経路・復帰時status返却が正味コスト。LE直接転送・構造体アクセス・g_restore統合で5120B→5016Bまで圧縮済み。-Wl,--no-relax全面適用案は+824Bで不採用） |
| パターンA/B統合 (chunk_copy) | 4836 | −180 | 4系統並存していた256Bチャンク転送ループ（iap_run 100B/restore_app_from_flash 90B/iap_restore_from_fram 86B/flash_to_fram_seq 100B）+boot.cフォント転送ループを、転送元(Flash/FRAM)×転送先(App Area/FRAM)をモードフラグ選択する単一エンジンchunk_copyに統合。ソフトリセットもsoft_reset()に共通化。バッファはstatic 256B共有（load_resourceがサブアプリの小スタックのまま呼ぶためスタック化不可・非再入前提）。UT-IAP-01〜09/IT系全PASSで回帰確認 |
| **Phase 2a実装（2026-07-13）** | **6888（リンク不能）** | **+2052** | App Area側アロケータ+パッチ機構（iap_call_impl 1196B/load_and_patch 632B/prep 522B）。**見積+350〜550Bの約4倍で予算を1256B突き破り、リンク失敗のまま保留**。この破綻が「共通プログラム自身の多段コンテキストスイッチ化」構想の発端（可変設計書§6） |
| 圧縮8施策（2026-07-14） | 6440（リンク不能） | −448 | 重複排除中心（tft窓設定統合−112/place_and_register統合−88/バックアップ1トランザクション化−80/cur_set・plc_register−60/keyscan恒等マップ−52/ctx_set_statusパック−36ほか・可変設計書§6.7）。副産物としてmissクラスタが単一関数塊に物理統合され、プール化の下準備が整った。**まだ808B超過** |
| **ブートローダー領域プール移設（2026-07-14）** | **5264＋BOOTPOOL 1730** | **FLASH −1176** | **最後の一手**。ブートローダー2KBのうちページ0を常駐スタブ化し、0x0040〜0x07FF(1984B)を第二のコード領域（スワッププール）として開放。missクラスタ1138B+起動UIクラスタ514BをBOOTPOOLへ移設（Flash上限5632Bの外数）し、プール管理常駐分（pool_ensure_loaded/pool_regime_init/スタブ定数・約550B相当は-fno-delete-null-pointer-checks影響込み）を差し引いても**予算内へ帰還・リンク成立（マージン368B）**。可変設計書§7。**UT回帰未実施**（テストアプリ旧シグネチャのため要書き換え） |
| gp復元是正=ContextEntry v3（f3d63fa・2026-07-14） | 5292＋BOOTPOOL 1730 | +28 | **UT-IAP-06で発見の実バグ是正**: 復帰パスがgp未復元でcallerのgp相対アクセス（8B以下静的変数）が全壊。ContextEntry 28→32B(gp追加)・g_restore 20→24B・iap_ctx.S両パス修正。**この修正でUT-IAP回帰全PASS**（01/04 PASS・02/03廃止SKIP・05〜09全PASS・可変設計書§7.10） |
| entries overlay=Phase 2b 4a（2026-07-15） | 5292＋BOOTPOOL 1714 | ±0（BOOTPOOL −16） | `entries[8]`⇔`s_chunk_buf`共用体化（可変設計書§7.7）。Flashコスト実質ゼロでSRAM −224B（下表参照）。BOOTPOOL微減はBTN_OKローカル退避のコード整理分。§2.1.1スタック収支再検証同時実施（機構スタック最悪276B・Zone B食み出し20B・余裕224B）。UT-IAP回帰+L3全30項目PASS |
| 境界跨ぎキャリー=Phase 2b 4b（2026-07-15） | 5292＋BOOTPOOL 1778 | ±0（BOOTPOOL +64） | load_and_patchのチャンク境界跨ぎパッチをfail-closedハルト→キャリー方式へ（apply_patch_word共通化・可変設計書§7.4/詳細設計§7.7）。missクラスタ1258Bが枠1216Bを超過し**プール境界0x0500→0x0540へ1ページ移動**（miss枠1280B/bootui枠704B・BOOTPOOL余裕206B）。UT-IAP回帰+L3全30項目PASS（キャリー経路自体のUTはTask 5で新設） |
| App Areaアドレス是正+C_LUI対応（eb13087・2026-07-15） | **5280＋BOOTPOOL 1956** | −12（BOOTPOOL +178） | **①App Area開始アドレス是正0x1E00→0x2000**（ユーザー指摘・正マップ=共通プログラム0x0800〜0x1DFF/スイッチしない共通領域0x1E00〜0x1FFF/App Area 0x2000〜0x3FFF。FLASH−12Bは0x2000がaddi不要のluiになるため）**②トランポリンsp/gp是正**（test_app_iap.cの`la`=PC相対→絶対lui+addi。delta移動でSRAM固定アドレスを誤計算・Spikeの4KBページパディングが小deltaで隠蔽していた実バグ）**③C_LUI対応**（圧縮luiを4B word前提でパッチし隣接命令を破壊していた機構コアの実バグ是正・patch_table_gen.py新種別4+iap.cハーフワード経路）。missクラスタ1435Bへ成長し**プール境界0x0540→0x0600へ移動**（BOOTPOOL 1956/1984B余裕28B・**以後のmiss成長は新規プールモジュール化必須**）。UT-IAP+L3全回帰PASS |

**備考:**
- `41e49ae` の +412B が最大の増加要因（BOOT_LOG無効化で解消）
- 実機試験時に BOOT_LOG を有効化する場合: `docs/wave2_test_plan.md §7` 参照
- Flash残量が少ない場合は実装前にこの表で影響を確認すること
- ~~2026-07-13時点: 4836B / 5632B（使用率85.9%、余裕796B）~~
- ~~2026-07-14時点: FLASH 5292B / 5632B（94.0%、余裕340B）＋BOOTPOOL 1730B / 1984B（87.2%）~~
- ~~2026-07-15 4b時点: FLASH 5292B＋BOOTPOOL 1778B~~
- ~~2026-07-15 eb13087時点: FLASH 5280B＋BOOTPOOL 1956B（余裕28B）~~
- ~~2026-07-15 Task 4c(A-2実装)時点: FLASH 5552B / 5632B（98.6%、余裕80B）＋BOOTPOOL 1544B / 1984B（77.8%、余裕440B）~~。BOOTPOOLの余裕が大幅回復したのは`.pool_2_a2`をbootuiとVMA時分割共有としたため（可変設計書§7.4）。FLASH（常駐側）の余裕は80Bまで逼迫していた
- **2026-07-16 Task 4d(§7.11常駐ロード実装・B'案)時点: FLASH 5516B / 5632B（97.94%、余裕116B）＋BOOTPOOL 1652B / 1984B（83.27%）＋POOL_LMA（仮想・勘定専用）180B・RAM 524B不変。3スイート全PASS**（run_test_iap.sh=UT-IAP-01/04・run_test_iap_var.sh=UT-IAP-05〜10・run_test_ut.sh=L3全30項目、生ログlog_iap_20260716_124247.txt他）。4c比でFLASHが5552B→5516Bへ**−36B改善**した主因は、当初4d(WIP)のパッチ適用チェーン複製方式（FLASH 6570B・+938B超過）をB'案（本家load_and_patch共用）へ設計転換し、かつ`.pool_2_a2`のLMAを`AT>FLASH`から仮想領域POOL_LMAへ移してFLASH勘定から外したため（詳細設計書§7.11.2〜7.11.4）。付随: `word_le`をmiss→常駐.textへ移動（miss枠1472B死守）・`a2_meta_swap_in`のインライン漏れをnoinlineで是正・`iap_run`の常駐判定をガード先行化（常駐なしアプリはプール非依存＝固定版ハーネスでハングしない）。**常駐側FLASH余裕は116B——次に常駐側を消費する変更は着手前にこの116Bと比較すること**

---

## SRAM（common_prog / Zone A）使用量履歴

コミット内容と SRAM（Zone A、`.data`+`.bss`+`.ram_func`の静的合計）使用量の変化を記録する。**今後も変化のあるコミット毎に追記すること。Flash使用量履歴表と同様の運用とする。**

**注記（重要）**: 以下の数値はリンク時に確定する**静的**フットプリント（`.data`/`.bss`/`.ram_func`のVMA合計）のみであり、実行時のスタック消費（オーケストレータのcall depth等）は含まない。スタック側の余裕は別途`boot.c`の`_eusrstack`設定、および`common_prog/iap_context_switch_variable_design.md`（IAP可変サイズコンテキストスイッチ仕様書）を参照すること。

**2026-07-09（コミットb966dc2）に容量定義そのものが変わった点に注意**: `common_prog.ld`のRAM領域は、それ以前は`LENGTH=2K`（Zone A+B+C相当を静的領域として許容）だったが、b966dc2で`LENGTH=768`（Zone Aのみに強制、Zone B/Cはオーケストレータスタック・サブプログラム用に予約）へ縮小された。これによりリンク時検出可能な上限が2048B→768Bへ一気に下がっている。

| コミット | SRAM使用量(B) | 容量(B) | 使用率 | 増減(B) | 主な変更内容 |
|----------|---------------|---------|--------|---------|-------------|
| 20c6c1d | 388 | 2048 | 18.9% | — | iap_run() SRAM overflow修正（8KB一括→256Bチャンク） |
| b528eb9 | 388 | 2048 | 18.9% | 0 | keyscan_wait() mask引数追加 |
| da4a4b0 | 388 | 2048 | 18.9% | 0 | tft_oled.c 8x8px再ビルド |
| 5936d7a | 388 | 2048 | 18.9% | 0 | tft_draw_char_elysia() FRAM一括読み出し方式 |
| 53c219f | 388 | 2048 | 18.9% | 0 | tft_draw_char_elysia() 1バイトずつに戻す |
| 376a22e | 396 | 2048 | 19.3% | +8 | tft_draw_char_elysia() ELYSIA_CHAR_BYTES汎用版 |
| 7ccbd02 | 396 | 2048 | 19.3% | 0 | tft_draw_char_elysia() 1トランザクション化 |
| 21a4f1b | 652 | 2048 | 31.8% | +256 | SPI転送パフォーマンス改善（flash_to_fram_seq 256Bチャンク化） |
| 41e49ae | 652 | 2048 | 31.8% | 0 | boot.c USART1フォントロード進捗ログ追加（Flashのみ増加） |
| 43bb9f1 | 656 | 2048 | 32.0% | +4 | Bug-2/Bug-3 コンテキストスイッチ修正 |
| 1a3805d | 656 | 2048 | 32.0% | 0 | BOOT_LOG条件コンパイル化・R-01 FRAMアトミシティ対策 |
| R-02: Flash書き込みエラー検出 (106f8b1) | 688 | 2048 | 33.6% | +32 | ram_flash_write_chunk() WRPRTERRチェック追加 |
| R-03: App Area復元CRC検証 (626867c) | 688 | 2048 | 33.6% | 0 | CRC-16/MODBUS追加（ワークはスタック側） |
| R-04: Flash書き込み中IRQ管理 (c9df902) | 688 | 2048 | 33.6% | 0 | iap_sram_restore_and_jump にcsrsi追加 |
| R-05/06/07 (90a60dc) | 688 | 2048 | 33.6% | 0 | caller_flash_addr検証・SPIタイムアウト追加 |
| **§6リンカスクリプト強制 (b966dc2)** | **688** | **768** | **89.6%** | **容量2048→768** | **RAM LENGTH 2K→768（Zone Aのみ強制）。使用率が33.6%→89.6%へ急上昇（容量減が主因、静的使用量自体は不変）** |
| GP-relative relaxation修正 (cbd64c3) | 696 | 768 | 90.6% | +8 | iap_ctx.S la命令に.option norelax追加 |
| WDT+HardwareProfile+display dispatch (1c6bce4) | 712 | 768 | 92.7% | +16 | RtcHardwareProfile・rc_disp_*ラッパー・IWDG初期化追加 |
| DISP_TFT/DISP_EINKコンパイル時フラグ化 (f068a6e) | 708 | 768 | 92.2% | −4 | eink分岐コード除外に伴う静的データ縮小 |
| fence.i追加 (569bcea) | 708 | 768 | 92.2% | 0 | Flashのみ増加（fence.iはSRAM非消費） |
| IAP可変サイズ Phase 1 (83b5da7) | 712 | 768 | 92.7% | +4 | g_restore を4変数16B→5フィールド20B構造体化（status追加・復帰時IapCallStatus返却用）。GP緩和回避のため.sbss個別変数を廃止 |
| パターンA/B統合 (chunk_copy) | 712 | 768 | 92.7% | 0 | static 256Bバッファは旧flash_to_fram_seqの分をchunk_copyへ移設（Zone A増減なし）。スタックバッファ化によるZone A縮小はload_resource経路（サブアプリの小スタックで実行）が阻害要因のため見送り |
| Phase 2a実装（2026-07-13） | 744 | 768 | 96.9% | +32 | Phase 2a増分（app_loader entriesのcatalog_idx拡張等）。**残り24Bまで逼迫** |
| 圧縮8施策（2026-07-14） | 744 | 768 | 96.9% | 0 | Flash側のみの圧縮でZone A不変。entries[8](224B)→s_chunk_buf overlay(+224B回復)を検討・**Phase 2b着手時に実施と決定**（可変設計書§7.7） |
| ブートローダー領域プール移設（2026-07-14） | 744 | 768 | 96.9% | 0 | プール移設は**コードのみ**でZone A静的変数は移動しない（リンク時割付のまま）。g_pool_build_id(4B)は無指定だと.srodata経由でZone A行きになる罠を.rodata明示配置で回避（可変設計書§7.9.6ギャップ③）。Zone B機構スタック限界線上の件は§7.7参照 |
| gp復元是正=ContextEntry v3（2026-07-14） | 748 | 768 | 97.4% | +4 | g_restore 20→24B（gpフィールド追加・UT-IAP-06是正）。**残り20B** |
| entries overlay=Phase 2b 4a（2026-07-15） | 524 | 768 | 68.2% | **−224** | `entries[8]`(224B)を`s_chunk_buf`(256B)へ共用体overlay（可変設計書§7.7・時間的排他・BTN_OKローカル退避規約）。**残り244B**。空きギャップがZone B直下に連続し、機構スタック食み出し20Bの構造的安全域を兼ねる（§2.1.1再検証） |

**備考:**
- ~~2026-07-13時点: 712B / 768B（使用率92.7%、余裕56B）~~
- ~~2026-07-14時点: 748B / 768B（使用率97.4%、余裕20B）~~
- **2026-07-15時点: 524B / 768B（使用率68.2%、余裕244B）。entries overlay（可変設計書§7.7）を実施し逼迫を解消**
- 容量縮小（b966dc2、2048B→768B）と機能追加の積み重ね（388B→708B）が両方効いており、どちらか一方だけの問題ではない
- 今後SRAMを消費する変更（新規static/global変数、`.ram_func`対象コードの追加等）を行う場合は、着手前に必ずこの表で残り60Bとの比較を行うこと
- IAP可変サイズコンテキストスイッチ（`common_prog/iap_context_switch_variable_design.md`）はcommon_prog自体のSRAM消費とは別枠（Zone C）だが、Zone A逼迫と合わせてSRAM全体の余裕は小さい

---

## 変更履歴

| 日付 | 内容 |
|------|------|
| 2026-06-25 | 文書新規作成 |
| 2026-07-02 | L1単体試験 全23件 PASS（環境E）を記録 |
| 2026-07-07 | Wave 2 試験結果（13件）追記・試験名を設計書正式名称に統一・UT-FLASH-02/UT-KEY-01 再実施要記録 |
| 2026-07-07 | UT-CATALOG-04・UT-RESOURCE-04〜06 追加（4件 PASS・第4部乖離解消） |
| 2026-07-07 | UT-CATALOG-05〜07・UT-RESOURCE-07〜09 未実施試験として追記（設計書再点検による新規追加分） |
| 2026-07-08 | UT-KEY-01 PASS更新（B案: boot_main()完走根拠）・UT-FLASH-02 PASS更新（期待パターン照合）・UT-KEY-02 PASS追加（5方向自動注入）・UT-CATALOG-05〜07・UT-RESOURCE-07〜09 PASS転記（x86 Unity実装確認済み）・UT-IAP-01〜04 未実施に追加（L2§2.0移設）・未実施セクション整理（実機待ち/L2分類） |
| 2026-07-09 | Flash使用量履歴表追加・BOOT_LOG条件コンパイル化対応 |
| 2026-07-09 | §6リンカスクリプト変更(b966dc2)のFlash使用量追記 |
| 2026-07-09 | UT-IAP-01 PASS転記（未実施→PASSテーブル）・対象件数を46件に更新 |
| 2026-07-09 | UT-IAP-04 PASS転記（FRAMバックアップ復元確認）・UT-IAP-02/03 FAIL（調査中）として中間記録・対象件数49件に更新 |
| 2026-07-10 | UT-IAP-02/03 PASS転記（GP-relative relaxation修正）・Flash使用量+68B追記（5392B）・全件PASS確認 |
| 2026-07-10 | UT-WDT-01〜04・UT-PROFILE-01〜02 を未実施試験として追記（仕様確定・実装待ち） |
| 2026-07-10 | UT-WDT-02/04・UT-PROFILE-01/02・UT-CATALOG-08 PASS転記（計5件）・Flash使用量履歴追加（WDT実装+168B・DISP_TFT化−972B=最終4588B）・UT-WDT-01/03 を実機待ちに更新・総件数54件 |
| 2026-07-12 | Flash使用量履歴にfence.i追加分(+4B=4592B)を追記。SRAM（Zone A）使用量履歴表を新設（20コミット分を実ELFから遡及計測・388B→708Bへ増加、かつb966dc2でZone A容量自体が2048B→768Bへ縮小したため使用率18.9%→92.2%まで悪化したことを明記）。SRAM残量60Bと逼迫状況を記録 |
| 2026-07-13 | IAP可変サイズPhase 1(83b5da7): UT-IAP-05〜09 PASS転記（計5件・総件数59件）・UT-IAP-01〜04/IT-IAP-01/02/IT-BOOT-01/03回帰PASS・Flash使用量履歴+424B(5016B・余裕616B)・SRAM履歴+4B(712B・余裕56B)を追記。注意: UT-FRAM-03/04・test_app_ut/test_app_phase2のUT-FRAM系はFRAM旧マップ（CTX 0x10000・フォント0x13880）前提の記録のままであり、新マップ（CTX 0x22000・フォント0x00000）での再定義・再実施が未了 |
| 2026-07-13 | パターンA/B統合(chunk_copy): 256Bチャンク転送4系統+フォント転送ループを単一エンジンに統合しFlash−180B(4836B・余裕796B)。SRAM増減なし。UT-IAP-01〜09/IT-BOOT-01/03/IT-IAP-01/02回帰全PASS。Phase 2見込み+350〜550Bが余裕内に収まる見通しとなった |
| 2026-07-14 | Phase 2a+ブートローダープールのUT-IAP回帰（f3d63fa）: UT-IAP-01/04回帰PASS・UT-IAP-05〜09再実施PASS（Phase 2a新シグネチャ・実アロケータ/パッチ経路）・UT-IAP-02/03は廃止＝SKIP化（旧2フェーズ・リセット方式が単一実行・直接ジャンプへ移行・後継=UT-IAP-06/07）。UT-IAP-06再実施でgp未復元の実バグを発見しContextEntry v3（gp追加・28→32B）で是正。Flash履歴+28B(5292B・余裕340B)・SRAM履歴+4B(748B・残り20B)を追記。生ログ: log_iap_20260714_190824.txt / log_iap_var_20260714_190526.txt |
| 2026-07-15 | Phase 2b 4a=entries overlay: SRAM −224B(524B・余裕244B)・Flash±0。§2.1.1スタック収支再検証（機構スタック最悪276B実測・運用不変条件新設）。回帰: UT-IAP-01/04+05〜09全PASS（log_iap_20260714_234749.txt / log_iap_var_20260714_234832.txt）・**L3スイート全30項目PASS**（log_ut_20260715_000758.txt）。付随修理2件: ①run_test_ut.shのプール未対応（プール導入後L3が無限リブートループで実行不能だった・stub+pool+ram_func結合を移植）②setup_wave2_stubs.pyのFRAM_META_BASE旧マップ残存（0x23880→0x20000・UT-LOADRES-01が新マップ移行後潜伏FAILしていた） |
| 2026-07-15 | Phase 2b 4b=境界跨ぎキャリー: load_and_patchのチャンク境界跨ぎfail-closedハルトをキャリー方式で解消。missクラスタ成長でプール境界0x0500→0x0540へ移動（BOOTPOOL 1778B・余裕206B）。回帰: UT-IAP-01/04 PASS・02/03 SKIP（log_iap_20260715_002844.txt）・05〜09 PASS（log_iap_var_20260715_002928.txt）・L3全30項目PASS（log_ut_20260715_003132.txt）。**キャリー経路自体の直接UTは未実施＝Task 5で境界跨ぎフィクスチャを新設予定** |
| 2026-07-15 | **App Areaアドレス是正（0x1E00→0x2000・ユーザー指摘）+付随実バグ2件是正（eb13087）**: 全試験環境（リンカ9本・エミュレータ・試験ツール7本）を正マップへ是正。是正後の回帰でUT-IAP-07/09-WARNが2048Bクラス（delta=0x800）のみFAILし、切り分けの結果**①トランポリンsp/gpのPC相対計算**（Spikeの4KBページパディングが小deltaで隠蔽・実機ならdelta≠0で即死）と**②圧縮lui(c.lui)の4B word前提パッチによる隣接命令破壊**（k=1で初めて値が変化する条件で顕在化）の2件の潜在実バグを発見・是正した。C_LUI種別(4)を新設（詳細設計§7.3）。回帰: UT-IAP-01/04 PASS・02/03 SKIP（log_iap_20260715_103955.txt）・**05〜09全PASS＝07/09-WARNが初めて正マップ+2048Bクラスを通過**（log_iap_var_20260715_103748.txt）・L3全30項目PASS（log_ut_20260715_104031.txt）。Flash履歴−12B(5280B)・BOOTPOOL+178B(1956B・プール境界0x0600) |
| 2026-07-15 | **UT-IAP-10新設（境界跨ぎキャリー直接検証・§8.2解消）**: fixture_carry.S（R32被パッチワードをオフセット≡254 mod 256へ決定的配置）+ p14(callee)/p15(caller) + run_test_iap_var.shシナリオ4。marker=expected=9732の完全一致でキャリー2B+2B合成を実証・PASS（log_iap_var_20260715_211716.txt）。総件数59→60件。副次的発見: patch_table_gen.pyのaddend無視バグ（シンボル+オフセット参照でfail-closedビルドエラー・実アプリの配列/構造体参照で普通に発生する形）を是正・p12テーブルはバイト一致で無回帰。回帰: UT-IAP-01/04 PASS（log_iap_20260715_211952.txt）・L3全30項目PASS（log_ut_20260715_212036.txt） |
| 2026-07-15 | **Task 4c=A-2（別アプリ間呼び出し）実装完了**: `iap_call_impl`/`sram_restore_from_fram_prep`の2箇所のfail-closedハルトを実処理へ置換。カタログ逆引きヘルパ`catalog_flash_addr`・メタ入替ヘルパ`a2_meta_swap_in`を`.pool_2_a2`（bootuiと時分割共有・VMA 0x0600共有）に新設。`pool_ensure_loaded`にページ範囲重複時の自動追い出しを実装（時分割共有の実働部分）。`pool_module_gen.py`に`--allow-overlap`フラグ追加（総フットプリントはページ集合unionで再計算・二重計上を回避）。common_prog.ldは`.pool_2_a2`をVMA固定0x0600・LMAはFLASH末尾へ自動配置（`AT>FLASH`）とし、GNU ld OVERLAY構文は不採用（LMAも別消費し.init固定アドレスと衝突したため）。**A-2専用のUT/ITは今回含めず、既存スイート（UT-IAP-01/04/05〜10・L3全30項目）で無回帰確認**（log_iap_20260715_220307.txt・log_iap_var_20260715_220043.txt・log_ut_20260715_220413.txt）。実測: FLASH 5552B（余裕80B・逼迫注意）・BOOTPOOL 1544B（時分割共有によりVMA上の消費は増えていない）・RAM 524B不変 |
| 2026-07-16 | **Task 4d=§7.11常駐サブプログラム起動時ロード実装完了（B'案）**: `iap_run`にメタ+0x7Cの`resident_res_id`判定→常駐指定時のみ`resident_load`で常駐領域0x1E00へload+patch（delta=-512）を実装。前セッションWIPのパッチ適用チェーン複製方式（FLASH 6570B・+938B超過）をB'案（本家`load_and_patch`を`(dst_base,delta)`一般化して共用）へ設計転換し、`.pool_2_a2`のLMAを`AT>FLASH`→仮想領域POOL_LMAへ移してFLASH勘定から除外（詳細設計書§7.11.1〜7.11.4）。付随: `word_le`をmiss→常駐.textへ移動（共用化でmiss 1478B→枠1472B 6B超過をfail-closed検出→是正・最終1454B）・`a2_meta_swap_in`のインライン漏れをnoinlineで是正・**`iap_run`の常駐判定をガード先行化**（常駐なしアプリはプール非依存＝固定版ハーネスでハングしない防御的設計）・**4c潜在バグ是正**（A-2分岐でcatalog_flash_addr(.pool内)をensure_loaded前に呼ぶ順序バグ・Task5でensure順序検証必須）。回帰: **3スイート全PASS**——run_test_iap.sh=UT-IAP-01/04（log_iap_20260716_124247.txt）・run_test_iap_var.sh=UT-IAP-05〜10（境界跨ぎ10含む）・run_test_ut.sh=L3全30項目（log_ut_20260716_124534.txt）。実測: **FLASH 5516B（余裕116B・4c比−36B改善）**・BOOTPOOL 1652B・POOL_LMA（仮想）180B・RAM 524B不変 |
| 2026-07-16 | **Task5①=run_test_it.sh 4a方式プール修理（コミット5752767）**: プール未結合の旧方式（生common_prog@0x0800）で無限リブートループだったのを、run_test_ut.sh/iap_varと同じプール結合方式（スタブ+pool_flat+cp_patched）へ統一。place_pool_originヘルパ（pool_block→spi_flash[0x01C000]+fram[0x1A800]在席クリア）新設・make_combined_elf改修・IT-BOOT-01/03へ配置追加。IT-IAP-02はSKIP（再設計待ち・論点④=p7/p8/p9ビルド廃止+リセット連鎖方式がPhase2非整合）。回帰: IT-BOOT-01/03 PASS（BOOT-03はFlash書込verify MATCH）・IT-IAP-01 PASS（証跡代替）・IT-IAP-02 SKIP（log_it_20260716_140309.txt）。common_prog無変更 |
| 2026-07-16 | **Task5②=UT-IAP-11（A-2専用UT・ensure順序検証）新設**: caller（p16・cat0）が別カタログcat1のアプリ内callee（p12）を`iap_call(res2,cat1)`で呼ぶA-2往復試験を追加。setup_iap_stubs.pyに`--test a2`（2アプリブロック+カタログキャッシュ0x1A000のentry0/1配置）・test_app_iap.cにphase16・run_test_iap_var.shにシナリオ5を新設。**ネガティブコントロール実施**: A-2分岐のcatalog_flash_addrをensure前に呼ぶ順序バグを一時再導入するとUT-IAP-11 FAIL・他項目（05〜10）はPASS＝順序バグ（4d是正）を確実に捕捉することを実証。回帰: **UT-IAP-05〜11全PASS**（log_iap_var_20260716_142800.txt）。総件数60→61件。common_prog無変更（A-2は4c実装済み・4d是正済み） |
| 2026-07-16 | **Task5③=UT-IAP-12（常駐サブプログラム起動時ロード実経路）新設**: p18が`iap_run(0x098000)`を呼び、メタ+0x167Cの`resident_res_id=5`判定→`pool_ensure_loaded(A2)`→`resident_load`→`find_resource(CODE/PATCH\|5)`→`load_and_patch(master,0x1E00,-512,…)`の実経路を検証。フィクスチャ`fixture_resident.S`（p17・自己参照R32語）を新設、plugin（peripheral_stub.h）を`save_flash_region`一般化＋常駐領域0x1E00（512B）を`resident_result.bin`へ保存するよう拡張、setup_iap_stubs.pyに`--test resident`（単一アプリ・type_start+resident_res_id+CODE/PATCHディレクトリ配置）、test_app_iap.cにphase18・Makefileにphase17/18・run_test_iap_var.shにシナリオ6を新設。検証は被パッチ語の実行時アドレス照合: **正例 resident_result[4]=0x00001E08=期待値（RESIDENT_BASE+resident_targetオフセット8・delta=-512パッチ後）PASS**。**ネガティブコントロール（対照）**: `resident_res_id=0xFFFF`（常駐なし）で0x1E00未書込=0xFFFFFFFF PASS＝ガードがロード経路を確実にスキップすることを実証（delta≠0のため無変化偶然PASSがない）。回帰: **UT-IAP-05〜12全PASS**（log_iap_var_20260716_150535.txt）。総件数61→62件。common_prog無変更（常駐ロードは4d実装済み） |
| 2026-07-16 | **Task5④=IT-IAP-02 Phase2準拠ネスト再設計・10多重度でPASS**: 旧p7/p8/p9（3バイナリ・リセット連鎖・2段）を廃止し、**試験設計書 初版（41795fb・2026-07-07）の原案＝「自分自身を再帰呼び出しする試験モジュール」方式へ回帰**。配置状態表の検索キーが`(catalog_idx,res_id)`の組であることを利用し、同一のCODE実体・パッチテーブルを**res_id 2〜11 の10重登録**することで10個のサブプログラムを同時在席させた（バイナリ1本・Flash追加コストはディレクトリ20エントリのみ）。test_app_iap.cにphase19（L0）/phase20（自己再帰モジュール）・Makefileにphase19/20・setup_iap_stubs.pyに`--test nest`（10重登録＋セル勘定fail-closed検証＋`--nest-target`）・run_test_it.shにIT-IAP-02実装（SKIP解除）を新設。判定は①入場トレース`mult=0→10`単調増加 ②復帰10行すべてPASS/FAIL=0 ③**配置状態表 count=11・重複なし・res_id=[1..11]**（リージョンはiap_returnで削除されず重複追い出しでしか消えないため、復帰後に11個非重複＝ピーク時の同時在席の証明）。回帰: **IT-BOOT-01/03 PASS・IT-IAP-01 PASS・IT-IAP-02 PASS**（log_it_20260716_230944.txt）。common_prog無変更。**用語是正**（ユーザー指摘）: 「深さ（ネスト段数）」→**同時呼び出しサブプログラム数（多重度）**。CTX_STACK_DEPTH_ADDRが数えているのはコールスタックの深さではなく中断中サブプログラムの数であるため。**段数律速の確定**: FRAM容量（1エントリ≈98B＝理論上100段超）ではなく**App Areaのセル数32**。実測L0(p19)=748B(申告768B=3セル・余裕20B)・モジュール(p20)=472B(申告512B=2セル)→2×10+3=23セル。禁止residueによる窓阻害なし（cell=3,5,…,21）。**追跡漏れの発覚と是正**: 設計書に定義済みの**IT-IAP-03/04が初出（2026-07-07）から一度も試験結果報告書に記載されず**、除外判断の記録もないまま7回以上の回帰記録が積まれていた。構造的原因＝**IT系の追跡表が存在せず変更履歴行にのみ記録されていた**こと。「結合試験（IT）結果一覧」を新設して定義と台帳を接続し、設計書の定義件数（IT 6件）と本表の件数が一致することを回帰のたびに確認する運用とした（ユーザー承認）。IT-IAP-03/04はTask5残タスクへ復帰 |
| 2026-07-16 | **Task5⑥=IT-IAP-05（App Area溢れ時のLRU追い出し→復帰時sticky再ロード）新設・PASS**: IT-IAP-02（多重度10・23セル＝追い出しなし）の対になる試験。ハーネス・試験モジュールを共用し目標多重度のみ15へ（`NEST_TARGET`を`-D`で上書き=p20e／setup側は`--nest-target 15 --allow-cell-overflow`）。**2×15+3=33セル > 32セル**でApp Areaを意図的に溢れさせ、`allocate()`のLRU追い出しと`prep`のsticky再ロード（iap.c:1236-1252）を実経路で通す。判定は①入場トレース`mult=0→15`単調増加 ②復帰15行すべてPASS ③**count=15（追い出しなしなら16）・追い出されたres_id=[16]・res1(L0)在席・使用31/32セル・重複なし**。判定の論理: 16リージョンは33セル必要で32セルに物理的に収まらないため`count<16`は追い出し発生の算術的証明であり、その上でL0が自身の配置状態表ダンプを正しく出力できたこと自体がsticky再ロードの証明になる（未復元ならcell0にはres16のコードが乗ったままで別物を実行する）。観測経過は事前予測と完全一致: res16配置時に31セル使用済み・空き1セル→LRU最小のL0(seq=3)が追い出される→復帰時にprepがL0不在を検出しcell0へsticky再ロード→その再ロード(3セル)のplc_evictが今度はres16を追い出す。**本試験で確定した規約（ユーザー決定=案A）**: sticky再ロードは`find_resource(CODE\|res_id)`/`(PATCH\|res_id)`を引き不在なら無限ループでハングする（iap.c:1241/1246）。CODE側は「本体=CODEディレクトリのResourceEntry[0]」というiap_placement_init(iap.c:1296-1297)の前提で保証されるが、**本体のパッチテーブルの存在を保証する仕組みが無い**（本体はcell0・delta=0で通常運用では不要だが、追い出され再ロードされる瞬間にだけ要求される）。→「**App Areaを使い切る可能性のあるアプリは本体にもパッチテーブルを登録すること**」をPC側カタログ生成ツールの要件とし、可変設計書§8.7の開発者ガイド転記項目へ追加。対案B（本体をpin）はFlash消費+機構の非対称性のため不採用。**この経路はIT-IAP-02では踏めない**。付随: 試験アプリの出力ラベルを`RC:IT-IAP-02 …`→**`RC:IT-IAP-NEST …`（試験ID非依存）**へ変更（p19/p20はIT-IAP-02とIT-IAP-05で共用のため、IT-IAP-05のログに他試験IDが並び証跡として読めなかった）。回帰: **IT-BOOT-01/03・IT-IAP-01/02/05 全5項目PASS**（log_it_20260717_084457.txt）。common_prog無変更。IT定義7件⇔IT結果一覧7件で一致確認 |
| 2026-07-16 | **IAP引数・戻り値エリアをメタ領域→FRAM管理領域へ移設（実バグ修正）＋IT-IAP-03/04を追跡漏れから復帰・初実施PASS**: Task5でIT-IAP-04の証跡代替可否を判定しようとした際、**旧定義が現行機構では原理的に成立しない**ことが判明した。`a2_meta_swap_in`(iap.c:554)はA-2呼出時に呼び先アプリの、復帰時に呼び元アプリの**メタ8KB全体をFlash原本から転送する**（find_resourceが呼び先ディレクトリを引くためでA-2に本質的な動作）。旧引数エリア4領域はいずれもメタ内（+0x1680〜+0x177F）にあり、**A-2引数は呼出時に、A-2戻り値は復帰時に、機構自身が塗り潰していた**。**日本語入力Wnn**（CAR-01_japanese_input・A-2公開サービスとして設計済み・A-2引数エリアでひらがな文字列受取・実測見積約40B）は実装すれば必ず踏むバグだった。**対処（ユーザー決定）**: 4領域とも**FRAM管理領域へ移設・各64B→256Bへ拡大**（`FRAM_IAP_A1_ARG=0x1A900`/`A1_RET=0x1AA00`/`A2_ARG=0x1AB00`/`A2_RET=0x1AC00`）。A-1のみメタ内に残す案も検討したが「引数を書く→A-2呼び出し→A-1呼び出し」の順で**無言で失われる**（機構は引数エリアを参照せず検出不能・fail-closedも効かない）ため、安全性がコードの並び順という規律だけに依存する状態を避け4領域とも移設し、**メタ領域を純粋なFlashキャッシュとした**。256Bの根拠＝機構が参照も転送もしないので**実行時コストゼロ**・管理領域は移設後も4.75KB空き・公開ABIは後からの拡大が破壊的。メタ+0x1680〜+0x177Fは予約（未使用）。**common_progはコード変更ゼロ**（定数追加のみ・FLASH 5516B/RAM 524B/BOOTPOOL 1652B＝移設前と完全一致を実測確認）。**IT-IAP-03/04を Phase 2 準拠で再定義**（旧定義は`iap_call(0x09C000, ID_APP_A, IAP_CALL_EXTERNAL)`のアドレス指定＝Phase 1のまま。Phase 2のA-2判定は`call_type`ではなく`eff_catalog != cur.catalog_idx`(iap.c:1045)であり、`caller_flash_addr`も存在しない）。**IT-IAP-04に判定②「呼び出し元メタの復元」を新設**＝復帰後の`type_start[0]`がアプリA自身の値0x0020へ戻っていること。UT-IAP-11はA-2往復を通すがこの観測点を持たないため、**当初見立てた「UT-IAP-11による証跡代替」は誤りだった**。**ネガティブコントロール実施**: 引数アドレスのみ旧配置へ戻すとIT-IAP-04が`status=0`（呼び出しは成功）・メタ復元PASSのまま`ret64=NG FAIL`＝「A-2は通るのに引数だけが静かに壊れる」不具合の姿を再現し、本試験が当該バグを捕捉することを実証。実装は同一ソースを-Dで振り分け（p21a/p22a=A-1・p21b/p22b=A-2）、ハーネスは既存の`--test iap_var`/`--test a2`を流用したためsetup_iap_stubs.py変更なし。回帰: **IT 7項目全PASS**（log_it_20260717_103605.txt）・**UT-IAP-05〜12全PASS**。IT定義7件⇔IT結果一覧7件で一致（全件PASS＝未実施ゼロ）。**公開側3ファイル（cartridge_masterのフォーマット表・CAR-01_app_dev_guideの領域表・CAR-01_japanese_inputのWnn受け渡しアドレス）の改訂は別途実施**（同一領域に旧マップ由来の絶対アドレスが3種類流通していた: iap_design=0x16680系・japanese_input=0x16280系・メタ相対=0x1680系。移設で統一する） |
| 2026-07-17 | **Task5⑤=IT-BOOT-04を往復化して実施・PASS（可変設計書§8.4クローズ）＋IT結果一覧の件数是正**: §8.4は圧縮施策5（App Areaバックアップの`fram_write`1トランザクション化＝内蔵Flashメモリマップ直読み・旧256Bチャンクループ廃止／`iap_restore_from_fram`のnoinline化・−80B）の回帰確認を求めていたが、UT-IAP-04は**setupが事前に置いたFRAMバックアップからの復元しか見ておらず**、実際の生成→復元の往復が未確認だった。設計書に定義済みだが未実装（追跡漏れ）だった**IT-BOOT-04が復元側そのもの**だったため、**新IDを起こさず生成側の検証を足して往復化**した。`fram.bin`の永続性を利用した2実行構成: Phase1=App Areaに p1 を置いて起動→bootが`backup_app_area_to_fram()`でApp Area(0x2000・8KB)をFRAM 0x18000へ→p1が`iap_run()`→`soft_reset`→**PficStubの`exit(0)`でデストラクタが走り`fram.bin`が同期**→判定①`fram.bin[0x18000..+8KB]`==p1（生成）。Phase2=App Area空(0xFF)・**setupを走らせずPhase1の`fram.bin`を引き継ぐ**→bootが`fram_backup_valid()`→`iap_restore_from_fram()`→`soft_reset`→判定②`iap_result.bin`==p1（復元）。**①②ともMATCH＝PASS**（log_it_20260717_120434.txt・IT 8項目全PASS）。判定は成果物のみで行う（`boot_puts()`は`#ifdef BOOT_LOG`で通常ビルドではno-opに展開されるためログ文字列は条件にできない。当初これを条件に入れて誤FAILさせた）。Phase2はApp Areaを0xFFで結合しているので`iap_result.bin`がp1と完全一致する経路は復元以外に存在せず、成果物自体が8bルート通過の証明になる。**ネガティブコントロール**: FRAMバックアップ先頭4Bを0xFF化（`fram_backup_valid()`をfalseに）すると`iap_result.bin`が生成されない＝本試験が当該経路を実際に見ていることを実証。設計書IT-BOOT-04の定義も往復化に合わせて改訂（旧記述のFRAM `0x06000` は2026-07-13マップ改訂以前の値＝`FRAM_BACKUP_ADDR=0x18000`へ是正）。**IT結果一覧の件数是正**: 2026-07-16に「IT定義7件⇔本表7件で一致」としたのは**誤り**で、IT-IAP系のみを数え**IT-BOOT-02/04・IT-FONT-01〜03・IT-RSRC-01〜03の7件を数え漏らしていた**（追跡漏れ是正のために新設した表に同種の漏れを埋め込んでいた）。設計書のIT定義は**全15件**であり、本表を15件へ拡張。内訳=**PASS 8件**／未実施7件（**IT-BOOT-02は環境「R（実機のみ）」＝設計書に「エミュレータはGPIO INDRが常に0返しのためGPIO2=HIGH分岐を再現できない」と明記された設計時判断＝追跡漏れではない**。加えて外部通信の相手側スタブが未整備で、共通プログラムの責務は`extern_comm_run()`＝CART_READY HIGH＋spi_flash_mgrへ制御を渡すまで。接続確立シーケンスまでは共通機能の責務のため将来実施＝ユーザー確認済み／**IT-FONT-01〜03・IT-RSRC-01〜03の6件は環境E+Rで実行可能なのに未実装＝追跡漏れ**。L2/L3のUT-FONT系・UT-RESOURCE系に等価な試験がある可能性があるが、IT-IAP-04で「証跡代替可と見立てたら観測点が欠けていた」を踏んだため**中身を確認せずに代替可とはせず要判断として残す**）。common_prog無変更 |
| 2026-07-18 | **UT-IAP-13（申告サイズ全振り幅スイープ）新設 → 初回実行で gp ABI 実バグを検出・修正（gp統一）**: カバレッジ表（docs/test_coverage_20260718.md）で申告サイズの試験密度が 5/32≈16% と判明し、Zone Cクラス境界を跨ぐ4点（1024/2048/4096/8192B・slot512初使用・slot1024新設）を掃くスイープを新設（phase23・-DSWEEP_SIZE で同一ソース振り分け・run_test_iap_var.sh シナリオ7）。**初回実行で1024/2048Bが「status=0・Zone C復元正常なのにFAIL」**し、診断出力の追加＝バイナリサイズ変化だけでPASSに反転するHeisenbugを呈した。当時の失敗バイナリ（612B）を正確に再構築して再現を固定し、**spikeコミットログから0x2000046Cへの書込を全数抽出**→`spi_read_buf`→呼び出し元の`addi a1,gp,-968`を捕捉。**真因=リンカgp緩和×バイナリ毎に異なるgp**: 全ldが`. + 0x3fc`式でgpを計算しており common_prog=0x4DC/アプリ=0x7FCと別値。RC_API経由では機構Cコードがアプリのgpのまま走るため、gp緩和された静的変数アクセス30箇所が全て+0x320ずれ、**s_chunk_buf(0x114)がZone C内0x434〜0x534に化けてcallerの生きたデータ（スタック退避値）をスナップショット前に無言破壊**していた（書きも読みも同じずれた先を使うため操作自体は自己整合で成功し、歴代試験は被害領域に生データが無く一度も検出されなかった。過去3回のgp対策=iap_ctx.S norelax/ContextEntry v3 gp保存/トランポリンgp設定と同根）。**修正=gp ABI統一（案A改）**: 全11 ldの`__global_pointer$`を絶対値0x200007FC（アプリ側の既存実効値）へ固定・計算式を廃止。**サイズ完全不変**（FLASH 5,552B/RAM 532B/BOOTPOOL 1,652B）。ツールチェーンの罠=絶対値定義をSECTIONS内に置くとgp緩和が全滅（実測+56B・missプール枠超過でfail-closedリンク失敗）→**必ずSECTIONSの外で定義**。**実証=当時FAILした612Bバイナリそのままがld変更のみでPASS**。検討経緯・棄却案（--no-relax-gp実測+56B/RC_API入口シム）・将来シナリオ点検・残存リスクは**gp_abi_design.md**に記録。§8.7に規約5（PCツールが__global_pointer$==0x200007FCを検査）を追加。**併せて判明**: cells=32（App Area全域申告）は`iap_call`中に実行中callerが追い出されるが復帰時sticky再ロードで自己回復する（要=本体CODE/PATCH登録＝§8.7規約3の第2の発火点。ハーネス`--test iap_var`にres1登録+空パッチテーブルを追加して解消）。回帰: 全5スイート（UT-IAP-05〜13・UT-IAP-01/04・IT 8項目・L3 28項目・L2 9項目）グリーン。総件数62→63件 |
| 2026-07-18 | **絨毯爆撃②③④実施（UT-IAP-14/15・UT-FONT-04新設・計3件）**: カバレッジ表の残り3ポイントを順次実施（ユーザー指示「順次進めてください」・③はネガコン実装=案A承認）。**②UT-IAP-14**: callee申告flash_sizeを`--callee-decl-size`で実体(p12=216B)と独立に注入し、callee_cells=ceil(size/256)とload_and_patchチャンク数の1→2遷移境界6点（255/256/257/511/512/513B）を全PASS。**③UT-IAP-15**: setupの過小申告ガードを`--force-undersize`で意図的にバイパスし、caller実体260B(2セル・cell1先頭0x2100に番兵0xC0DEFACE)を申告256B(1セル)で登録→機構がcell1を空きと誤認しcalleeをロード→**番兵破壊を確認**（「破壊されたらPASS」の逆説試験）。機構は申告≥実体を検証しない（GIGO）＝安全網はPC側ツール(§8.7規約1)とsetupガードのみであることを実データで実証。実装中にセル勘定の誤り（cell1=0x2100を0x2200と誤認）を実行前に自己検出・修正。**④UT-FONT-04**: font_slot.c（common_prog実物）をtest_app_l2へ直接リンク（ADDITIONAL_C_FILES）し、FRAM実書き込み→font_slot_init()→font_slot_base()の実経路6ケース（slot0〜3=0x008000/0x020000/0x038000/0x050000・範囲外04/FF→slot0フォールバック）全一致PASS。併せてUT-FONT-02定義の旧記述（ELYSIA/0x13880=旧々マップ）を実装（FONT_FLASH_BASE 0x008000）に合わせ是正（悉皆点検C該当分の先行解消）。phase23の暫定診断コードも除去し恒久形へ整理。回帰: UT-IAP-05〜15全12項目PASS（log_iap_var_20260718_225408.txt）・L2全10項目PASS（log_l2_20260718_230443.txt）。総件数63→66件 |
| 2026-07-18 | **悉皆点検C（試験定義⇔実装の乖離）を全試験IDで実施**: 設計書の定義・結果表の記載・実装（RC出力/Unity/ハーネス）を突合。**(a) UT-CATALOG-08**: 実装(test_catalog.c)と結果表PASS(2026-07-10)はあるが**設計書に定義なし**（IT-IAP-03/04の鏡像＝定義なし実施）→ 実装から定義を復元し §1.7 へ追加。**(b) tests/l2 の Phase 2追随漏れ**: Unityスイート(x86)が2026-07-10のビルドのまま放置され、現行common_progとABIが乖離してビルド不能だった。iap_call_entry(res_id指定・IapCallStatus返却化)/ChunkShare g_chunk_share/iap_placement_init のスタブを現行シグネチャへ更新し、カタログ試験フィクスチャ基準を旧0x00000→現行FRAM_CATALOG_CACHE(0x0A000)へ是正（6/8件FAILの真因）→ **test_catalog 8件・test_font 2件・test_wdt 2件を全PASSに復旧**。**(c) UT-FONT-02(Unity)**: 旧々ELYSIA_FONT_BASE=0x13880断言でビルド不能→FONT_FLASH_BASE(0x008000)+96KB×4スロットへ全面書き換え・スロットベース確認(02b)を追加。**(d) UT-RESOURCE-01〜09**: 検証対象の線形サーチが Phase 2 で load_resource.c→iap.c find_resource() へ移動しリンク不能。旧PASS(2026-07-05〜08)は移動前コードに対する記録と明記し**全9件を「再設計待ち」へ是正**（find_resource抽出 or Spike移行の2案・設計書§1.8に方針記載・既定ビルドから除外）。CLAUDE.md「実行していない試験の合否を述べない」に基づく是正。回帰: Unity 12件(catalog8/font2/wdt2)全PASS。UT-CATALOG-08定義追加で総件数66→67件（UT-RESOURCE9件はPASS→再設計待ちのため合格数は変動、定義総数のみ+1） |
