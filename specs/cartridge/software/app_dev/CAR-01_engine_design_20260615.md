# ラテカピューターカートリッジ ゲームエンジン設計目論見書

**2026年6月策定**  
**対象ハードウェア：CAR-01（UIAPduino版 / CH32V003）**  
**ステータス：設計方針確定・各エンジン詳細設計着手前**

---

## 1. 基本方針

### 1.1 エンジンプラットフォームとしての設計思想

CAR-01は単なるゲームカートリッジではなく、**持続的なコミュニティを持てるゲームプラットフォーム**として設計する。その核心は以下の3層ユーザー構造にある。

```
Layer 1：エンジン開発者
  2層ライブラリ・各種エンジンを改造・拡張
  C言語とCAR-01アーキテクチャの知識が必要
  提供物：ライブラリソース・エンジンソース

Layer 2：MODクリエイター
  シナリオエディタでパラメータ・素材を作る
  プログラミング不要
  提供物：シナリオエディタ・MODローダー・write_program.py

Layer 3：プレイヤー
  MODを選んで遊ぶだけ
  技術知識不要
  提供物：コンパイル済みエンジン・MODファイル
```

**ノベルゲームエンジン・ダンジョンエンジン・シューティングエンジン・アドベンチャーエンジンは全て同じ思想で設計する。** データとエンジンを分離することで、ゲーム制作のハードルが劇的に下がる。

### 1.2 エンジンラインナップ

```
SPI Flash システム領域：
  rc_novel_engine.bin      ノベルゲームエンジン（仕様書別途）
  rc_dungeon_engine.bin    ダンジョンRPGエンジン
  rc_shooting_engine.bin   シューティングエンジン
  rc_adventure_engine.bin  アドベンチャーエンジン（将来）
  rc_rpg_engine.bin        フィールド型RPGエンジン（将来）
```

### 1.3 MODエコシステムの構造

```
【制作者側】
  シナリオエディタ（PC）
      ↓
  mod.zip（配布パッケージ）
      ├ scenario.json   パラメータ・フロア定義
      ├ enemies/        敵スプライト画像
      ├ walls/          壁・床・天井テクスチャ
      ├ items/          アイテム画像
      ├ sounds/         効果音
      └ mod.json        MODメタ情報（名前・作者・バージョン）

【利用者側】
  MODローダー（PC）
      ↓ コンパイル・変換・転送
  SPI Flash上のMOD領域
      ↓
  カートリッジ起動 → MOD選択メニュー → プレイ
```

### 1.4 SPI Flash上のMODレイアウト

```
0x000000〜0x0FFFFF（1MB）：システム領域
  恵梨沙フォント（55KB）
  各エンジン本体
  ブートメニュー

0x100000〜0xFFFFFF（15MB）：MOD領域
  MODディレクトリ（先頭4KB）
    [MOD#0] アドレス・サイズ・名前・作者・バージョン
    最大127本のMOD登録可能

  MOD#0データ：
    scenario.bin   パラメータ・フロア定義
    sprites.bin    スプライトアトラス（RGB565）
    textures.bin   壁テクスチャアトラス（RGB565）
    sounds.bin     効果音データ
```

### 1.5 iap_call()との関係

エンジンはApp Area（8KB）に常駐して動き続ける。iap_call()は毎フレーム呼ばない。

```
起動時に1回だけiap_call()でエンジンをApp Areaに書き込む（110ms）
    ↓
エンジンがApp Areaに常駐したまま動き続ける
    ↓
ゲームデータはload_resource()でSPI Flash→FRAMに随時ロード
    ↓
iap_call()は階段・ドア・場面転換など無音タイミングだけ発生
```

---

## 2. 共通設計事項

### 2.1 PC版Platformレイヤー（SDL）

CAR-01完成を待たずにエンジン開発を先行できるよう、PC上でネイティブビルドできる抽象化レイヤーを最初から設計に組み込む。

