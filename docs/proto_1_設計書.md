---
project: Panel_y
doc_type: 設計書
target: Proto #1
created: "2026-03-15"
updated: "2026-03-16"
status: 完成
---

# Proto #1 設計書 — 実用ビューア

---

## 1. 目標

日常の波形確認作業で **実際に使えるレベル** にする。
#0.2 の確立済みパターン（行レイアウト・ホバー値・スパイクライン・ズーム同期）を継承し、
ファイル選択・チャンネル管理・差分計算の3つの柱を追加する。

---

## 2. 機能一覧

| # | 機能 | #0.2 との差分 | 状態 |
|---|------|--------------|------|
| 1 | **ファイル選択 UI**（パス入力 + 補完 + 読み込みボタン） | **NEW** | [x] |
| 2 | **データ前処理ユーティリティ**（CSV/MAT → Parquet） | **NEW**（`utils/` + `multiconverter.py`） | [x] |
| 3 | **チャンネル表示/非表示**（ドロップダウン型マルチセレクト） | **NEW** | [x] |
| 4 | **カーソル差分計算**（計測モードトグル + クリック交互方式） | **NEW** | [x] |
| ~~5~~ | ~~マウスアウト時のホバー値リセット~~ | 不採用（最後の値を残す方が実用的） | — |
| 6 | 行レイアウト `[ch名/値] [波形グラフ]` | #0.2 から継続 | [x] |
| 7 | 全チャンネルのホバー値同時表示 | #0.2 から継続 | [x] |
| 8 | 全グラフ縦断スパイクライン | #0.2 から継続 | [x] |
| 9 | グラフ間ズーム・パン同期 | #0.2 から継続 | [x] |
| 10 | スクロールズーム・ツールバー | #0.2 から継続 | [x] |

**完了条件**: 実際のモータ制御ログ（Parquet）を読み込んで、上記10機能がすべて動作する

---

## 3. 使用方法

### 起動

```bash
cd proto_1
source .venv/bin/activate
python app.py
# → http://localhost:8050
```

### 前処理（事前変換）

```bash
# CSV → Parquet
python -m utils.converter input.csv data/output.parquet

# MAT → Parquet（時間カラム・単位を明示）
python -m utils.converter data.mat data/output.parquet --time-col ECUTIME --time-unit ms

# フォルダ内 .mat を一括変換
python multiconverter.py --input ./input --output ./output
```

### 操作方法

| 操作 | 動作 |
|------|------|
| ファイルパス入力 | パスを入力すると候補（📁 / 📄）が表示される。候補クリックでパス更新 |
| 「読み込み」ボタン | 入力済みの .parquet ファイルをサーバー側で読み込み |
| ヘッダーのドロップダウン | チャンネルをマルチセレクトで選択 → 波形行を動的生成 |
| グラフ上でマウスホバー | 左列に全チャンネルの値を同時表示 + 全グラフに縦断スパイクライン |
| マウスアウト | 最後のホバー値がそのまま残る（#0.2 と同じ） |
| ドラッグ | 時間軸をパン。全グラフが連動 |
| マウスホイール | 時間軸をズーム。全グラフが連動 |
| ダブルクリック | ズームリセット（全体表示に戻る） |
| 「📏 計測」ボタン | 計測モードの ON/OFF をトグル（ON 時ボタンが青くハイライト） |
| **クリック**（計測モード ON 時） | カーソル A/B を交互に設置（A → B → A更新…） → 差分表示 |

---

## 4. ファイル構成

```
proto_1/
├── app.py                # Dash アプリ本体
├── multiconverter.py     # フォルダ内 .mat 一括変換 CLI
├── requirements.txt      # 依存パッケージ
├── assets/
│   └── style.css         # Dash カスタム CSS
├── data/                 # 変換済み Parquet を配置（.gitignore 除外）
└── utils/                # 前処理ユーティリティ
    ├── __init__.py
    ├── converter.py       # 変換エントリポイント（公開API + CLI）
    ├── reader_csv.py      # CSV 読み込み
    ├── reader_mat.py      # MATLAB MAT 読み込み（v7.2以前 + v7.3 HDF5）
    ├── time_normalizer.py # 時間カラム検出・単位正規化
    ├── column_filter.py   # 非数値カラム除外
    └── resampler.py       # 等間隔チェック・線形補間リサンプリング
```

---

## 5. 全体アーキテクチャ

