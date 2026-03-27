---
project: Panel_y
doc_type: 設計書
target: Proto #0.4
created: "2026-03-24"
updated: "2026-03-24"
status: 完成・動作確認済み
---

# Proto #0.4 設計書

---

## 1. 目的

Proto #0.3 のテキストエリア方式に代わる、**ドロップダウンUI方式**での行グルーピングを検証する。
#0.3 と比較して使い勝手を評価し、#1 への移植方式を決定する。

---

## 2. 機能一覧

| # | 機能 | #0.3 との差分 |
|---|------|--------------|
| 1 | 1行複数チャンネル重ね表示 | #0.3 から継続 |
| 2 | **ドロップダウンUI方式でレイアウト定義** | **NEW** — 行ごとにマルチセレクトドロップダウン |
| 3 | **行の動的追加・削除** | **NEW** — 「+ 行追加」ボタン / 行ごとの「×」削除ボタン |
| 4 | **Pattern-Matching Callbacks** | **NEW** — 動的に増減するコンポーネントの一括ハンドリング |
| 5 | チャンネル色分け（8色パレット） | #0.3 から継続 |
| 6 | 時間軸同期（shared_xaxes） | #0.1 から継続 |
| 7 | ホバー（x unified） | #0.1 ベース |
| 8 | スクロールズーム | #0.1 から継続 |

### #0.3 との比較

| 観点 | #0.3 テキストエリア | #0.4 ドロップダウン |
|------|-------------------|-------------------|
| チャンネル選択 | 手入力（タイプミスリスク） | 選択式（ミスなし） |
| 行の追加・削除 | テキスト編集 | ボタン操作 |
| 操作の直感性 | プログラマ向き | 一般ユーザー向き |
| 柔軟性 | 高い（自由記述） | 中程度（選択肢から選ぶ） |
| #1 移植の親和性 | 低い（既存UIと異質） | 高い（ドロップダウンは #1 と同じ操作体系） |

---

## 3. 使用方法

### 起動

```bash
cd proto_0_4
source .venv/bin/activate
python app.py
# → http://localhost:8050
```

### サンプルデータ生成

```bash
python generate_sample.py
```

サンプルデータは #0.3 と同一仕様（モータ制御ステップ応答、6チャンネル、10kHz、100ms）。

### 操作方法

| 操作 | 動作 |
|------|------|
| 行のドロップダウン | チャンネルをマルチセレクト（同一行に重ねて表示） |
| 「+ 行追加」ボタン | 空の行を末尾に追加 |
| 「×」ボタン | その行を削除（最後の1行は削除不可） |
| 「更新」ボタン | ドロップダウンの選択内容を波形に反映 |
| ドラッグ | 時間軸をパン |
| マウスホイール | 時間軸をズーム |
| ダブルクリック | ズームリセット |

---

## 4. 実装設計

### ファイル構成

```
proto_0_4/
├── app.py                 # Dashアプリ本体（294行）
├── generate_sample.py     # サンプル波形生成スクリプト（#0.3と共通）
├── sample_data/
│   └── sample_waveform.parquet
└── .venv/
```

### UIレイアウト

```
┌──────────────────────────────────────────────────┐
│ Panel_y — Proto #0.4                              │
│ ドロップダウンで行ごとにチャンネルを選択 / 重ね表示 │
├──────────────────────────────────────────────────┤
│ [+ 行追加]  [更新]                                │
│                                                    │
│ 行1  [ id_ref ▼ | id ▼        ]  [×]             │
│ 行2  [ iq_ref ▼ | iq ▼        ]  [×]             │
│ 行3  [ voltage_u ▼             ]  [×]             │
│ 行4  [ voltage_v ▼             ]  [×]             │
├──────────────────────────────────────────────────┤
│  行1: id_ref (青) / id (オレンジ) ── 重ね表示     │
│  ────────────────────────────────────────────── │
│  行2: iq_ref (緑) / iq (赤) ── 重ね表示          │
│  ────────────────────────────────────────────── │
│  行3: voltage_u ── 単独表示                       │
│  ────────────────────────────────────────────── │
│  行4: voltage_v ── 単独表示                       │
└──────────────────────────────────────────────────┘
```

### コンポーネント構成

