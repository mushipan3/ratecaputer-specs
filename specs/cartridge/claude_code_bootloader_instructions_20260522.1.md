# UIAPduinoブートローダ改変・書き換えプログラム 実装指示書

**対象: Claude Code**
**目的: UIAPduinoブートローダの改変とブートローダ書き換えプログラムの実装**
**Rev. 1.1 — 2026年5月22日**

---

## 0. この指示書の読み方

本指示書はClaude Codeへの実装指示である。詳細な設計背景は以下を参照すること。

- `cartridge_spec_master.md` — カートリッジ共通規格・CAR-01個別仕様（2.8節・ブートローダ構造）
- `bootloader_dev_spec.md` — 共通プログラム開発仕様書（セクション1.1・起動シーケンス）
- `bootloader_modification_guide.md` — ブートローダ改変手順書
- `bootloader_writer.c` — ブートローダ書き換えプログラムの雛形

**本指示書の成果物:**

| ファイル | 内容 |
|---------|------|
| `bootloader_custom/` | 改変ブートローダのビルド環境一式 |
| `bootloader_writer/` | ブートローダ書き換えプログラムのビルド環境一式 |
| `bootloader_custom.bin` | 改変ブートローダバイナリ（2KB以内） |
| `bootloader_writer.bin` | 書き換えプログラムバイナリ（USB経由で書き込む） |

---

## 1. システム構成の前提

### CH32V003内蔵Flash（16KB）の3層構造

```
0x0000 ┌─────────────────────────┐
       │ UIAPduinoブートローダ   │ 2KB（0x0000〜0x07FF）
       │ 【今回改変する対象】    │ ジャンプ先を0x0800に確認・修正
0x0800 ├─────────────────────────┤
       │ ラテカ共通プログラム    │ 6KB枠（0x0800〜0x1FFF）
       │ 【書き換えプログラムは  │ 今回はここに配置する
       │   ここに書き込む】      │
0x1C00 ├─────────────────────────┤
       │ 共通モジュール置き場    │ 512B（0x1C00〜0x1DFF）
0x1E00 ├─────────────────────────┤
       │ App Area               │ 8KB（0x1E00〜0x3FFF）
       │ IAPで随時書き換え      │
0x3FFF └─────────────────────────┘
```

### 処理の流れ

```
【今回やること】
1. rv003usbのブートローダソースを確認・改変（ジャンプ先を0x0800に）
2. 改変ブートローダをビルド → bootloader_custom.bin（2KB以内）
3. CRC32を計算してbootloader_writer.cに埋め込む
4. bootloader_writer.cをビルド → bootloader_writer.bin
5. USB経由でUIAPduinoに書き込む

【書き込み後の動作】
bootloader_writer.binが起動
    ↓
CRC32検証OK
    ↓
0x0000〜0x07FFを改変ブートローダで上書き
    ↓
App Area（0x1E00〜0x3FFF）を消去
    ↓
ソフトウェアリセット
    ↓
新ブートローダが起動 → 0x0800へジャンプ
```

---

## 2. Task 1: rv003usbブートローダの確認と改変

### 2.1 ソースコード取得

```bash
git clone https://github.com/YuukiUmeta-UIAP/rv003usb.git bootloader_custom
cd bootloader_custom
git checkout 7ae2940676df0e1dcdad906ea7b1c9fff6c0b971
git submodule update --init --recursive
```

### 2.2 ジャンプ先アドレスの確認

ブートローダがタイムアウト後にユーザープログラムへジャンプする箇所を探す。

```bash
# ジャンプ先の記述を検索
grep -rn "0x800\|user_code\|UserCode\|user_app\|jump\|JUMP\|jr\|jalr" \
  bootloader/ --include="*.c" --include="*.S" --include="*.h"
```

**確認すべき内容:**
- ジャンプ先アドレスが `0x00000800` になっているか
- タイムアウト処理・通常起動処理の両方で `0x0800` へジャンプしているか

**修正が必要な場合の修正例:**

```c
/* C言語の関数ポインタによるジャンプの場合 */
/* 修正前 */
void (*user_app)(void) = (void (*)(void))(BOOTLOADER_SIZE);  /* 0x0800のはずだが確認 */
/* 修正後（明示的に0x0800を指定） */
void (*user_app)(void) = (void (*)(void))(0x00000800UL);
user_app();
```

```asm
# アセンブリのジャンプの場合
# 修正前（例）
li t0, 0x0000
jr t0
# 修正後
li t0, 0x800
jr t0
```

### 2.3 リンカスクリプトの確認

ブートローダのリンカスクリプトでFLASHサイズが2KB以内に制限されているか確認する。

```
MEMORY {
    FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 2K  /* 必ず2K以内 */
    RAM (xrw)  : ORIGIN = 0x20000000, LENGTH = 2K
}
```

2KBを超える場合はコンパイルエラーになる設定にすること。

### 2.4 ビルド

