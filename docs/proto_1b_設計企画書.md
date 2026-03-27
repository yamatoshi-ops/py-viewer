---
project: Panel_y
doc_type: 設計企画書
target: Proto #1b
created: "2026-03-16"
updated: "2026-03-16"
status: 企画中
---

# Proto #1b 設計企画書 — Panel + Bokeh 版 実用ビューア

---

## 1. 目的

Proto #1（Dash + Plotly）と**同一の仕様**を、Panel（HoloViz）+ Bokeh で実装する。
技術スタックの比較評価を目的とした**並行プロトタイプ**であり、
Proto #2 以降のベースをどちらにするか判断するための材料にする。

---

## 2. 実現する機能（#1 と同一）

| # | 機能 | 状態 |
|---|------|------|
| 1 | ファイル選択 UI（パス入力 + 補完 + 読み込みボタン） | [ ] |
| 2 | データ前処理ユーティリティ（CSV/MAT → Parquet） | [x] ← `utils/` をそのまま流用 |
| 3 | チャンネル表示/非表示（マルチセレクト） | [ ] |
| 4 | カーソル差分計算（計測モードトグル + クリック交互方式） | [ ] |
| 5 | 行レイアウト `[ch名/値] [波形グラフ]` | [ ] |
| 6 | 全チャンネルのホバー値同時表示 | [ ] |
| 7 | 全グラフ縦断スパイクライン | [ ] |
| 8 | グラフ間ズーム・パン同期 | [ ] |
| 9 | スクロールズーム・ツールバー | [ ] |

---

## 3. 技術スタック

| 項目 | Proto #1（Dash版） | Proto #1b（Panel版） |
|------|-------------------|---------------------|
| Webフレームワーク | Dash | **Panel**（HoloViz） |
| グラフライブラリ | Plotly.py | **Bokeh** |
| データハンドリング | Pandas + NumPy | Pandas + NumPy（同一） |
| 前処理ユーティリティ | `utils/` | `utils/`（**共有**） |
| 標準データ形式 | Parquet | Parquet（同一） |
| サーバー | Flask（Dash内蔵） | Tornado（Bokeh/Panel内蔵） |

### なぜ Bokeh か

Panel は複数のプロットバックエンドに対応するが、本プロジェクトでは **Bokeh をネイティブに使う**。

- **リンクドレンジ**: `x_range` オブジェクトを共有するだけでズーム・パンが同期する（Dash の Store + Callback 方式より簡潔）
- **CrosshairTool / HoverTool**: 組み込みツールでスパイクライン・ホバー値の基本機能が得られる
- **CustomJS**: クライアントサイド JS コールバックにより、サーバー往復なしで高速なインタラクションを実現可能
- **ColumnDataSource**: サーバー・クライアント間のデータ同期モデルが明示的

---

## 4. Dash → Panel 移行の見通し

### 4-1. ズーム・パン同期（容易 → ネイティブ対応）

**Dash #1 の実装**: `relayoutData` → `xaxis-range-store` → `Patch()` で全グラフを更新（サーバー往復）

**Panel/Bokeh の実装**: 全 figure の `x_range` に同一オブジェクトを渡すだけ

```python
from bokeh.plotting import figure

shared_x_range = None

def make_figure(ch, df, is_first=False):
    nonlocal shared_x_range
    if is_first:
        fig = figure(...)
        shared_x_range = fig.x_range
    else:
        fig = figure(..., x_range=shared_x_range)
    fig.line(x='time', y=ch, source=source)
    return fig
```

**評価**: Bokeh のリンクドレンジは**完全にクライアントサイド**で動作する。
Dash 版の Store→Callback チェーンが不要になり、パフォーマンスが向上する可能性が高い。

---

### 4-2. スパイクライン（要工夫 → CrosshairTool + CustomJS）

**Dash #1 の実装**: `hover-x-store` → `update_graphs` で全グラフに `shapes` を Patch 追加（サーバー往復）

**Panel/Bokeh の実装**:

Bokeh の `CrosshairTool` は**単一 figure 内のみ**で動作する。
全グラフ縦断のスパイクラインを実現するには以下の方式が必要:

**方式: 共有 Span + CustomJS**

```python
from bokeh.models import Span, CrosshairTool, CustomJS

# 各 figure に Span（縦線）を追加
spans = []
for fig in figures:
    span = Span(dimension='height', line_color='gray',
                line_alpha=0.7, line_width=1, visible=False)
    fig.add_layout(span)
    spans.append(span)

# 1つの figure の HoverTool から全 Span を更新する CustomJS
code = """
const x = cb_data.geometry.x;
for (const span of spans) {
    span.location = x;
    span.visible = true;
}
"""
callback = CustomJS(args=dict(spans=spans), code=code)
hover = HoverTool(tooltips=None, callback=callback, mode='vline')
figures[0].add_tools(hover)
```