```
┌──────────────────────────────────────────────────────────────────────────┐
│ app.layout                                                                │
│                                                                          │
│  ┌─ ヘッダーバー ──────────────────────────────────────────────────────┐ │
│  │ [PanelY]  [____パス入力____][読み込み]  [▼ ch選択▾]  [📏 計測]     │ │
│  │           id="file-path-input"          id="ch-dropdown"            │ │
│  │           id="file-suggestions"(候補)   id="measure-toggle-btn"    │ │
│  │           id="load-btn"                 id="load-status"(右端)     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─ 差分表示パネル（カーソル設置時に表示）─────────────────────────────┐ │
│  │ Cursor A: 0.02340 s   Cursor B: 0.04560 s                          │ │
│  │ Δt = 22.200 ms                                                     │ │
│  │ ch1: A=141.4  B=−141.4  Δ=−282.8                                  │ │
│  │ id="delta-panel"                                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─ waveform-container ───────────────────────────────────────────────┐ │
│  │ [Channel / Value] [Waveform]                  ← ヘッダー行         │ │
│  │ ┌──────────────┬──────────────────────────────────────────────┐    │ │
│  │ │ ch_name      │ dcc.Graph(id={type:"wf-graph",channel:ch})  │    │ │
│  │ │ ホバー値     │    [波形グラフ + カーソルA/B 縦線]           │    │ │
│  │ │ id={wf-val}  │                                              │    │ │
│  │ ├──────────────┼──────────────────────────────────────────────┤    │ │
│  │ │ ch_name      │ dcc.Graph(id={type:"wf-graph",channel:ch})  │    │ │
│  │ │ ホバー値     │    [波形グラフ + カーソルA/B 縦線]           │    │ │
│  │ │ id={wf-val}  │                                              │    │ │
│  │ └──────────────┴──────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  dcc.Store(id="hover-x-store")                                           │
│  dcc.Store(id="xaxis-range-store")                                       │
│  dcc.Store(id="data-store")         ← ファイルパス + チャンネル名リスト   │
│  dcc.Store(id="cursor-a-store")     ← カーソル A の X 座標               │
│  dcc.Store(id="cursor-b-store")     ← カーソル B の X 座標               │
│  dcc.Store(id="measure-mode")       ← 計測モード ON/OFF (bool)           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Callback フロー

```
[ユーザー操作]
      │
      ├─ ファイルパス入力
      │       │
      │       ▼
      │  suggest_files
      │    Input:  file-path-input.value（入力中のパス文字列）
      │    Output: file-suggestions.children（候補ボタンリスト）
      │            file-suggestions.style（表示/非表示）
      │    処理:   入力パスの親ディレクトリを iterdir() で走査し
      │            📁 サブディレクトリ + 📄 .parquet をボタンとして返す（最大30件）
      │       │
      │       ├──▶ on_suggestion_click（候補ボタンクリック時）
      │       │      Input:  {type:"suggestion", path:ALL}.n_clicks
      │       │      Output: file-path-input.value（パスを更新）
      │       │
      │       ▼
      │  load_file（「読み込み」ボタンクリック時）
      │    Input:  load-btn.n_clicks
      │    State:  file-path-input.value
      │    Output: data-store.data（ファイルパス + チャンネル情報）
      │            ch-dropdown.options / value（チャンネル一覧 / 未選択[]）
      │            load-status.children（✓ ファイル名, ch数, 点数, Hz）
      │            cursor-a-store / cursor-b-store（リセット）
      │    処理:   サーバー側で pd.read_parquet(path) → グローバル df に保持
      │
      ├─ チャンネルドロップダウン変更
      │       │
      │       ▼
      │  update_waveform_rows
      │    Input:  ch-dropdown.value
      │    State:  data-store.data
      │    Output: waveform-container.children（波形行の動的生成）
      │    処理:   選択された ch の波形行を生成・表示。未選択 ch は行自体を除去
      │
      ├─ 計測モードトグル
      │       │
      │       ▼
      │  toggle_measure
      │    Input:  measure-toggle-btn.n_clicks
      │    Output: measure-mode.data（ON/OFF）
      │            measure-toggle-btn.style（ON=青, OFF=グレー）
      │            cursor-a-store / cursor-b-store（OFF時リセット）
      │
      ├─ グラフ上でホバー
      │       │
      │       ▼
      │  store_hover_x                                ← #0.2 から継続
      │    Input:  {type:"wf-graph", channel:ALL}.hoverData
      │    Output: hover-x-store.data
      │       │
      │       ├──▶ update_values                      ← #0.2 から継続
      │       │      Input:  hover-x-store.data
      │       │      State:  {type:"wf-val", channel:ALL}.id
      │       │      Output: {type:"wf-val", channel:ALL}.children
      │       │      処理:   x_val が None → "---" / 値あり → df 逆引き
      │       │
      │       └──▶ update_graphs                      ← #0.2 から継続 + カーソル描画追加
      │              triggered="hover-x-store"
      │              → スパイクライン + カーソル A/B 縦線を shapes で描画
      │
      ├─ ズーム・パン操作
      │       │
      │       ▼
      │  store_xaxis_range                            ← #0.2 から継続
      │    Input:  {type:"wf-graph", channel:ALL}.relayoutData
      │    Output: xaxis-range-store.data
      │       │
      │       ▼
      │  update_graphs                                ← #0.2 から継続
      │    triggered="xaxis-range-store"
      │    → 全グラフの xaxis.range を Patch で同期
      │
      ├─ クリック（計測モード ON 時）
      │       │
      │       ▼
      │  set_cursor
      │    Input:  {type:"wf-graph", channel:ALL}.clickData
      │    State:  measure-mode.data, cursor-a-store, cursor-b-store
      │    Output: cursor-a-store.data / cursor-b-store.data
      │    処理:   クリック交互方式（A → B → A更新(Bクリア) → B → ...）
      │       │
      │       ▼
      │  update_delta_panel
      │    Input:  cursor-a-store.data, cursor-b-store.data
      │    State:  ch-dropdown.value
      │    Output: delta-panel.children / style
      │    処理:   カーソル位置表示 + 両カーソルが揃ったら Δt + 各ch の ΔV を算出
