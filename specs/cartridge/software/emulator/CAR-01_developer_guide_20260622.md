# ラテカピューターカートリッジ CAR-01 アプリ開発者向け統合ガイド（ドラフトv1）

**2026年6月19日版**
**対象読者：CAR-01向けアプリ・ゲームを開発したい方**
**前提知識：C/C++の基礎**

> **本書の位置付けについて**
> 本書は以下の内部仕様書から、アプリ開発者に必要な情報を抽出・統合したドラフトである。
> - `cartridge_master_20260617.md`
> - `boards_A_board_hardware_20260617.md`（バッテリー表示の挙動のみ）
> - `boards_B_board_hardware_20260615.md`（音声まわり）
> - `boards_C_board_hardware_20260615.md`（ストレージまわり）
> - `boards_D_board_hardware_20260615.md`（入力・OLEDまわり）
> - `CAR-01_emulator_dev_20260622.md`（エミュレータ仕様）
>
> 公開配布前に、各章のレジスタアドレス・ピン番号等の詳細を元の個別仕様書と突き合わせて確認すること。特にプラットフォームAPIの最終的な関数シグネチャはPhase 2〜3の実装段階で変わる可能性がある。

---

## はじめに

CAR-01は、1979年のシャープ製ポケットコンピュータ「PC-2000/PC-2001（通称ラテカセ）」の筐体を使い、現代の部品で再構築したカートリッジ式ゲーム・アプリプラットフォームです。本体（UIAPduino）にカートリッジを挿して動作させる構成で、シューティングゲームからノベルゲームまで幅広いジャンルのアプリが開発できます。

実機カートリッジが手元になくても、本書のエミュレータを使えばPC・スマートフォン・タブレットのブラウザ上で開発・テストプレイができます。

---

## 1. CAR-01カートリッジとは

CAR-01は1枚の基板ではなく、役割の異なる4枚の基板（A〜D）とメイン基板（UIAPduino）で構成されています。

| 基板 | 役割 | アプリ開発者が意識すること |
|---|---|---|
| UIAPduino（本体） | CH32V003マイコンによるメイン処理 | アプリのロジックはここで動く |
| A基板 | 電源管理・バッテリー監視 | 基本的に意識不要（OS的な裏方） |
| B基板 | FM音源（YMF825）・ADPCM音声出力 | 効果音・BGMの再生 |
| C基板 | SPI Flash・FRAMストレージ | アプリ本体・セーブデータの保存先 |
| D基板 | ジョイスティック入力・日本語入力UI（OLED） | 入力受付・テキスト入力 |

```
                    I2C共通バス
          ┌──────────────┬──────────────────┐
     [A基板]電源管理   [D基板]入力・日本語UI  CH1115 OLED
          │
    [UIAPduino = CH32V003]
          │
    SPI共通バス
    ├── [C基板] SPI Flash + FRAM（ストレージ）
    ├── [B基板] FM音源 + ADPCM（音声）
    └── TFT液晶（240×240・表示）
```

---

## 2. 開発の始め方：エミュレータのセットアップ

### 2.1 必要なもの

- PC（Windows推奨）
- WSL2（Windows上のLinux環境）
- ブラウザ（配布時の表示確認に使用）

### 2.2 セットアップ手順

```bash
# 1. 依存パッケージのインストール（WSL2 Ubuntu 24.04の場合）
sudo apt update
sudo apt install -y build-essential autoconf automake libtool \
  device-tree-compiler libboost-regex-dev libboost-system-dev \
  python3 python3-pip git cmake gcc-riscv64-unknown-elf \
  libsdl2-dev    # SDL2（開発時ネイティブ表示用）

# 2. Spike（RISC-Vエミュレータ）のビルド
git clone https://github.com/riscv-software-src/riscv-isa-sim.git
cd riscv-isa-sim
mkdir build && cd build
../configure --prefix=$HOME/riscv-spike
make -j$(nproc)
make install
echo 'export PATH=$HOME/riscv-spike/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 3. car01_plugin（CAR-01専用拡張、配布物として提供予定）をロードして起動
spike -d --isa=rv32ec --extlib=./car01_plugin.so your_app.elf
```