**課題**:
- HoverTool の callback は**トリガー元の figure 上でのみ**発火する
- 全 figure に HoverTool を追加し、それぞれが全 Span を更新する必要がある
- マウスアウト時の Span 非表示には `PointerMove` イベント or `MouseLeave` の CustomJS が必要

**評価**: Dash 版より多少冗長だが、**サーバー往復が発生しない**ため高速。
CustomJS の初期実装が必要だが、一度書けば再利用可能。

---

### 4-3. ホバー値同時表示（要工夫 → CustomJS + Div 更新）

**Dash #1 の実装**: `hover-x-store` → `update_values` Callback（サーバー往復で df 逆引き）

**Panel/Bokeh の実装**:

**方式A: CustomJS（クライアントサイド完結）**

```python
from bokeh.models import Div, CustomJS

# 各チャンネルの値表示 Div
value_divs = {}
for ch in channels:
    value_divs[ch] = Div(text="---", styles={"color": "#7ec8e3", "font-size": "20px"})

# HoverTool の CustomJS で ColumnDataSource から値を逆引き
code = """
const x = cb_data.geometry.x;
const time = source.data['time'];
// 最近傍インデックスを二分探索
let lo = 0, hi = time.length - 1;
while (lo < hi) {
    const mid = (lo + hi) >> 1;
    if (time[mid] < x) lo = mid + 1;
    else hi = mid;
}
const idx = lo;
for (const [ch, div] of Object.entries(divs)) {
    const val = source.data[ch][idx];
    div.text = val.toFixed(4);
}
"""
```

**方式B: param.watch（サーバーサイド、Dash #1 と同等）**

Panel の `param.watch` で Python コールバックを使い、df から値を取得する。
サーバー往復が発生するが、実装はシンプル。

**評価**: 方式A（CustomJS）を推奨。サーバー往復なしで値が即座に更新され、
大容量データでもレスポンス劣化しない。ただし、全データを `ColumnDataSource` に
載せる必要があるため、メモリ使用量が増加する（後述: §6 制約事項）。

---

### 4-4. カーソル差分計算（やや困難 → TapTool + CustomJS + Python連携）

**Dash #1 の実装**: `clickData` → `set_cursor`（交互ロジック）→ `update_delta_panel`（サーバー往復）

**Panel/Bokeh の実装**:

```python
from bokeh.models import TapTool, Span, CustomJS

# カーソル Span（A: 青, B: 赤）を全 figure に追加
cursor_a_spans = [Span(dimension='height', line_color='#4fc3f7',
                       line_width=2, line_dash='dashed', visible=False)
                  for _ in figures]
cursor_b_spans = [Span(dimension='height', line_color='#ef5350',
                       line_width=2, line_dash='dashed', visible=False)
                  for _ in figures]

# クリック交互ロジックを CustomJS で実装
code = """
const x = cb_data.geometries[0].x;
if (state.next === 'a') {
    for (const s of a_spans) { s.location = x; s.visible = true; }
    for (const s of b_spans) { s.visible = false; }
    state.cursor_a = x;
    state.cursor_b = null;
    state.next = 'b';
} else {
    for (const s of b_spans) { s.location = x; s.visible = true; }
    state.cursor_b = x;
    state.next = 'a';
}
// delta-panel の更新（テキスト）
if (state.cursor_a !== null && state.cursor_b !== null) {
    const dt = Math.abs(state.cursor_b - state.cursor_a) * 1000;
    delta_div.text = `Δt = ${dt.toFixed(3)} ms`;
}
"""
```

**課題**:
- 計測モードの ON/OFF トグルは Panel ウィジェット（`pn.widgets.Toggle`）で実装
- トグル OFF 時に TapTool を無効化する必要がある（`CustomJS` で `tool.active` を切り替え）
- 差分パネルの各チャンネル ΔV 計算は `ColumnDataSource` から JS で算出可能だが、コード量が増える
- **代替**: 差分計算のみサーバーサイド（`param.watch`）で実行し、`pn.pane.HTML` を更新する方式もある

**評価**: 基本的なカーソル設置・Δt 表示は CustomJS で実現可能。
各チャンネルの ΔV 計算まで含めると CustomJS のコード量が大きくなるため、
**カーソル位置取得は CustomJS → Δ計算は Python コールバック** のハイブリッド方式が現実的。

---

### 4-5. ファイル選択 UI（容易 → Panel ウィジェット）

**Dash #1 の実装**: `dcc.Input` + 候補ボタンリスト + 読み込みボタン（Callback 3つ）