```bash
cd bootloader_custom/bootloader/  # またはブートローダのMakefileがある場所
make

# 生成されたバイナリを確認
ls -la *.bin *.elf
riscv32-unknown-elf-size *.elf
```

**成功条件:**
- バイナリサイズが2048B（2KB）以内であること
- `0x0000` から開始するバイナリであること

```bash
# バイナリを ../bootloader_custom.bin にコピー
cp bootloader.bin ../../bootloader_custom.bin

# サイズ確認
wc -c ../../bootloader_custom.bin
# → 2048以内であること
```

---

## 3. Task 2: CRC32計算とbootloader_writer.cへの埋め込み

### 3.1 CRC32計算とC配列生成

```bash
python3 << 'EOF'
import zlib, sys

# ブートローダバイナリを読み込む
with open('bootloader_custom.bin', 'rb') as f:
    data = f.read()

# 2KBにパディング（0xFF埋め）
data = data + b'\xff' * (2048 - len(data))
assert len(data) == 2048, f"サイズ異常: {len(data)}B"

# CRC32計算
crc = zlib.crc32(data) & 0xffffffff
print(f"// CRC32: 0x{crc:08X}")
print(f"#define BOOTLOADER_CRC32  0x{crc:08X}UL")
print()

# C配列生成
vals = [f'0x{b:02X}' for b in data]
lines = [', '.join(vals[i:i+16]) for i in range(0, len(vals), 16)]
print(f"static const uint8_t bootloader_image[BOOTLOADER_IMAGE_SIZE] = {{")
for line in lines:
    print(f"    {line},")
print("};")

print(f"\n// バイナリサイズ: {len(data)}B", file=sys.stderr)
print(f"// CRC32: 0x{crc:08X}", file=sys.stderr)
EOF
```

### 3.2 bootloader_writer.cの更新

`bootloader_writer.c`（雛形）の以下2箇所を上記の出力で置き換える。

**変更箇所1: CRC32定義**
```c
/* 変更前 */
#define BOOTLOADER_CRC32  0xFFFFFFFFUL

/* 変更後（python3の出力に合わせる） */
#define BOOTLOADER_CRC32  0x????????UL  /* 実際の値 */
```

**変更箇所2: バイナリ配列**
```c
/* 変更前 */
static const uint8_t bootloader_image[BOOTLOADER_IMAGE_SIZE] = {
    [0 ... BOOTLOADER_IMAGE_SIZE - 1] = 0xFF
};

/* 変更後（python3の出力を貼り付け） */
static const uint8_t bootloader_image[BOOTLOADER_IMAGE_SIZE] = {
    0x??, 0x??, ...  /* 実際のバイナリ */
};
```

---

## 4. Task 3: bootloader_writer.cのビルド

### 4.1 ビルド環境の確認

```bash
# ch32funが必要
ls ../ch32fun/ch32fun.mk  # なければgit cloneする
# git clone https://github.com/cnlohr/ch32fun.git ../ch32fun
```

### 4.2 リンカスクリプトの修正

**重要**: `bootloader_writer.c` は `0x0800` から配置しなければならない。
UIAPduinoブートローダが `0x0000〜0x07FF` を占有しているため。

ch32funのデフォルトリンカスクリプトを修正するか、以下の内容で
`bootloader_writer.ld` を作成する。

```ld
/* bootloader_writer.ld */
/* bootloader_writer専用リンカスクリプト */
/* UIAPduinoブートローダの後（0x0800〜）に配置する */

MEMORY
{
    FLASH (rx)      : ORIGIN = 0x00000800, LENGTH = 5632  /* 共通プログラム枠5.5KB */
    RAM (xrw)       : ORIGIN = 0x20000000, LENGTH = 2K
}

SECTIONS
{
    .text :
    {
        . = ALIGN(4);
        *(.text)
        *(.text*)
        *(.rodata)
        *(.rodata*)
        . = ALIGN(4);
        _etext = .;
    } > FLASH

    /* SRAMで実行するFlash書き換えルーチン */
    .ram_func :
    {
        . = ALIGN(4);
        _sram_func = .;
        *(.ram_func)
        . = ALIGN(4);
        _eram_func = .;
    } > RAM AT > FLASH

    _ram_func_lma = LOADADDR(.ram_func);

    .data :
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data)
        *(.data*)
        . = ALIGN(4);
        _edata = .;
    } > RAM AT > FLASH

    _sidata = LOADADDR(.data);

    .bss :
    {
        . = ALIGN(4);
        _sbss = .;
        *(.bss)
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        _ebss = .;
    } > RAM

    /* スタック */
    .stack (NOLOAD) :
    {
        . = ALIGN(8);
        _sstack = .;
        . = . + 1024;   /* 1KBスタック */
        . = ALIGN(8);
        _estack = .;
    } > RAM
}
```

### 4.3 Makefileの作成