```
Platformレイヤー：
  PC版（SDL/PNG出力）← エンジン開発・デバッグに使用
  CAR-01版（TFT・SPI）← 実機確認に使用
  どちらも同一のエンジンコードを使う
```

Catacombs of the Damnedのオリジナルソースにも`#ifdef _WIN32`でPC版ビルドが含まれており、同じ手法が実績あり。

### 2.2 エンジン単体試験のシーン構成

各エンジンは以下のシーン単位で単体試験を行い、全シーン通過後にMOD互換性検証（Arduboyゲーム移植）を実施する。

```
Scene 1：基本描画・移動テスト
Scene 2：弾・当たり判定テスト（シューティング）/ レンダリングテスト（ダンジョン）
Scene 3：敵AI・スポーンテスト
Scene 4：スクロール・背景テスト
Scene 5：フルゲームループテスト（スコア・残機・ゲームオーバー）
Scene 6：セーブ・ロード・フロア遷移テスト（ダンジョン）
```

### 2.3 フレームレートの標準設計

調査した全Arduboyゲームが60FPS設計であることが確認された。

```
標準FPS：60（全エンジン共通）
速度設計：speed = 定数 / FPS（フレームレート正規化）
FPS変更：シナリオエディタのパラメータで調整可能
```

### 2.4 2層ライブラリとの関係

各エンジンは2層ライブラリをゲームロジックと同じApp Areaに同居させる。リンカの未使用コード除去（--gc-sections）で必要最小限のサイズにする。

```
エンジンが使う2層ライブラリの目安：
  シューティング：約1〜2KB（drawOverwrite・フレーム制御・衝突判定）
  ダンジョン：約450B（fillRect・drawBitmap・フレーム制御のみ）
  RPG：約2〜3KB（描画全般・ターン制バトル）
```

---

## 3. 参照ゲーム選定基準と理由

### 3.1 選定基準

| 優先度 | 基準 | 内容 |
|--------|------|------|
| 必須 | ライセンス | MIT・BSD-3のみ採用。GPLは検証目的のみ |
| 必須 | ソースへのアクセス | GitHubから直接ソースを取得・解析できること |
| 高 | エンジン設計の参照価値 | パラメータと課題を発見するための素材として価値があること |
| 高 | ジャンルの代表性 | 各ジャンルで最もシンプルかつ代表的なタイトル |
| 中 | Arduboyコミュニティでの実績 | 実際にプレイされ評価されていること |

**重要：ゲームを移植することが目的ではない。エンジン設計に必要なパラメータと課題を発見するための素材として選ぶ。**

### 3.2 採用タイトルと理由

| ジャンル | タイトル | ライセンス | バイナリ | 選定理由 |
|---------|---------|----------|---------|---------|
| 固定画面シューティング | picovaders | MIT | 17.1KB（実測） | 最小規模・シンプルな構造・ソース全解析済み・getPixel課題の発見素材 |
| 横スクロールアクション | Shadow Runner | MIT | 推定20KB台 | drawErase課題・スクロール背景・差分描画設計の検証素材 |
| 縦スクロールシューティング | galaxion | MIT相当 | 未測定 | 曲線降下AI・波型難易度上昇の参照 |
| 横スクロールシューティング | evade | MIT | 未測定 | HPベースダメージ・チャージ制・ボス設計の参照 |
| 擬似3Dダンジョン | Catacombs of the Damned | MIT | 32.5KB（実測） | レイキャスト・BSP生成・全パラメータ解析済み・エンジン化の原型 |
| RPG・アドベンチャー | Arduventure | MIT（コードのみ） | 推定28KB前後 | ターン制・装備・ショップ設計の参照（ソース非公開・FAQから補完） |

### 3.3 採用から外したタイトルと理由

