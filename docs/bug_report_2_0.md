---
project: Panel_y
doc_type: バグレポート
target: Proto #2.0
created: "2026-03-24"
status: 全件修正済み
---

# バグレポート — Proto #2.0 開発中の問題と対処

---

## 問題1: make_subplots 方式でホバー・スパイクラインが全行連動しない

### 症状

初期実装で `make_subplots(shared_xaxes=True)` + `hoversubplots="axis"` を使用したが、
ホバー値が操作中のサブプロットのみ表示され、スパイクラインもサブプロット境界を越えなかった。

### 原因

bug_report_0_1.md で既知の問題と同一。
`make_subplots` では各トレースが異なる X 軸オブジェクト（`x`, `x2`, ...）を参照するため、
`hoversubplots="axis"` が期待通り機能しない。

### 対処

`make_subplots` を廃止し、Proto #1 と同じ **独立Graph方式**（行ごとに `go.Figure()`）に方針転換。
`hover-x-store` / `xaxis-range-store` を介した Store 同期パターンを採用。

### 教訓

`make_subplots` のホバー連動制限は Plotly のアーキテクチャに起因する根本的な制約。
複数行の波形ビューアでは独立Graph方式が唯一の実用的解法。
過去のバグレポートを参照して方針転換の判断を迅速に行えた。

---

## 問題2: config.scrollZoom に文字列 "x" を渡すとエラー

### 症状

スクロールズームをX軸のみに制限するため `config={"scrollZoom": "x"}` を指定したところ、
Dash がバリデーションエラーを返した。

```
Invalid argument `config.scrollZoom` passed into Graph with ID "{"row":1,"type":"wf-graph"}".
Expected `boolean`. Was supplied type `string`.
```

### 原因

Plotly.js 自体は `scrollZoom: "x"` を受け付けるが、
Dash の `dcc.Graph` コンポーネントの prop validation は `boolean` のみを許容する。

### 対処

`scrollZoom` は `True`/`False`（boolean）のみ使用し、
Y軸方向のズーム制限は `yaxis.fixedrange=True` で実現する。

```python
# Before（エラー）
config={"scrollZoom": "x"}

# After（修正）
config={"scrollZoom": True}  # boolean のみ
yaxis=dict(fixedrange=True)  # Y軸ズーム無効化
```

### 教訓

Plotly.js と Dash (Python wrapper) で受け付ける型が異なるケースがある。
Dash のドキュメントで型制約を確認すること。

---

## 問題3: ctx.triggered_id 判定でズーム同期が発火しない

### 症状

1行でスクロールズームしても、他の行にX軸レンジが反映されなかった。

### 原因

`update_graphs` callback で `ctx.triggered_id` によりイベント種別を判定し、
shapes更新とrange同期を条件分岐していた。

```python
# Before
if triggered in ("hover-x-store", "cursor-a-store", "cursor-b-store"):
    p["layout"]["shapes"] = shapes
if triggered == "xaxis-range-store":
    p["layout"]["xaxis"]["range"] = xaxis_range
```

複数の Input が同時発火した場合や、Dash のコールバック実行順序によって
`triggered_id` が期待値と一致せず、range同期が実行されないケースがあった。

### 対処

`ctx.triggered_id` による条件分岐を廃止し、shapes と range を常に全て適用するよう変更。

```python
# After — 常に全更新を適用
p["layout"]["shapes"] = shapes
if xaxis_range:
    p["layout"]["xaxis"]["range"] = xaxis_range
    p["layout"]["xaxis"]["autorange"] = False
else:
    p["layout"]["xaxis"]["autorange"] = True
```

### 教訓

複数 Input を持つ Callback では `ctx.triggered_id` による分岐を避け、
冪等な全適用方式にする方が安定する。Patch() の差分更新なら不要な項目を送っても軽量。

---

## 問題4: Y軸目盛り桁数の違いでグラフ左端がずれる

### 症状

行ごとにY軸の値の桁数が異なる場合（例: -10〜10 vs -1000〜1000）、
Plotly が Y軸ラベル幅を自動調整するため、グラフ描画領域の左端が行ごとにずれていた。
スパイクラインやカーソル線は正しく時間軸に揃っているが、視覚的にずれて見える。

### 対処

3段階の対策を適用:

1. **`yaxis.automargin=False`**: Plotly の自動マージン調整を無効化
2. **`margin.l=60`**: 左マージンを固定値に設定
3. **`yaxis.tickformat=".3s"`**: SI接頭辞表記（1k, 1M, 1m 等）で桁数を一定化

SI接頭辞表記により、値の桁数に関わらずラベル幅が一定になるため、
固定マージンでも大きな値・小さな値の両方に対応できる。

### 教訓

Plotly の `automargin` は便利だが、複数Graph を縦に並べる場合は
グラフ間のアライメントを崩す原因になる。
固定マージン + 固定幅の tickformat の組み合わせが最も安定する。
