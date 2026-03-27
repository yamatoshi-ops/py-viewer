---
project: Panel_y
doc_type: 設計書
target: Proto #2.0
created: "2026-03-24"
updated: "2026-03-24"
status: 完成
---

# Proto #2.0 設計書 — 実用ビューア + ドロップダウン行UI

---

## 1. 目標

Proto #1（実用ビューア）と Proto #0.4（ドロップダウン行UI）を統合する。
行ごとにドロップダウンで複数チャンネルを選択し重ね表示できるようにしつつ、
#1 の全機能（ファイル選択・計測モード・ホバー同期・ズーム同期）を維持する。

加えて、タッチパッドでの操作性改善（スケールロック）とY軸範囲指定を追加する。

---

## 2. 機能一覧

| # | 機能 | #1 との差分 | 状態 |
|---|------|------------|------|
| 1 | **行ごとドロップダウン（複数チャンネル重ね表示）** | **NEW**（#0.4 から移植） | [x] |
| 2 | **行の追加・削除** | **NEW**（#0.4 から移植） | [x] |
| 3 | **スケールロック（scrollZoom ON/OFF）** | **NEW** | [x] |
| 4 | **Y軸範囲の数値指定（行ごと）** | **NEW** | [x] |
| 5 | **Y軸目盛りSI接頭辞表記** | **NEW** | [x] |
| 6 | **更新キーボードショートカット（Ctrl+Enter）** | **NEW** | [x] |
| 7 | ファイル選択 UI（パス入力 + 補完 + 読み込み） | #1 から継続 | [x] |
| 8 | カーソル差分計算（計測モード） | #1 から継続 | [x] |
| 9 | 全チャンネルのホバー値同時表示 | #1 から継続 | [x] |
| 10 | 全グラフ縦断スパイクライン | #1 から継続 | [x] |
| 11 | グラフ間ズーム・パン同期 | #1 から継続 | [x] |
| ~~12~~ | ~~ヘッダーの単一チャンネルドロップダウン~~ | **廃止**（行ドロップダウンに置換） | — |

**完了条件**: 実際のモータ制御ログ（Parquet）を読み込んで、上記11機能がすべて動作する

---

## 3. アーキテクチャ選定

### make_subplots 方式 vs 独立Graph方式

Proto #2.0 の開発過程で `make_subplots` 方式を試行したが、以下の理由で不採用とした。

| 項目 | make_subplots（試行→不採用） | 独立Graph（採用） |
|------|---------------------------|------------------|
| ホバー連動 | `hoversubplots="axis"` が機能せず（各トレースが異なるX軸を参照） | `hover-x-store` 経由で全行連動 |
| スパイクライン | サブプロット境界を越えられない | `yref="paper"` で各Graph全高を縦断 |
| ズーム同期 | `shared_xaxes=True` で自動（ただし上記制約あり） | `xaxis-range-store` 経由で全行同期 |
| 実装複雑度 | 低（単一Figure） | 中（Store + pattern-matching callback） |

**結論**: ホバー連動・スパイクライン縦断が必須要件のため、#1 と同じ独立Graph方式を採用。
この判断は bug_report_0_1.md で記録済みの知見と一致する。

---

## 4. 使用方法

### 起動

```bash
cd panel-y
source .venv/bin/activate   # 共通venv（panel-y直下）
python proto_2_0/app.py
# → http://localhost:8050
```

### 操作方法

| 操作 | 動作 |
|------|------|
| ファイルパス入力 | パスを入力すると候補（📁 / 📄）が表示される。候補クリックでパス更新 |
| 「読み込み」ボタン | 入力済みの .parquet ファイルをサーバー側で読み込み。初期行を自動生成 |
| 行ドロップダウン | 行ごとにチャンネルをマルチセレクト（重ね表示対応） |
| 「+ 行追加」ボタン | 空の行を末尾に追加 |
| 「×」ボタン | 該当行を削除 |
| Y: min ~ max | 行ごとにY軸範囲を数値で指定（空欄ならautorange） |
| 「更新 (Ctrl+Enter)」ボタン | ドロップダウン・Y軸設定を反映して波形を再描画 |
| 「🔒 スケール固定」ボタン | スクロールズームの ON/OFF。デフォルトOFF（ページスクロール優先） |
| スクロールズーム解除時 | X軸方向のみスクロールズーム可能（Y軸は固定） |
| グラフ上でマウスホバー | 左列に全行のチャンネル値を同時表示 + 全グラフに縦断スパイクライン |
| ドラッグ | 時間軸をパン。全グラフが連動 |
| ダブルクリック | ズームリセット（全体表示に戻る） |
| 「📏 計測」ボタン | 計測モードの ON/OFF をトグル |
| クリック（計測モード ON 時） | カーソル A/B を交互に設置 → 差分表示 |
| Ctrl+Enter | 「更新」ボタンと同等（キーボードショートカット） |

