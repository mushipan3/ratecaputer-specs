# CAR-01規格 A+B+C基板 パネリング発注目論見書

## 完全確定版 — Claude Code投入用マスタードキュメント

---

## 1. パネル概要

### プロジェクト概要

ラテカピュータ（CAR-01規格）を構成するA・B・C基板3種を、JLCPCBの100×100mm枠内に収まる1枚のパネル（面付け）として一括発注する。3基板すべての縦幅を28mmに統一することで、パネル全域をV-CUT直線のみで分割できる完全格子構造を実現する。

### パネル物理仕様

| 項目 | 仕様 |
|------|------|
| パネル外形サイズ | 84mm × 84mm |
| 基板厚 | 0.8mm（JLCPCB 2層 FR-4） |
| 表面仕上げ | HASL-LF（鉛フリーハンダレベラー） |
| 分割方式 | **V-CUTのみ**（ミシン目・マウスバイトなし） |
| V-CUTライン（縦） | 5本（各列境界：x=10mm, x=20mm, x=38mm, x=56mm, x=66mm） |
| V-CUTライン（横） | 2本（各行境界：y=28mm, y=56mm） |
| 外枠捨て基板 | なし（JLCPCBのクランプ余白は84mmパネル外形で確保） |
| PCBA有無 | あり（A基板・B基板のみPCBA。C基板は生基板のみ） |
| 発注枚数（パネル単位） | **5パネル** |

---

## 2. パネルレイアウト詳細

### 基板配置マップ

```
←─────────────────── 84mm ───────────────────→
┌──────────┬──────────┬──────────────────────┐  ↑
│  B基板#1 │  B基板#4 │       B基板#7        │  │
│ 18×28mm  │ 18×28mm  │      18×28mm         │  28mm
│  PCBA    │  PCBA    │       PCBA           │  │
├──────────┼──────────┼──────────────────────┤  ↓
│  B基板#2 │  B基板#5 │  B基板#8  │  A基板#1 │  ↑
│ 18×28mm  │ 18×28mm  │ 18×28mm  │ 10×28mm  │  28mm
│  PCBA    │  PCBA    │  PCBA    │  PCBA    │  │
├──────────┼──────────┼──────────┴──────────┤  ↓
│  B基板#3 │  B基板#6 │  A基板#2 │  A基板#5 │  ↑
│ 18×28mm  │ 18×28mm  │ 10×28mm  │ 10×28mm  │  │
│  PCBA    │  PCBA    │  PCBA    │  PCBA    │  28mm
│          │          ├──────────┼──────────┤  │
│          │          │  A基板#3 │  C基板#1 │  │
│          │          │ 10×28mm  │ 10×28mm  │  │
│          │          │  PCBA    │ 生基板のみ│  ↓
└──────────┴──────────┴──────────┴──────────┘

※ 上記は概念図。正確な配置はKiCadパネルファイルを参照。
```

### 正確な列構成（左→右）

| 列 | 横幅 | 内容 | 段数 |
|----|------|------|------|
| 1列目 | 18mm | B基板 × 3段 | #1・#2・#3 |
| 2列目 | 18mm | B基板 × 3段 | #4・#5・#6 |
| 3列目 | 18mm | B基板 × 2段 + A基板 × 1段 | #7・#8・A#1 |
| 4列目 | 10mm | A基板 × 3段 | #2・#3・#4 |（注記参照）
| 5列目 | 10mm | A基板 × 3段 | #5・#6・#7 |
| 6列目 | 10mm | C基板 × 3段 | #1・#2・#3 |

> **注記：** 3列目のA基板1枚と4・5列目のA基板6枚で合計7枚/パネル。

### 寸法検算

| 方向 | 計算 | 合計 |
|------|------|------|
| 横幅 | 18+18+18+10+10+10 | **84mm** ✅ |
| 縦幅 | 28×3 | **84mm** ✅ |
| JLCPCB 100mm枠内余白 | 上下左右各8mm | ✅ PCBA爪干渉なし |

---

## 3. 取得枚数と配分計画

### 5パネル発注時の取得枚数

| 基板 | 1パネル内取数 | 5パネル合計 | 用途 |
|------|------------|-----------|------|
| **B基板**（FM音源） | 8枚 | **40枚** | YMF825-EZ在庫上限に対応 |
| **A基板**（電源管理） | 7枚 | **35枚** | CAR-01共通インフラ用 |
| **C基板**（ストレージ） | 3枚 | **15枚** | UIAPduino用スポット在庫 |

---

## 4. 実装方針サマリー