| タイトル | 理由 |
|---------|------|
| MicroCity | GPL-3・シミュレーションはエンジン化の優先度低 |
| LodeRunner | パズル要素が強くアクションエンジンとは別物 |
| Mystic Balloon | Shadow Runnerと類似ジャンル・追加情報少 |
| Arduventure（コード） | ソース非公開・ゲーム設計はFAQから把握可能 |

---

## 4. シューティングエンジン設計方針

### 4.1 スクロール方向の統一

縦スクロールと横スクロールでエンジンを分けない。パラメータで切り替える。

```json
"scroll": {
  "direction": "vertical" | "horizontal" | "fixed",
  "speed": 2
}
```

地形ありの横スクロール（魔界村系）はタイルマップレンダラーが必要になるが、これはアドベンチャーエンジンのコンポーネントと共有できる。

### 4.2 ゲームパラメータ（調査結果より）

**調査対象：picovaders（固定）/ galaxion（縦）/ evade（横）**

全タイトル共通：60FPS

```json
"player": {
  "move_speed": 1.0,
  "bullet_speed": 2.0,
  "max_bullets": 1,
  "reload_frames": 15,
  "lives": 3,
  "bullet_types": [
    {"damage": 1, "speed": 2.0}
  ]
},

"difficulty": {
  "type": "speed" | "wave" | "iteration",
  "speed_scale_per_kill": 0.01,
  "iteration_hp_bonus": 500,
  "iteration_bullet_speed_bonus": 0.1,
  "iteration_bullet_speed_max": 1.5
},

"enemies": [
  {
    "id": "normal",
    "hp": 1,
    "move_pattern": "straight" | "curve" | "random",
    "fire_chance": 0.001,
    "score": 10
  },
  {
    "id": "boss",
    "hp": 1000,
    "move_pattern": "horizontal",
    "fire_interval": 50
  }
]
```

### 4.3 難易度上昇の3パターン（調査結果）

| パターン | 代表タイトル | 仕組み |
|---------|------------|--------|
| 速度型 | picovaders | 敵が減るほど移動速度上昇・残り1体が最速 |
| 波型 | galaxion | 波ごとに行動パターン変化・攻撃インターバル変動 |
| 反復型 | evade | currentIteration加算でHP・弾速が上昇・無限難易度 |

### 4.4 MODパッケージ構成

```
shooting_mod.zip/
  mod.json          名前・作者・エンジンバージョン要件
  scenario.json     全パラメータ定義
  sprites/
    player.png      自機（16×16px）
    enemy_*.png     敵スプライト
    boss_*.png      ボススプライト
    bullet_*.png    弾スプライト
  stages/
    stage_01.json   敵出現タイミングスクリプト
    stage_02.json
  sounds/
    bgm.mus
    se_shot.snd
    se_explosion.snd
```

---

## 5. ダンジョンエンジン設計方針

### 5.1 設計コンセプト

Catacombs of the Damnedのリスペクト新作としてのカラーダンジョンゲームエンジン。オリジナルのゲームロジックをそのまま再現するのではなく、**フロアごとの難易度テーブルなどオリジナルの弱点を強化した設計**にする。

### 5.2 オリジナルCatacombsとの対比

| 項目 | オリジナル | CA-01版 |
|------|----------|---------|
| 表示 | モノクロ1bit | カラーRGB565 |
| テクスチャ | なし（単色壁） | 16×16pxカラーテクスチャ |
| FPS | 30 | 60 |
| マップサイズ | 32×24セル | 16×16〜32×24セル（パラメータ） |
| マップ生成 | BSP | BSP or Eller's Algorithm（選択可） |
| 同時敵数 | 24体 | 16〜24体（パラメータ） |
| 難易度上昇 | フロアが進んでも変化なし（弱点） | フロアごとの出現テーブルで制御 |

### 5.3 ゲームパラメータ（Catacombs解析結果より）

**プレイヤーパラメータ**