```

---

## 7. 機能別 実装設計

### 7-1. ファイル選択 UI（テキスト入力 + 候補ボタン + 読み込みボタン方式）

**方式**: `dcc.Input` でパスを自由入力。入力中にサーバー側でディレクトリを走査し、
候補を `html.Button` のリストとして動的表示する。候補クリックでパスを更新し、
明示的な「読み込み」ボタンで Parquet を読み込む。
ファイルの中身はサーバー側で直接読み込むため、**サイズ制限なし**（100MB超対応）。

> `dcc.Upload`（ブラウザ側 base64 読み込み）は大容量ファイルでブラウザが重くなるため不採用。
> `dcc.Dropdown(searchable)` はディレクトリ階層の掘り下げに不向きなため不採用。

```python
dcc.Input(
    id="file-path-input",
    type="text",
    value=str(DATA_DIR) + "/",
    placeholder="Parquet ファイルパスを入力...",
    debounce=False,
)
html.Div(id="file-suggestions")   # 候補ボタンのコンテナ
html.Button("読み込み", id="load-btn")
```

**補完の動作**:

```
ユーザー入力: /Users/sugusokothx/da
                                    │
                          suggest_files
                          Input: file-path-input.value
                                    │
                                    ▼
                          parent.iterdir() でフィルタ（最大30件）
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │ [📁 data/              ]  ← Button │
                    │ [📁 dashboard/         ]  ← Button │
                    └───────────────────────────────┘

ボタンクリック: data/  → on_suggestion_click → file-path-input.value を更新
                    ┌───────────────────────────────┐
                    │ [📄 motor_log_001.parquet]     │
                    │ [📄 motor_log_002.parquet]     │
                    │ [📁 2026-03/             ]     │
                    └───────────────────────────────┘

ボタンクリック: motor_log_001.parquet → パス確定
「読み込み」ボタン → load_file → pd.read_parquet(path)
```

**load_file の処理フロー**:
1. `load-btn.n_clicks` で発火、`file-path-input.value` からパスを取得
2. バリデーション: 存在チェック / ディレクトリ判定 / `.parquet` 拡張子チェック
3. `pd.read_parquet(path)` でサーバー側読み込み → グローバル変数 `df` に保持
4. `ch-dropdown.options` にチャンネル一覧をセット、`value=[]`（未選択状態）
5. `load-status` にファイル名・ch数・サンプル数・サンプリング周波数を表示
6. カーソル A/B をリセット

**初期ディレクトリ**: `app.py` と同階層の `data/`（`DATA_DIR`）を起動時の初期値として表示

---

### 7-2. チャンネル表示/非表示（ドロップダウン型マルチセレクト）

**方式**: ヘッダーバーに `dcc.Dropdown(multi=True)` を配置。
データ読み込み直後は **波形を表示しない**。ユーザーがドロップダウンから表示 ch を選択する。

```python
dcc.Dropdown(
    id="ch-dropdown",
    options=[],          # ファイル読み込み後にチャンネル一覧を設定
    value=[],            # 初期は未選択（波形なし）
    multi=True,
    searchable=True,     # 検索機能 ON（30ch 超でも絞り込み可能）
    placeholder="チャンネルを選択...",
)
```

**動作**:
- ファイル読み込み → `options` にチャンネル一覧をセット、`value=[]`（未選択）
- ユーザーが ch を選択 → 選択された ch の波形行を動的に生成して表示
- ch を解除 → 該当行を除去
- ドロップダウンの検索ボックスでチャンネル名をインクリメンタル検索可能

---

### 7-3. カーソル差分計算（Δt / ΔV / ΔA）

**方式**: 計測モードトグル + クリックでカーソル設置

**UI コンポーネント**:
```python
# 計測モードトグルボタン（クリックで ON/OFF、ON 時は青くハイライト）
html.Button("📏 計測", id="measure-toggle-btn", n_clicks=0)