| 基板 | 自動実装（PCBA）対象 | 手はんだ対象 |
|------|-------------------|------------|
| A基板 | 全点（ATtiny202・IRLML6402・Buck DCDC・MT3608・受動部品） | なし |
| B基板 | PAM8302A・受動部品全点 | YMF825-EZ・12.288MHz水晶・3.5mmジャック |
| C基板 | **C1・C2のみPCBA（Basic）** | U1（W25Q128JVSIM）・U2（MB85RS4MTYPF） |

---

## 5. PCBA対象部品の実装区分

### A基板 PCBA（パネル内で自動実装）

Extended Parts段取り費：4種 × $3 = $12/パネル

| RefDes | 部品 | LCSC番号 | 区分 |
|--------|------|---------|------|
| U1 | ATtiny202 (SOIC-8) | C2052970 | Extended |
| U2 | IRLML6402 P-ch MOSFET (SOT-23) | C5148470 | Extended |
| U3 | Buck DCDC 3.3V (SOT-23-5) | 要確認 | Extended |
| U4 | MT3608 昇圧DCDC (SOT-23-6) | C84817 | Extended |
| R1〜R3・C1〜C7・L1・L2・D1・SW1 | 受動部品各種 | 各Basic | Basic/Extended |

### C基板 PCBA（パネル内で自動実装）

Basic Parts のみのため Extended 段取り費なし（追加費用$0）

| RefDes | 部品 | LCSC番号 | 区分 |
|--------|------|---------|------|
| C1 | 積層セラミックコンデンサ 0.1µF / 50V / X7R (1608) | C14663 | Basic |
| C2 | 積層セラミックコンデンサ 0.1µF / 50V / X7R (1608) | C14663 | Basic |

| RefDes | 部品 | LCSC番号 | 区分 |
|--------|------|---------|------|
| U1 | PAM8302AAYCR (MSOP-8) | C11117 | Basic |
| C1〜C7・R1〜R5 | 受動部品各種 | 各Basic | Basic |

---

## 6. Claude Code用 パネルデータ生成指示書

```text
# Claude Code Instruction: KiCad Panel (Panelize) PCB Data Generation
# for CAR-01 A+B+C Board Panel (Final Confirmed Version)

## 1. Context and Constraint
- Panel Name       : CAR-01 Panel Rev.1
- Panel Size       : 84mm x 84mm (fits within JLCPCB 100x100mm limit)
- PCB Thickness    : 0.8mm, 2-Layer FR-4
- Singulation      : V-CUT ONLY. No mouse bites, no routing slots.
- V-CUT Lines      :
    Vertical (5 lines):
      x = 10mm (between Col1-A10 and Col2-B18)
      x = 28mm (between Col2-B18 and Col3-B18)
      x = 46mm (between Col3-B18 and Col4-A10)
      x = 56mm (between Col4-A10 and Col5-A10)
      x = 66mm (between Col5-A10 and Col6-C10)
    Horizontal (2 lines):
      y = 28mm (between Row1 and Row2)
      y = 56mm (between Row2 and Row3)
- PCBA Fiducial    : Place 3x fiducial marks on panel corners
                     (required for PCBA pick-and-place machine).
- Tooling Holes    : Place 4x M2 tooling holes at panel corners
                     (3.2mm diameter, copper-free keepout).

## 2. Individual Board Placement (origin = panel bottom-left)

### B-Board (18x28mm) — 8 instances per panel
- B1 : x=0,    y=56mm  (Col1, Row3)
- B2 : x=0,    y=28mm  (Col1, Row2)
- B3 : x=0,    y=0     (Col1, Row1)
- B4 : x=18mm, y=56mm  (Col2, Row3)
- B5 : x=18mm, y=28mm  (Col2, Row2)
- B6 : x=18mm, y=0     (Col2, Row1)
- B7 : x=36mm, y=56mm  (Col3, Row3)
- B8 : x=36mm, y=28mm  (Col3, Row2)

### A-Board (10x28mm) — 7 instances per panel
- A1 : x=36mm, y=0     (Col3, Row1)
- A2 : x=46mm, y=56mm  (Col4, Row3)
- A3 : x=46mm, y=28mm  (Col4, Row2)
- A4 : x=46mm, y=0     (Col4, Row1)
- A5 : x=56mm, y=56mm  (Col5, Row3)
- A6 : x=56mm, y=28mm  (Col5, Row2)
- A7 : x=56mm, y=0     (Col5, Row1)

### C-Board (10x28mm) — 3 instances per panel
- C1 : x=66mm, y=56mm  (Col6, Row3)
- C2 : x=66mm, y=28mm  (Col6, Row2)
- C3 : x=66mm, y=0     (Col6, Row1)

## 3. Source Files Required
- A-Board KiCad project : a_board_power_spec generated .kicad_pcb
- B-Board KiCad project : b_board_audio_spec generated .kicad_pcb
- C-Board KiCad project : c_board_storage_spec generated .kicad_pcb

## 4. Output Files Required
- panel.kicad_pcb     : Complete panel PCB layout
- panel_gerber/       : Gerber files (compressed .zip for JLCPCB upload)
  - panel-F.Cu.gbr    : Front copper
  - panel-B.Cu.gbr    : Back copper
  - panel-F.SilkS.gbr : Front silkscreen
  - panel-B.SilkS.gbr : Back silkscreen
  - panel-F.Mask.gbr  : Front solder mask
  - panel-B.Mask.gbr  : Back solder mask
  - panel.drl         : Drill file (Excellon)
  - panel-Edge.Cuts.gbr : Board outline + V-CUT lines
- panel_bom_a.csv     : BOM for A-Board PCBA (JLCPCB format)
- panel_bom_b.csv     : BOM for B-Board PCBA (JLCPCB format)
- panel_cpl_a.csv     : Component Placement List for A-Board
- panel_cpl_b.csv     : Component Placement List for B-Board
  (CPL coordinates must be panel-absolute, not board-relative)

## 5. JLCPCB Upload Checklist
- [ ] Gerber ZIP uploaded
- [ ] Board thickness: 0.8mm confirmed
- [ ] Layer count: 2 confirmed
- [ ] V-CUT selected in panelization options
- [ ] SMT Assembly (PCBA) enabled
- [ ] BOM file (panel_bom_a.csv + panel_bom_b.csv) uploaded
- [ ] CPL file (panel_cpl_a.csv + panel_cpl_b.csv) uploaded
- [ ] C-Board marked as "no assembly" in JLCPCB order

Generate all output files using KiCad MCP server.
Verify V-CUT lines align exactly with board boundaries.
Verify no copper, pads, or components overlap V-CUT lines.
```