```makefile
# bootloader_writer/Makefile

TARGET    = bootloader_writer
SRCS      = bootloader_writer.c

CH32FUN_DIR ?= ../../ch32fun

CROSS     = riscv32-unknown-elf-
CC        = $(CROSS)gcc
OBJCOPY   = $(CROSS)objcopy
SIZE      = $(CROSS)size

# ch32fun の共通定義を使う
include $(CH32FUN_DIR)/ch32fun.mk

# リンカスクリプトを上書き
LDFLAGS   := $(filter-out -T%,$(LDFLAGS))
LDFLAGS   += -T bootloader_writer.ld

# ビルド
$(TARGET).elf: $(SRCS) bootloader_writer.ld
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $(SRCS) \
	  -I$(CH32FUN_DIR) \
	  -L$(CH32FUN_DIR) \
	  -lch32fun
	$(SIZE) $@

$(TARGET).bin: $(TARGET).elf
	$(OBJCOPY) -O binary $< $@
	@echo "生成完了: $@ (サイズ: $$(wc -c < $@)B)"

.PHONY: size
size: $(TARGET).elf
	@echo "=== サイズ確認（目標: 6144B以内） ==="
	$(SIZE) $(TARGET).elf

.PHONY: flash
flash: $(TARGET).bin
	@echo "=== USB経由でUIAPduinoに書き込み ==="
	minichlink -w $(TARGET).bin flash
	@echo "書き込み完了。UIAPduinoが自動的にブートローダを書き換えます。"

.PHONY: clean
clean:
	rm -f $(TARGET).elf $(TARGET).bin $(TARGET).map *.o
```

### 4.4 ビルドと確認

```bash
make
make size

# 成功条件:
# - .text + .data が 6144B（6KB）以内
# - .ram_func セクションが存在すること（do_flash_write関数）
# - エラーなし
```

---

## 5. Task 4: 書き込みと動作確認

### 5.1 USB経由で書き込み

UIAPduinoをPCにUSB接続した状態で:

```bash
make flash
# または
minichlink -w bootloader_writer.bin flash
```

### 5.2 期待される動作

書き込み完了後にUIAPduinoが自動リセットして `bootloader_writer` が起動する。

**正常動作の場合:**
- PA1（CART_READY）がHIGHになる（LED点灯等で確認）
- 数秒後にリセットがかかる（App Area消去 + ブートローダ書き換え完了）
- 再起動後、USB接続でPCに認識される（新ブートローダが起動）

**CRC32検証失敗の場合:**
- PA1が100msで点滅する
- 書き換えは実行されない（文鎮化防止）
- 電源を切り直して原因を調査すること

### 5.3 書き換え成功の確認

```bash
# 新ブートローダで起動後、共通プログラムを書き込んでみる
minichlink -w ../common_prog/common_prog.bin flash

# boot_main()が0x0800から正常に起動すればOK
```

---

## 6. 完了条件チェックリスト

| 確認項目 | 方法 |
|---------|------|
| 改変ブートローダのジャンプ先が0x0800 | ソースコード確認 |
| bootloader_custom.binが2KB以内 | `wc -c bootloader_custom.bin` |
| bootloader_writer.binのFlash使用量が6KB以内 | `make size` |
| CRC32が正しくbootloader_writer.cに埋め込まれている | ソースコード確認 |
| .ram_funcセクションにdo_flash_writeが配置されている | mapファイル確認 |
| 書き込み後にCRC32検証OKになる | PA1の点滅なし |
| ブートローダ書き換え後に0x0800へジャンプする | 共通プログラムの起動確認 |

---

## 7. 注意事項

- **書き換え中は絶対に電源を切らないこと。** 失敗するとWCH-LinkEなしでの復旧が不可能になる。
- **CRC32が `0xFFFFFFFF` のままビルドしないこと。** 必ず実際のバイナリを埋め込んでからビルドすること。
- **改変ブートローダのサイズは必ず2KB以内にすること。** 超過するとApp Area（0x0800〜）が破壊される。
- **bootloader_writer.cのリンカ設定は必ず0x0800から開始すること。** 0x0000から開始するとUIAPduinoブートローダが破壊される。
- ch32funのデフォルトリンカスクリプトは `0x0000` から開始するため、**必ず `bootloader_writer.ld` で上書きすること。**

---

## 8. トラブルシューティング

### ビルドエラー: section '.ram_func' will not fit in region 'RAM'

`do_flash_write()` 関数が大きすぎてSRAMに収まらない場合。
関数を最小限に削ること。必要なら`FLASH_PAGE_SIZE`の値を確認すること。

### minichlink: device not found

UIAPduinoが書き込みモードになっていない。
リセットボタンを押しながらUSBを接続し直す。

### CRC32不一致（PA1点滅）

`bootloader_writer.c` の `BOOTLOADER_CRC32` と実際のバイナリが一致していない。
python3スクリプトで再計算してCRC32値を確認すること。

### 書き換え後にUSB認識されない

新ブートローダがUSB初期化に失敗している可能性がある。
rv003usbのUSBピン設定（D+/D-）がUIAPduinoのハードウェアと一致しているか確認すること。