```json
"player": {
  "max_hp": 100,
  "max_mana": 100,
  "mana_fire_cost": 20,
  "mana_recharge_rate": 1,
  "attack_strength": 10,
  "reload_frames": 8,
  "potion_strength": 25,
  "collision_size": 48,
  "interact_distance": 60,
  "move_speed": 4,
  "turn_speed": 3
}
```

**敵アーキタイプ（4種）**

| パラメータ | Skeleton | Mage | Bat | Spider |
|-----------|---------|------|-----|--------|
| HP | 50 | 30 | 20 | 10 |
| 移動速度 | 4 | 5 | 7 | 7 |
| 攻撃力 | 20 | 20 | 10 | 5 |
| 攻撃硬直 | 3f | 3f | 2f | 1f |
| スタン時間 | 2f | 2f | 0f | 0f |
| 遠距離攻撃 | ❌ | ✅ | ❌ | ❌ |

**マップ生成パラメータ**

```json
"map": {
  "width": 32,
  "height": 24,
  "algorithm": "bsp" | "eller",
  "min_room_size": 3,
  "max_room_size": 8,
  "demolish_wall_chance": 20
},
"spawns": {
  "torches": {"count": 64, "min_spacing": 3},
  "monsters": {"count": 24, "min_spacing": 3},
  "items": {"count": 8, "min_spacing": 3, "clearance": 6}
}
```

**フロアごとの出現テーブル（オリジナルにない強化点）**

```json
"floors": [
  {
    "floor": 1,
    "monster_table": [
      {"enemy": "spider", "weight": 50},
      {"enemy": "bat", "weight": 50},
      {"enemy": "skeleton", "weight": 0},
      {"enemy": "mage", "weight": 0}
    ]
  },
  {
    "floor": 4,
    "monster_table": [
      {"enemy": "spider", "weight": 20},
      {"enemy": "bat", "weight": 20},
      {"enemy": "skeleton", "weight": 50},
      {"enemy": "mage", "weight": 10}
    ]
  },
  {
    "floor": 7,
    "monster_table": [
      {"enemy": "spider", "weight": 10},
      {"enemy": "bat", "weight": 10},
      {"enemy": "skeleton", "weight": 40},
      {"enemy": "mage", "weight": 40}
    ]
  }
],
"total_floors": 10
```

### 5.4 レンダリング設計

```
毎フレームの処理：
  1. 全列（128〜240列）についてレイを飛ばす（CPU演算）
  2. 各列の壁スライスをTFTに直接縦転送（バッファ不要）
  3. スプライト（モンスター・アイテム）をTFTに直接描画

性能試算（240列・壁高さ平均40px）：
  CPU演算：約2.0ms
  壁描画：約8.8ms
  スプライト：約1.5ms
  合計：約12.8ms → 78FPS
```

### 5.5 MODパッケージ構成

```
dungeon_mod.zip/
  mod.json
  scenario.json     全パラメータ・フロア出現テーブル
  textures/
    wall_*.png      壁テクスチャ（16×16px・RGB565）
    floor_*.png     床テクスチャ
    ceiling_*.png   天井テクスチャ
  sprites/
    skeleton.png    敵スプライト（16×16px）
    mage.png
    bat.png
    spider.png
    items/*.png     アイテムスプライト
  sounds/
    bgm.mus
    se_*.snd
```

MOD容量目安：テクスチャ32KB＋スプライト8KB＋効果音2KB＋scenario.bin 1KB ≒ **約43KB/MOD**

15MBのMOD領域に約**348本**収まる。

---

## 6. RPG・アドベンチャーエンジン設計方針

### 6.1 ターン制バトルのパラメータ

**調査結果：Arduventure（ソース非公開・FAQから補完）**

```json
"player": {
  "initial_hp": 30,
  "initial_mp": 20,
  "initial_attack": 8,
  "initial_defense": 3,
  "hp_per_level": 15,
  "mp_per_level": 5,
  "attack_per_level": 3,
  "defense_per_level": 2,
  "max_level": 15
},

"battle": {
  "type": "turn_based",
  "damage_formula": "attack - defense/2",
  "critical_chance": 0.10,
  "critical_multiplier": 1.5,
  "escape_chance": 0.70,
  "encounter_rate": 0.15
}
```