---

## 5. ファイル構成

```
proto_2_0/
├── app.py                # Dash アプリ本体
├── multiconverter.py     # フォルダ内 .mat 一括変換 CLI（#1 から継続）
├── requirements.txt      # 依存パッケージ
├── assets/
│   └── style.css         # Dash カスタム CSS
├── data/                 # 変換済み Parquet を配置（.gitignore 除外）
└── utils/                # 前処理ユーティリティ（#1 から継続）
    ├── __init__.py
    ├── converter.py
    ├── reader_csv.py
    ├── reader_mat.py
    ├── time_normalizer.py
    ├── column_filter.py
    └── resampler.py
```

---

## 6. 全体アーキテクチャ

```
┌──────────────────────────────────────────────────────────────────────────┐
│ app.layout                                                                │
│                                                                          │
│  ┌─ ヘッダーバー ──────────────────────────────────────────────────────┐ │
│  │ [PanelY] [____パス入力____][読み込み] [🔒スケール固定] [📏計測]    │ │
│  │          id="file-path-input"         id="scroll-zoom-btn"         │ │
│  │          id="file-suggestions"        id="measure-toggle-btn"      │ │
│  │          id="load-btn"                id="load-status"(右端)       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌─ 行定義エリア ───────────────────────────────────────────────────┐   │
│  │ [+ 行追加] [更新 (Ctrl+Enter)]                                    │   │
│  │ 行1: [▼ ch選択 (multi)] [Y: min ~ max] [×]                       │   │
│  │ 行2: [▼ ch選択 (multi)] [Y: min ~ max] [×]                       │   │
│  │ ...                                                                │   │
│  │ id="rows-container"                                                │   │
│  │   {type:"row-dropdown", index:N}                                   │   │
│  │   {type:"ymin-input", index:N} / {type:"ymax-input", index:N}     │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─ 差分表示パネル ─────────────────────────────────────────────────┐   │
│  │ Cursor A: 0.02340 s   Cursor B: 0.04560 s                        │   │
│  │ Δt = 22.200 ms                                                   │   │
│  │ ch1: A=141.4  B=−141.4  Δ=−282.8                                │   │
│  │ id="delta-panel"                                                  │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─ waveform-container ───────────────────────────────────────────────┐ │
│  │ [Channel / Value] [Waveform]                  ← ヘッダー行         │ │
│  │ ┌──────────────┬──────────────────────────────────────────────┐    │ │
│  │ │ ch1/ch2      │ dcc.Graph(id={type:"wf-graph",row:0})       │    │ │
│  │ │ ホバー値     │   [重ね表示波形 + カーソルA/B + スパイク]    │    │ │
│  │ │ id={wf-val}  │                                              │    │ │
│  │ ├──────────────┼──────────────────────────────────────────────┤    │ │
│  │ │ ch3          │ dcc.Graph(id={type:"wf-graph",row:1})       │    │ │
│  │ │ ホバー値     │   [波形 + カーソルA/B + スパイクライン]      │    │ │
│  │ │ id={wf-val}  │                                              │    │ │
│  │ └──────────────┴──────────────────────────────────────────────┘    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  dcc.Store(id="row-count")          ← 行カウンタ（追加用の連番管理）     │
│  dcc.Store(id="row-groups-store")   ← 現在の行グルーピング [[ch,...],..] │
│  dcc.Store(id="hover-x-store")      ← ホバー X 座標                     │
│  dcc.Store(id="xaxis-range-store")  ← 同期中の X 軸レンジ               │
│  dcc.Store(id="data-store")         ← ファイルパス + チャンネル名リスト  │
│  dcc.Store(id="cursor-a-store")     ← カーソル A の X 座標              │
│  dcc.Store(id="cursor-b-store")     ← カーソル B の X 座標              │
│  dcc.Store(id="measure-mode")       ← 計測モード ON/OFF                 │
│  dcc.Store(id="scroll-zoom-store")  ← スクロールズーム ON/OFF           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Callback フロー

```
[ユーザー操作]
      │
      ├─ ファイルパス入力
      │       ▼
      │  suggest_files → on_suggestion_click → load_file
      │  （#1 と同一。詳細は Proto #1 設計書 §7-1 参照）
      │  ※ load_file は追加で行UIの初期生成も行う
      │     → rows-container に ch数分のドロップダウン行を配置
      │
      ├─ 行ドロップダウン操作 + 「更新」ボタン / Ctrl+Enter
      │       ▼
      │  update_waveform_rows
      │    Input:  update-btn.n_clicks, scroll-zoom-store.data
      │    State:  {type:"row-dropdown"}.value(ALL),
      │            {type:"ymin-input"}.value(ALL),
      │            {type:"ymax-input"}.value(ALL)
      │    Output: waveform-container.children（波形行の動的生成）
      │            row-groups-store.data（行グルーピング情報を保存）
      │    処理:   空でない行のみ抽出 → 各行の waveform_row() を生成
      │            scrollZoom / Y軸範囲 / SI表記をFigureに反映
      │
      ├─ 行の追加・削除
      │       ▼
      │  manage_rows
      │    Input:  add-row-btn.n_clicks,
      │            {type:"row-delete"}.n_clicks(ALL)
      │    Output: rows-container.children, row-count.data
      │
      ├─ スケールロックトグル
      │       ▼
      │  toggle_scroll_zoom
      │    Input:  scroll-zoom-btn.n_clicks
      │    Output: scroll-zoom-store.data（ON/OFF）
      │            scroll-zoom-btn のラベル・スタイル
      │       │
      │       └──▶ update_waveform_rows（scroll-zoom-store がInputのため自動発火）
      │            → scrollZoom=True/False + yaxis.fixedrange で再描画
      │
      ├─ グラフ上でホバー
      │       ▼
      │  store_hover_x → update_values + update_graphs
      │  （#1 と同一パターン。row-groups-store を参照して行単位で値表示）
      │
      ├─ ズーム・パン操作
      │       ▼
      │  store_xaxis_range → update_graphs
      │  （#1 と同一パターン。常に全行に range を適用）
      │
      └─ クリック（計測モード ON 時）
              ▼
         set_cursor → update_graphs + update_delta_panel
         （#1 と同一パターン。row-groups-store から全チャンネルの差分を算出）