**Panel/Bokeh の実装**:

```python
import panel as pn

file_input = pn.widgets.TextInput(
    name='ファイルパス',
    value=str(DATA_DIR) + "/",
    placeholder='Parquet ファイルパスを入力...',
)
load_btn = pn.widgets.Button(name='読み込み', button_type='primary')
status = pn.pane.Markdown("", styles={"color": "#8bc34a"})

# 候補リスト
suggestions = pn.Column()

def on_path_change(event):
    # ファイル候補を動的生成（Dash版 list_path_suggestions と同等ロジック）
    ...

def on_load_click(event):
    global df
    df = pd.read_parquet(file_input.value)
    ...

file_input.param.watch(on_path_change, 'value')
load_btn.on_click(on_load_click)
```

**評価**: Dash 版とほぼ同等の実装量。Panel ウィジェットの方がコールバック定義がシンプル
（デコレータ不要、Output/Input/State の明示不要）。

---

### 4-6. チャンネル表示/非表示（容易 → MultiChoice + 動的レイアウト）

**Panel/Bokeh の実装**:

```python
ch_selector = pn.widgets.MultiChoice(
    name='チャンネル選択',
    options=[],
    value=[],
)

waveform_area = pn.Column()

def update_channels(event):
    rows = []
    for i, ch in enumerate(ch_selector.value):
        fig = make_figure(ch, df, is_last=(i == len(ch_selector.value) - 1))
        label = pn.pane.Markdown(f"**{ch}**\\n---", ...)
        rows.append(pn.Row(label, fig))
    waveform_area.objects = rows

ch_selector.param.watch(update_channels, 'value')
```

**評価**: Panel の動的レイアウト更新（`column.objects = [...]`）は
Dash の `children` 返却と同等。MultiChoice ウィジェットは Dash の
`dcc.Dropdown(multi=True)` に相当し、検索機能も組み込み。

---

## 5. 全体アーキテクチャ（Panel版）