### 6.2 マップエディタの必要性

RPGエンジンのMODエディタには**マップエディタ**が必須。シューティングのシナリオエディタより大幅に複雑。ダンジョンエンジンのBSP自動生成とは異なり、手動でタイルを配置する作業が必要。

---

## 7. 統合エディタ（RC Studio）設計方針

### 7.1 エンジン選択でUIが切り替わる統合エディタ

```
RC Dungeon Studio（統合エディタ）
    ↓ エンジン選択
  [ダンジョンRPG] → ダンジョンエディタUI
  [シューティング] → ステージエディタUI
  [アドベンチャー] → マップエディタUI
  [ノベル]         → スクリプトエディタUI
    ↓ 共通
  [アセット管理]   画像・音声の変換・管理
  [MODパッケージ]  mod.zip生成・SPI Flash転送
```

### 7.2 バランスチェック機能

エディタが自動で警告を出す仕組み。

```
⚠️ Floor 1: Mageのweightが0より大きい（推奨：初期フロアは弱い敵のみ）
⚠️ Player HP 100 / 最大ダメージ/秒 = Skeleton×3が囲んだ場合約1.8秒でゲームオーバー
✅ Mana: 5発撃って約3.3秒で全回復（バランス良好）
```

### 7.3 MODローダーの役割

```
mod.zip を受け取る
    ↓
mod.json でエンジンバージョン互換性確認
    ↓
画像変換：PNG → RGB565バイナリ
効果音変換：WAV → YMF825形式
シナリオコンパイル：scenario.json → scenario.bin
    ↓
SPI Flashのどのアドレスに書くか決定
    ↓
write_program.pyで転送
    ↓
カートリッジのMOD選択メニューに反映
```

---

## 8. 開発ロードマップ

### 全体の優先順位

```
ステップ1（CAR-01完成前に着手可能）：
  ├ PC版Platformレイヤー（SDL）実装
  ├ シューティングエンジン Scene 1〜4（PC版）
  ├ ダンジョンエンジン Scene 1〜3（PC版）
  └ 2層ライブラリ基盤実装

ステップ2（CAR-01完成後）：
  ├ 全Scene実機確認
  ├ write_program.pyとの連携確認
  └ picovaders・Shadow RunnerのMOD互換性検証

ステップ3：
  ├ シナリオエディタ・MODローダー原型
  ├ アドベンチャーエンジン
  └ 統合エディタRC Studio

ステップ4：
  ├ コミュニティ向けドキュメント
  ├ SDK整備
  └ ライセンス整理（MITで全配布）
```

### 実機なしで先行できる作業

CAR-01の完成を待たずに、PC上でネイティルビルドできるPlatformレイヤーを先に作ることで：

- エンジンのゲームロジック全体
- レンダリングアルゴリズム（SDL出力で視覚確認）
- シナリオエディタ・MODローダー
- バランスチェッカー

これらがCAR-01完成前に完成できる。CAR-01完成時点でPlatform差し替えだけで動く状態を目指す。

---

## 9. 未決事項

| # | 項目 | 内容 |
|---|------|------|
| 1 | シューティングエンジン詳細API設計 | 新規チャットで議論 |
| 2 | ダンジョンエンジン詳細API設計 | 新規チャットで議論 |
| 3 | アドベンチャーエンジン設計 | ステップ3以降 |
| 4 | MODパッケージフォーマット確定 | シナリオエディタ設計と同時 |
| 5 | エンジンバージョン管理方針 | 後方互換性の保証範囲 |
| 6 | Arduventureソース非公開問題 | 実機プレイで補完・または別RPGタイトルの調査 |

---

*本書は2026年6月1日時点の設計方針をまとめたものである。各エンジンの詳細設計は別チャットで議論・確定する。*