```

---

## 8. 機能別 実装設計

### 8-1. 行ごとドロップダウン（#0.4 から移植）

**方式**: 行定義エリアに `dcc.Dropdown(multi=True)` を行数分配置。
「更新」ボタンクリック（または Ctrl+Enter）で波形を再描画する。

```python
# 行1つ分のUI
make_dropdown_row(row_index, selected)
  → html.Span("行N")
  → dcc.Dropdown(id={type:"row-dropdown", index:N}, multi=True)
  → dcc.Input(id={type:"ymin-input", index:N}, type="number")  # Y軸min
  → dcc.Input(id={type:"ymax-input", index:N}, type="number")  # Y軸max
  → html.Button("×", id={type:"row-delete", index:N})
```

**重ね表示**: 1行のドロップダウンに複数チャンネルを選択すると、同一 Figure 上に重ねて描画。
色はパレット `TRACE_COLORS` から順番に自動割り当て。凡例は2ch以上で表示。

### 8-2. スケールロック

**課題**: タッチパッドではスワイプがスクロールズームに奪われ、ページスクロールができない。

**方式**:
- デフォルト: `scrollZoom=False`（ページスクロール優先）
- 解除時: `scrollZoom=True` + `yaxis.fixedrange=True`（X軸方向のみズーム）

**DashのscrollZoom制約**: `config.scrollZoom` は `boolean` のみ受け付ける（`"x"` 等の文字列は不可）。
そのため、Y軸方向のズーム制限は `yaxis.fixedrange=True` で実現する。

### 8-3. Y軸範囲の数値指定

各行のドロップダウン横に `dcc.Input(type="number")` で min / max を配置。
「更新」ボタンクリック時に `make_row_fig()` の `ymin` / `ymax` 引数に渡す。

- 両方指定: `yaxis.range=[ymin, ymax]`, `autorange=False`
- 片方のみ: `range=[ymin, None]` または `[None, ymax]`
- 両方空欄: autorange（デフォルト）

### 8-4. Y軸目盛りSI接頭辞表記

**課題**: Y軸の目盛り桁数が行ごとに異なると、グラフ左端の位置がずれる。

**対策**: `yaxis.tickformat=".3s"` + `yaxis.automargin=False` + `margin.l=60px`（固定）

- SI接頭辞（k, M, m, μ 等）で桁数が一定になるため、全行のグラフ左端が揃う
- パワエレ系計測ツール（オシロ等）の表記と親和性が高い

### 8-5. キーボードショートカット（Ctrl+Enter）

clientside callback で `document.addEventListener('keydown')` を登録。
`Ctrl+Enter`（Mac: `Cmd+Enter`）で `update-btn` を `click()` する。

```javascript
document.addEventListener('keydown', function(e) {
    if ((e.ctrlKey || e.metaKey) && e.key === 'Enter') {
        e.preventDefault();
        document.getElementById('update-btn').click();
    }
});
```

---

## 9. #1 から継続する実装パターン

| パターン | 内容 |
|---------|------|
| 独立Graph方式 | 各行は個別の `go.Figure()`。`make_subplots` は使わない |
| `hover-x-store` 同期 | いずれかのGraphの hoverData → Store → 全行に値表示 + スパイクライン |
| `xaxis-range-store` 同期 | いずれかのGraphの relayoutData → Store → 全行に range 適用 |
| `Patch()` 差分更新 | figure 全体を返さず shapes / xaxis.range のみ更新 |
| `yref="paper"` シェイプ | 全グラフ高さを縦断する縦線（スパイク・カーソル） |
| カーソル交互方式 | A → B → A更新(Bクリア) → B → ... |
| ダークテーマ | `template="plotly_dark"`, `paper/plot_bgcolor` |
| サーバーサイド df | 大容量DataFrameはサーバーメモリで保持 |

---

## 10. Store 一覧

| Store ID | 型 | 用途 |
|----------|-----|------|
| `row-count` | int | 行カウンタ（追加時の連番管理） |
| `row-groups-store` | list[list[str]] | 現在の行グルーピング |
| `hover-x-store` | float \| None | 現在のホバー X 座標 |
| `xaxis-range-store` | [float, float] \| None | 同期中の X 軸レンジ |
| `data-store` | dict | 読み込み済みファイルパス + チャンネル名リスト |
| `cursor-a-store` | float \| None | カーソル A の X 座標 |
| `cursor-b-store` | float \| None | カーソル B の X 座標 |
| `measure-mode` | bool | 計測モード ON/OFF |
| `scroll-zoom-store` | bool | スクロールズーム ON/OFF（デフォルト OFF） |

---

## 11. #1 → #2.0 の変更差分まとめ

| 領域 | #1 | #2.0 |
|------|-----|------|
| チャンネル選択 | ヘッダーの単一ドロップダウン（1ch=1行） | 行ごとドロップダウン（複数ch重ね表示） |
| 行管理 | チャンネル選択で自動生成 | 追加・削除ボタン + 更新ボタン |
| pattern-matching key | `{type:"wf-graph", channel:ch}` | `{type:"wf-graph", row:N}` |
| scrollZoom | 常時ON | デフォルトOFF、トグルで ON（X軸のみ） |
| Y軸範囲 | autorange固定 | 行ごとに min/max 数値指定可能 |
| Y軸目盛り | デフォルト表記 | SI接頭辞表記（`.3s`） |
| venv | 各proto個別 | panel-y直下に共通venv |

---

## 12. 参照ドキュメント

- [Proto #1 設計書](./proto_1_設計書.md) — ベースとなるアーキテクチャ・Callbackフロー
- [Proto #0.4 設計書](./proto_0_4_設計書.md) — ドロップダウン行UIの元実装
- [波形表示機能ロードマップ](./wave_viewer_roadmap.md) — Phase 1-5 計画
- [bug_report_0_1.md](./bug_report_0_1.md) — make_subplots 不採用の理由
- [bug_report_2_0.md](./bug_report_2_0.md) — #2.0 開発中の問題と対処