### 2.3 動作確認の使い分け

**開発中の動作確認（推奨）**：SDL2版を使います。WSL2上にネイティブウィンドウが開き、ブリッジのオーバーヘッドなしにゲームを確認できます。シューティング等の高速描画ジャンルでも入力遅延の切り分けが正確に行えます。

**配布時の最終確認**：`serial_monitor.py`を起動し、ブラウザから`panel.html`を開きます。TFT画面・OLED表示・デバッグログ・音声出力を配布時と同じ環境で確認できます。

---

## 3. プラットフォームAPIリファレンス

CAR-01版・PC版（SDL2エミュレータ版）共通で使えるAPIです。`#ifdef`によって実機・エミュレータどちらでも同じソースコードがビルドできます。

```c
typedef enum {
    RENDER_BACKEND_SDL2,     // 開発時：WSL2上でSDL2ネイティブウィンドウ
    RENDER_BACKEND_WEBVIEW,  // 配布デフォルト：WebView+生ピクセル配列
    RENDER_BACKEND_NATIVE    // 配布オプトイン：SurfaceView(Android)/Metal(iOS)
} PlatformRenderBackend;

// 描画
void Platform_Init(int width, int height, PlatformRenderBackend backend);
void Platform_DrawPixel(int x, int y, uint16_t color);
void Platform_FillRect(int x, int y, int w, int h, uint16_t color);
void Platform_DrawBitmap(int x, int y, const uint8_t* bitmap, int w, int h);
void Platform_Present(void);          // フレーム転送

// 入力
uint8_t Platform_GetInput(void);      // ビットマスクで返す

// ストレージ（SPI Flash / FRAM）
void Platform_FlashRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramRead(uint32_t addr, uint8_t* buf, size_t len);
void Platform_FramWrite(uint32_t addr, const uint8_t* buf, size_t len);

// タイミング
void Platform_SetFrameRate(int fps);
bool Platform_NextFrame(void);
uint32_t Platform_GetTick(void);
```

画面解像度は240×240です。

### 3.1 レンダリングバックエンドの選び方

| バックエンド | 使用場面 | 向いているジャンル | 開発者の負担 |
|---|---|---|---|
| `RENDER_BACKEND_SDL2` | **開発中（推奨）** | 全ジャンル。ブリッジオーバーヘッドがないため入力遅延の切り分けが正確 | `libsdl2-dev`をインストール済みであれば特になし |
| `RENDER_BACKEND_WEBVIEW` | 配布時・デフォルト | ノベル・パズル・RPG等、大半のジャンル | 特になし。配布時のデフォルト |
| `RENDER_BACKEND_NATIVE` | 配布時・オプトイン | 弾幕シューティング等、フレーム精度がシビアなジャンル | `Platform_Init()`で明示指定するだけ。描画パス自体はSDKが提供 |

**基本的な開発フロー**：開発中は`RENDER_BACKEND_SDL2`で動作確認→完成したら`RENDER_BACKEND_WEBVIEW`または`RENDER_BACKEND_NATIVE`に切り替えて配布動作を最終確認することを推奨します。

#### 各バックエンドの技術的背景

**SDL2**はWindows/Linux向けのクロスプラットフォーム描画ライブラリ（zlib license）です。WSL2(Linux)上でのネイティブウィンドウ表示に使用します。Windows向け配布版では`SDL2.dll`をexeと同じフォルダに同梱するため、ユーザーは何もインストールせずにそのまま動かせます。

**SurfaceView（Android）**はAndroidのUIフレームワークから独立した専用描画サーフェスです。メインスレッドとは別スレッドから直接ピクセルを書き込めるため、60fps安定描画に向いています。UnityやUnreal EngineがAndroidで高速描画できるのもSurfaceViewを使っているためです。

**Metal（iOS）**はAppleが2014年に導入した独自グラフィックスAPIです。GPUへの命令をコマンドバッファにまとめる設計でオーバーヘッドが小さく、現在iOSの全ゲームアプリが使用しています。CAR-01エミュレータでの使い方は「240×240のテクスチャを画面にコピーするだけ」という最もシンプルな用途です。