```
┌──────────────────────────────────────────────────────────────────────────┐
│ pn.template.FastDarkTheme (or custom)                                    │
│                                                                          │
│  ┌─ header ──────────────────────────────────────────────────────────┐   │
│  │ [PanelY]  [TextInput:パス] [Button:読み込み]                      │   │
│  │           [MultiChoice:ch選択]  [Toggle:📏計測]  [Markdown:status]│   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─ delta_panel (pn.pane.HTML) ──────────────────────────────────────┐   │
│  │ Cursor A: 0.02340 s   Cursor B: 0.04560 s   Δt = 22.200 ms      │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─ waveform_area (pn.Column) ───────────────────────────────────────┐   │
│  │ ┌─ Row ─────────────────────────────────────────────────────────┐ │   │
│  │ │ [ch名]    │ Bokeh figure (x_range=shared)                     │ │   │
│  │ │ [値 Div]  │   line + Span(spike) + Span(cursorA) + Span(cursorB)│ │   │
│  │ ├───────────┼───────────────────────────────────────────────────┤ │   │
│  │ │ [ch名]    │ Bokeh figure (x_range=shared)                     │ │   │
│  │ │ [値 Div]  │   line + Span(spike) + Span(cursorA) + Span(cursorB)│ │   │
│  │ └───────────┴───────────────────────────────────────────────────┘ │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ※ dcc.Store 相当は不要（Bokeh モデルが状態を保持）                      │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 制約事項・リスク

| 項目 | 内容 | 影響度 |
|------|------|--------|
| **ColumnDataSource のメモリ** | Bokeh は全データを JSON でブラウザに転送する。100万点超のデータではブラウザメモリが圧迫される可能性がある | 高 |
| **CustomJS のコード量** | スパイクライン・ホバー値・カーソルの JS コードが合計 100行超になる見込み。デバッグが Python より困難 | 中 |
| **CrosshairTool の制約** | figure 単体でしか動作しない。クロスフィギュアは全て自前実装 | 中 |
| **HoverTool の排他性** | HoverTool の callback は `tooltips=None` でないと干渉する。ツールチップとの共存に注意 | 低 |
| **Panel のダークテーマ** | `FastDarkTheme` テンプレートで概ね対応。Bokeh figure は `theme` 設定が別途必要 | 低 |
| **Dash 版 utils/ との共有** | `utils/` は純 Python でフレームワーク非依存。問題なく共有可能 | — |

### 大容量データへの対策

Dash #1 はサーバー側に DataFrame を保持し、ブラウザには Plotly figure（描画済み点）のみ送信する。
Bokeh の場合は `ColumnDataSource` 経由で**生データ全体**をブラウザに送る。

対策候補:
1. **サーバーサイドダウンサンプリング**: 表示ピクセル数に応じて間引き（LTTB アルゴリズム等）
2. **WebGL レンダリング**: `output_backend="webgl"` で大量ポイントの描画を GPU に委譲
3. **ColumnDataSource のストリーミング**: ズーム範囲のみデータを送信（Panel + Bokeh server で可能だがサーバー往復が発生）

---

## 7. Dash 版 vs Panel 版 比較サマリー

| 観点 | Dash + Plotly (#1) | Panel + Bokeh (#1b) |
|------|--------------------|--------------------|
| **ズーム・パン同期** | Store + Callback（サーバー往復） | **リンクドレンジ（クライアント完結）** |
| **スパイクライン** | Callback + Patch shapes | CustomJS + Span（クライアント完結） |
| **ホバー値表示** | Callback + df逆引き（サーバー往復） | CustomJS + ColumnDataSource（クライアント完結） |
| **カーソル差分** | Callback チェーン | CustomJS + Python ハイブリッド |
| **レスポンス性能** | ホバー・ズーム時にサーバー往復あり | **ほぼ全てクライアント完結で高速** |
| **大容量データ** | サーバー側保持で有利 | ブラウザ転送が必要（ダウンサンプリング要検討） |
| **コードの見通し** | Python のみで完結 | Python + CustomJS（二言語） |
| **実装量（概算）** | app.py: ~650行 | app.py: ~500行 + CustomJS: ~150行 |
| **依存パッケージ** | dash, plotly | panel, bokeh |
| **テンプレート/テーマ** | inline style + assets/CSS | FastDarkTheme + Bokeh Theme |

---

## 8. ファイル構成

```
proto_1b/
├── app.py                # Panel アプリ本体
├── js/                   # CustomJS コード（Python埋め込みでもよいが分離推奨）
│   ├── spike_line.js     # クロスフィギュア スパイクライン
│   ├── hover_values.js   # ホバー値の全チャンネル同時更新
│   └── cursor.js         # カーソル設置・交互ロジック
├── requirements.txt      # panel, bokeh, pandas, numpy, scipy, pyarrow, h5py
├── data/                 # → ../proto_1/data/ へのシンボリックリンク or 共有
└── utils -> ../proto_1/utils  # シンボリックリンクで共有
```

---

## 9. 起動方法

```bash
cd proto_1b
source .venv/bin/activate  # proto_1 と venv を共有 or 別途作成
panel serve app.py --show
# → http://localhost:5006
```

---

## 10. 実装の進め方（提案）

Proto #1 の全機能を一度に実装するのではなく、Bokeh 固有の技術検証を兼ねて段階的に進める。

| Phase | 内容 | 検証ポイント |
|-------|------|-------------|
| **Phase 1** | 固定 Parquet を読み込み、2ch の波形を表示。x_range 共有でズーム同期 | リンクドレンジの動作確認 |
| **Phase 2** | 全グラフ縦断スパイクライン + ホバー値表示（CustomJS） | CustomJS の実装パターン確立 |
| **Phase 3** | ファイル選択 UI + チャンネル動的切り替え | Panel ウィジェット + 動的レイアウト |
| **Phase 4** | 計測モード + カーソル差分パネル | TapTool + CustomJS/Python ハイブリッド |
| **Phase 5** | ダークテーマ統一 + スタイル調整 + 大容量データ検証 | 実データでの性能比較 |

**Phase 2 完了時点**で、Dash 版との操作感の差が判断できるため、
ここで続行/中止の判断ポイントを置くことを推奨する。

---

## 11. 判断基準（#1 vs #1b）

Proto #2 のベースをどちらにするかは、以下の観点で評価する:

| 評価軸 | 重み | 有利な方（予想） |
|--------|------|-----------------|
| ホバー・ズーム時のレスポンス | 高 | #1b（クライアント完結） |
| 大容量データ（50万点超）の扱い | 高 | #1（サーバー側保持） |
| コードの保守性・可読性 | 中 | #1（Python完結） |
| 拡張性（FFT パネル追加等） | 中 | 同等 |
| ダークテーマの仕上がり | 低 | #1（Plotly dark テンプレート完成度） |

---

## 12. 参照ドキュメント

- [Proto #1 設計書](./proto_1_設計書.md) — 同一仕様の Dash 版設計
- [Parquet データ仕様](./parquet_data_spec.md) — 入力データの制約
- [前処理ユーティリティ設計書](./preprocessor_design.md) — 共有する utils/ の設計
- [Panel 公式ドキュメント](https://panel.holoviz.org/)
- [Bokeh 公式ドキュメント](https://docs.bokeh.org/)