```
app.layout
├── dcc.Store(id="row-count")           ← 行インデックスのカウンター（単調増加）
├── html.H2 — タイトル
├── html.Div — 行定義エリア
│   ├── [+ 行追加] [更新] ボタン
│   └── html.Div(id="rows-container")   ← 行UIの親コンテナ
│       ├── make_row_div(0, ["id_ref", "id"])
│       ├── make_row_div(1, ["iq_ref", "iq"])
│       ├── make_row_div(2, ["voltage_u"])
│       └── make_row_div(3, ["voltage_v"])
└── html.Div(id="graph-container")
    └── dcc.Graph(id="waveform")
```

### 行UIの構造（`make_row_div`）

各行は以下の3要素で構成される:

```
html.Div (flex row)
├── html.Span("行N")                                    ← ラベル
├── dcc.Dropdown(id={"type":"row-dropdown","index":N})   ← マルチセレクト
└── html.Button("×", id={"type":"row-delete","index":N}) ← 削除ボタン
```

- `id` は Pattern-Matching ID（dict形式）を使用
- `index` は `row-count` Store から採番（単調増加、削除しても再利用しない）

### Callback フロー

```
[行操作]                           [波形更新]
    │                                   │
    ├─ 「+ 行追加」クリック              「更新」クリック
    │       │                                │
    │       ▼                                ▼
    │  Callback 1: manage_rows          Callback 2: update_graph
    │    Input:  add-row-btn.n_clicks     Input:  update-btn.n_clicks
    │    Input:  {"type":"row-delete",    State:  {"type":"row-dropdown",
    │             "index":ALL}.n_clicks            "index":ALL}.value
    │    State:  rows-container.children     │
    │    State:  row-count.data              ├── 空の value をフィルタ
    │       │                                ├── build_figure(row_groups)
    │       ├── 追加: make_row_div(count)    └── Output: graph-container
    │       │   → children に append
    │       │   → row-count + 1
    │       │
    │       └── 削除: triggered_id の
    │           index を持つ子要素を除外
    │           → 行が0にならない制約
    │
    ├─ 「×」クリック
    │       │
    │       └── (同上 manage_rows)
```

### 主要実装ポイント

**Pattern-Matching Callbacks（`ALL`）**

```python
Input({"type": "row-delete", "index": ALL}, "n_clicks")
State({"type": "row-dropdown", "index": ALL}, "value")
```

- `ALL` は同じ `type` を持つ全コンポーネントを一括で受け取る
- 動的に増減する行に対応できるDashの標準機能

**行インデックスの単調増加**

```python
dcc.Store(id="row-count", data=len(DEFAULT_ROWS))
```

- 行追加時に `row-count` をインクリメントし、新規行の `index` に使用
- 削除しても `index` を再利用しない → ID の一意性を保証

**削除時の子要素フィルタリング**

```python
row_div_children = child["props"]["children"]
dropdown = row_div_children[1]
dropdown_id = dropdown["props"]["id"]
if dropdown_id["index"] != delete_index:
    updated.append(child)
```

- `rows-container` の `children`（dict形式）から、削除対象の `index` を持つ要素を除外
- 最後の1行は削除不可（空の行リストになることを防止）

**`build_figure` は #0.3 と共通**

`parse_layout` → `build_figure` の2段構成から、ドロップダウンの `value` リスト → `build_figure` の1段構成に簡略化。`build_figure` 自体は #0.3 とほぼ同一。

---

## 5. 動作確認結果（2026-03-24）

| # | 確認項目 | 結果 |
|---|---------|------|
| 1 | デフォルト4行のドロップダウン表示 | OK |
| 2 | ドロップダウンからチャンネル選択 → 更新 → 重ね表示 | OK |
| 3 | 行追加ボタン → 空行追加 | OK |
| 4 | ×ボタン → 行削除 | OK |
| 5 | 最後の1行の削除防止 | OK |
| 6 | 2チャンネル重ね表示 | OK |
| 7 | 3チャンネル以上の重ね表示 | OK |
| 8 | 時間軸ズーム・パン | OK |

---

## 6. 検証結果と次のステップ

ドロップダウン方式は #0.3 のテキスト方式と比較して:

- **利点**: チャンネル名のタイプミスがない、操作が直感的、#1 の既存UI体系と一貫性がある
- **課題**: 行数が多い場合にUIが縦に長くなる（折りたたみ等の工夫が必要になる可能性）

→ **#1 への移植はドロップダウンUI方式を採用する**（Phase 1-1）。

---

## 7. 関連ドキュメント

- [Proto #0.3 設計書](./proto_0_3_設計書.md) — テキストエリア方式（比較対象）
- [Proto #1 設計書](./proto_1_設計書.md) — 実用ビューア（移植先）
- [波形表示機能ロードマップ](./wave_viewer_roadmap.md) — 今後の開発計画