なお、SDL2の内部実装はプラットフォームごとにSurfaceViewやMetalを使い分けています。モバイルでSDL2を経由するとラッパーの層が1つ余分に増えるだけなので、Android/iOSではSurfaceView/Metalを直接使う設計としています。

## 4. 入力（ジョイスティック）

| CAR-01実機 | エミュレータ（キーボード） |
|---|---|
| 十字キー上 | W / ↑ |
| 十字キー下 | S / ↓ |
| 十字キー左 | A / ← |
| 十字キー右 | D / → |
| Aボタン | Z / Space |
| Bボタン | X |

---

## 5. 画面表示(TFT・日本語フォント)

TFTは240×240、独自の「恵梨沙フォント」によるインデックスベースの日本語表示に対応しています。文字列はあらかじめ専用ツールでインデックス列に変換してから埋め込みます。

```bash
python char_convert.py encode "こんにちは"
# → [0x0034, 0x0068, 0x0012, ...]
```

画像素材は以下の形式に変換して使用します。

```
PNG → RGB565バイナリ(TFT直接書き込み用・通常のフルカラー素材向け)
PNG → 1bitバイナリ(白黒2値素材向け)
PNG → インデックスカラー256色 + 圧縮(背景画像など大きな素材向け)
```

---

## 6. 音声(FM音源・ADPCM)

CAR-01は2系統の音声合成を持ち、両方とも自動的にミックスされて出力されます。

- **FM音源(YMF825)**:16ボイスポリフォニック、4オペレータ、8アルゴリズム。BGM・主旋律向け
- **ADPCM**:CH32V003本体からのPWM出力。効果音・ボイス再生向け

レジスタへの直接アクセスは共通プログラム側で抽象化される予定のため、アプリ開発者は高レベルAPI(サウンドAPI、別紙で詳細化予定)を通して再生します。エミュレータ上ではWebAudio APIを通してブラウザでそのまま再生されるため、実機がなくても音の確認ができます。

---

## 7. セーブデータ

FRAM上にセーブデータ専用領域(16KB)が確保されています。アプリ固有のセーブデータはこの範囲内に書き込んでください。システム管理領域・共通プログラムバックアップ領域など他の領域への直接アクセスは避けてください。

---

## 8. バッテリー残量表示

エミュレータ上では、バッテリー残量は常に満充電として表示されます。電源管理やバッテリー監視はアプリ側で意識する必要はありません。

---

## 9. ツールチェーン一覧

| ツール | 用途 |
|---|---|
| `char_convert.py` | 文字列⇔恵梨沙フォントインデックスの変換 |
| `scenario_compile.py` | シナリオデータ(JSON)のバイナリ化(ノベルゲーム等) |
| `write_program.py` | 実機への書き込み(実機所有者向け) |
| `dummy_data_gen.py` | テストデータ生成 |

---

## 10. 配布パッケージ形式(.car)

アプリ・ゲームの配布形式として `.car` フォーマットを策定予定です(現時点では未確定)。確定後、本セクションに詳細を追記します。

---

## 11. 将来の拡張コネクタ対応

CAR-01には拡張コネクタを使った追加デバイス接続の構想があります。現時点ではアーキテクチャ上の拡張ポイントとしてのみ存在し、対応API等は未策定です。

---

## 12. 使用しているソフトウェアとライセンス

本エミュレータ・SDKは以下のオープンソースソフトウェアを使用しています。

| コンポーネント | ライセンス | 用途 |
|---|---|---|
| Spike (riscv-isa-sim) | BSD | RISC-Vエミュレーションコア |
| SDL2 | zlib license | 開発時ネイティブ表示（バイナリ配布時の表示義務なし） |
| fmfm.core(移植) | MIT | FM音源(YMF825)エミュレーション |
| YMF825公式サンプル(参考) | MIT(Yamaha Corporation) | レジスタ仕様リファレンス |

CAR-01 SDK自体もMITライセンスでの配布を予定しています。

---

*本書はドラフトv1です。各章の詳細は元の個別仕様書(cartridge_master、A/B/C/D各基板仕様書、CAR-01_emulator_dev)を一次資料として今後精査・更新します。*
