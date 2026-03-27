---
project: Panel_y
doc_type: バグレポート
target: Proto #0.2
created: "2026-03-15"
status: 未修正
---

# バグレポート — Proto #0.2 3件の不具合

---

## Bug #1: スパイクラインが表示されない

### 問題点
マウスをグラフに合わせても、全パネル縦断スパイクラインが表示されない。

### 原因
スパイクラインは Callback 4（`update_graphs`）が `hover-x-store` の更新を受けて描画する。
しかし `hover-x-store` は Callback 1（`store_hover_x`）が `hoverData` を受け取って初めて更新される。

`hoverData` が発火しない原因が Bug #2 にある。
→ **Bug #2 の連鎖障害**

### 対策
Bug #2 を修正すれば連動して解決する。

---

## Bug #2: ホバー値が表示されない

### 問題点
マウスをグラフに合わせても、左列の値（`val-{ch}`）が `---` のまま更新されない。

### 原因
トレース生成時に `hoverinfo="skip"` を指定している。

```python
# app.py: make_channel_fig() 内
fig.add_trace(go.Scatter(
    ...
    hoverinfo="skip",   ← これが原因
))
```

Plotly の `hoverinfo` の挙動:

| 値 | ツールチップ | hoverData イベント |
|----|-------------|-------------------|
| `"skip"` | 非表示 | **発火しない** ← 問題 |
| `"none"` | 非表示 | 発火する |
| `"x+y"` 等 | 表示 | 発火する |

`hoverinfo="skip"` はホバーイベント自体を無効化するため、
Dash の `hoverData` コールバックプロパティが一切更新されない。
→ Callback 1 が動かず、値表示もスパイクラインも動作しない。

### 対策
```python
# 修正前
hoverinfo="skip"

# 修正後（ツールチップは非表示にしつつイベントは発火させる）
hoverinfo="none"
```

---

## Bug #3: ズーム・パン・リセットボタンが消えた

### 問題点
#0.1 にあった Plotly 標準ツールバー（ズーム・パン・軸リセット・PNG保存）が表示されない。

### 原因
`dcc.Graph` の `config` に `displayModeBar: False` を指定している。

```python
# app.py: channel_row() 内
dcc.Graph(
    config={"scrollZoom": True, "displayModeBar": False},  ← ツールバー完全非表示
)
```

`displayModeBar` の挙動:

| 値 | 動作 |
|----|------|
| `True` | 常に表示 |
| `False` | 完全に非表示 ← 問題 |
| `"hover"` | マウスオーバー時のみ表示（#0.1 と同等） |

`displayModeBar: False` にしたのは行レイアウトで各グラフが小さいためバーが邪魔になると判断したため。
しかしズーム・リセット操作が完全に失われた。

### 対策
```python
# 修正後（ホバー時のみ表示、かつ不要なボタンを除外）
config={
    "scrollZoom": True,
    "displayModeBar": "hover",
    "modeBarButtonsToRemove": ["select2d", "lasso2d", "autoScale2d"],
}
```

---

## 修正ファイル

- `proto_0_2/app.py`
  - `make_channel_fig()` 内: `hoverinfo="skip"` → `hoverinfo="none"`
  - `channel_row()` 内: `displayModeBar: False` → `displayModeBar: "hover"`