---

## 7. 発注手順

### Step 1：各基板のKiCadプロジェクト生成

Claude Codeにて以下の順で実行する。

1. `a_board_power_spec_20260527.md` → A基板 `.kicad_sch` + `.kicad_pcb` 生成
2. `b_board_audio_spec.md` → B基板 `.kicad_sch` + `.kicad_pcb` 生成
3. `c_board_storage_spec_20260527.md` → C基板 `.kicad_sch` + `.kicad_pcb` 生成

### Step 2：パネルデータ生成

本ファイルのSection 6指示書をClaude Codeへ投入し、KiCad MCPサーバ経由でパネルPCBと全出力ファイルを生成する。

### Step 3：JLCPCBへのアップロード

| 提出ファイル | 内容 |
|------------|------|
| `panel_gerber.zip` | ガーバーデータ一式 |
| `panel_bom_a.csv` | A基板PCBA用BOM |
| `panel_bom_b.csv` | B基板PCBA用BOM |
| `panel_cpl_a.csv` | A基板部品配置リスト（パネル絶対座標） |
| `panel_cpl_b.csv` | B基板部品配置リスト（パネル絶対座標） |

### Step 4：JLCPCBオーダー設定

| 設定項目 | 値 |
|---------|---|
| PCB数量 | 5（パネル5枚） |
| 基板厚 | 0.8mm |
| 銅箔層数 | 2層 |
| 表面仕上げ | HASL-LF |
| 基板色 | Green |
| V-CUT | あり（縦5本・横2本） |
| SMT組立 | あり（A基板・B基板のみ。C基板は "no assembly"） |
| 組立側 | Top面 |

---

## 8. コスト試算（参考）

| 項目 | 単価（概算） | 小計 |
|------|------------|------|
| PCB製造（5パネル） | 〜$5/5枚 | 〜$5 |
| A基板 PCBA Extended段取り費（4種） | $3×4=$12/パネル | $60 |
| A基板 PCBA 部品実装費 | 部品数×単価 | 別途 |
| B基板 PCBA 部品実装費（Basicのみ） | 手数料$0 | 部品実費のみ |
| C基板 ICパーツ（手持ち） | U1:891円+U2:2,888円 | 既調達済 |
| C基板 パスコン（C1・C2） | 数十円 | 別途購入 |

> **実際の発注価格はJLCPCBの見積もり画面で確認すること。**

---

*文書バージョン：確定版 2026-05-27 / 本ファイルをそのままClaude Codeへ投入してください*