# 計測モード状態
dcc.Store(id="measure-mode", data=False)

# 差分表示パネル
html.Div(id="delta-panel", style={"display": "none"})
```

**カーソルの表示**:
- カーソル A: 青い縦線（`line.color = "#4fc3f7"`）
- カーソル B: 赤い縦線（`line.color = "#ef5350"`）
- `yref="paper"` で全グラフ高さを縦断（スパイクラインと同方式）
- Callback 4（`update_graphs`）内で shapes に追加

**差分パネルの表示内容**:
```
Cursor A: 0.02340 s    Cursor B: 0.04560 s
Δt = 22.200 ms
voltage_u: A=141.4  B=−141.4  Δ=−282.8
current_u: A=8.66   B=−8.66   Δ=−17.32
```

**クリック交互方式**:
- Plotly の `clickData` には修飾キー情報が含まれないため、Shift+クリック方式は不採用
- **クリック交互方式** を採用（1回目=A, 2回目=B, 3回目=A更新(Bクリア)…）
- `cursor-a-store` と `cursor-b-store` の状態で次にどちらを設置するか決定

```python
# クリック交互ロジック
if cursor_b is not None:
    # 両方埋まっている → A を更新（サイクル）
    return x_val, None
elif cursor_a is not None:
    # A のみ → B を設置
    return cursor_a, x_val
else:
    # 両方 None → A を設置
    return x_val, None
```

---

### ~~7-4. マウスアウト時のホバー値リセット~~ → 不採用

マウスアウト時は最後のホバー値をそのまま残す（#0.2 と同じ動作）。
値を確認してからメモを取る等の用途で、値が消えると不便なため。
`dcc.Interval` や `hover-timestamp-store` は不要。

---

## 8. #0.2 から継続する実装パターン

| パターン | 内容 |
|---------|------|
| `hoverinfo="none"` | ツールチップ非表示・hoverData イベントは発火 |
| `hovermode="x"` | X 軸方向のホバー検出 |
| `yref="paper"` シェイプ | 全グラフ高さを縦断する縦線 |
| `Patch()` 差分更新 | figure 全体を返さず該当箇所のみ更新 |
| `ctx.triggered_id` 分岐 | 1 Callback で複数イベントを処理 |
| ダークテーマ | `template="plotly_dark"`, `paper/plot_bgcolor` |
| ツールバー設定 | `displayModeBar="hover"`, `scrollZoom=True` |

---

## 9. Store 一覧

| Store ID | 型 | 用途 |
|----------|-----|------|
| `hover-x-store` | float \| None | 現在のホバー X 座標 |
| `xaxis-range-store` | [float, float] \| None | 同期中の X 軸レンジ |
| `data-store` | dict | 読み込み済みファイルパス + チャンネル名リスト |
| `cursor-a-store` | float \| None | カーソル A の X 座標 |
| `cursor-b-store` | float \| None | カーソル B の X 座標 |
| `measure-mode` | bool | 計測モード ON/OFF |

---

## 10. 参照ドキュメント

- [企画書](./企画書.md) — 全体ロードマップ・技術スタック
- [Parquet データ仕様](./parquet_data_spec.md) — 入力データの制約
- [前処理ユーティリティ設計書](./preprocessor_design.md) — utils/ の設計・API
- [Proto #0.2 設計書](./proto_0_2_設計書.md) — 継承する実装パターン
- [bug_report_0_1.md](./bug_report_0_1.md) — スパイクライン問題（make_subplots 不採用の理由）
- [bug_report_0_2.md](./bug_report_0_2.md) — hoverinfo/displayModeBar 設定ミス
