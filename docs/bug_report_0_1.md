---
project: Panel_y
doc_type: バグレポート
target: Proto #0.1 / #0.2
created: "2026-03-15"
status: 修正済み（Proto #0.2b で対応）
---

# バグレポート — ホバー値・スパイクライン 全チャンネル非連動

## 問題点

1. **ホバー値が合わせたチャンネルのみ表示される**
   - マウスを合わせたパネルの値しか表示されない
   - 他チャンネルの同時刻の値が確認できない

2. **スパイクラインが合わせたパネルのみに表示される**
   - 縦線が1パネル内にしか描画されない
   - 全パネル横断の垂直ラインにならない

---

## 原因

### `make_subplots` の軸参照構造の問題

`make_subplots(shared_xaxes=True)` を使うと、各トレースは **異なる X 軸オブジェクト** を参照する。

```python
# 実際の参照状態（python -c で確認）
trace=ch1, xaxis=x,  yaxis=y
trace=ch2, xaxis=x2, yaxis=y2
xaxis2.matches: None   ← matches リンクもなし
```

`hoversubplots="axis"` は「**同一 xaxis オブジェクトを参照するトレース間**」でのみ機能する。
`x` と `x2` は別オブジェクトのため、ホバーが連動しない。

### スパイクラインの制約

Plotly のスパイクライン（`showspikes=True`）は **サブプロット境界を越えられない**。
`xref` が `x` のスパイクはそのサブプロット内の描画領域にしか表示されない。

---

## 対策・修正ファイル

**修正ファイル**: `proto_0_2/app.py`（→ 実質 #0.2b として再実装）

### 対策1: ホバー値 — 手動レイアウトで全トレースを `xaxis="x"` に統一

`make_subplots` をやめ、`go.Figure()` に直接 Y 軸ドメインを割り当てる。
全トレースが `xaxis="x"` を参照するため、`hovermode="x unified"` が全チャンネルに効く。

```python
# Before（make_subplots）
fig.add_trace(go.Scatter(...), row=1, col=1)  # → xaxis="x"
fig.add_trace(go.Scatter(...), row=2, col=1)  # → xaxis="x2" ← 問題

# After（手動レイアウト）
fig.add_trace(go.Scatter(x=..., y=..., xaxis="x", yaxis="y"))   # ch1
fig.add_trace(go.Scatter(x=..., y=..., xaxis="x", yaxis="y2"))  # ch2
# → 全トレースが xaxis="x" を参照 → hover が全ch連動
```

### 対策2: スパイクライン — Dash Callback で `yref="paper"` シェイプを動的追加

`hoverData` を Callback で受け取り、図全体（`yref="paper"`）に縦線シェイプを追加。

```python
from dash import Patch

@app.callback(
    Output("waveform", "figure"),
    Input("waveform", "hoverData"),
    prevent_initial_call=True,
)
def update_spike(hoverData):
    patched = Patch()
    if hoverData is None:
        patched["layout"]["shapes"] = []
    else:
        x_val = hoverData["points"][0]["x"]
        patched["layout"]["shapes"] = [{
            "type": "line",
            "x0": x_val, "x1": x_val,
            "y0": 0, "y1": 1,
            "xref": "x",
            "yref": "paper",   # ← 図全体の高さにまたがる縦線
            "line": {"color": "rgba(150,150,150,0.8)", "width": 1},
        }]
    return patched
```
