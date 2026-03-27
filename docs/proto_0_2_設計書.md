---
project: Panel_y
doc_type: 設計書
target: Proto #0.2
created: "2026-03-15"
updated: "2026-03-15"
status: 完成・動作確認済み
---

# Proto #0.2 設計書

---

## 1. 機能一覧

| # | 機能 | #0.1 との差分 |
|---|------|--------------|
| 1 | **行レイアウト表示** `[ch名/値] [波形グラフ]` | **NEW** |
| 2 | **全チャンネルのホバー値同時表示** | **NEW**（#0.1は未実装） |
| 3 | **全グラフ縦断スパイクライン** | **FIX**（#0.1/0.2初稿は1パネルのみ） |
| 4 | **グラフ間ズーム・パン同期** | **NEW** |
| 5 | Parquetデータ読み込み | #0.1 から継続 |
| 6 | チャンネル自動検出 | #0.1 から継続 |
| 7 | スクロールズーム | #0.1 から継続 |
| 8 | ツールバー（ホバー表示） | **FIX**（#0.2初稿で消えていたものを復元） |

---

## 2. 使用方法

### 起動

```bash
cd proto_0_2
source ../proto_0_1/.venv/bin/activate   # proto_0_1 の venv を共用
python app.py
# → http://localhost:8050
```

### 操作方法

| 操作 | 動作 |
|------|------|
| グラフ上でマウスホバー | 左列に全チャンネルの値を同時表示。全グラフに縦断スパイクラインを表示 |
| ドラッグ | 時間軸をパン。全グラフが連動 |
| マウスホイール | 時間軸をズーム。全グラフが連動 |
| ダブルクリック | ズームリセット（全体表示に戻る） |
| ツールバー（マウスオーバーで出現） | ズーム・パン・PNG保存など Plotly 標準操作 |

---

## 3. 実装設計書

### ファイル構成

```
proto_0_2/
└── app.py      # Dashアプリ本体（proto_0_1 の sample_data を参照）
```

> `sample_data/` と `.venv/` は `proto_0_1/` を共用。`requirements.txt` は不要。

### 全体アーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│ app.layout                                          │
│  ┌──────────────┬──────────────────────────────┐   │
│  │ Channel/Value│ dcc.Graph(id="graph-voltage") │   │  ← channel_row()
│  │  "voltage_u" │                              │   │
│  │  id=val-volt │    [波形グラフ]               │   │
│  ├──────────────┼──────────────────────────────┤   │
│  │ "current_u"  │ dcc.Graph(id="graph-current") │   │  ← channel_row()
│  │  id=val-curr │    [波形グラフ]               │   │
│  └──────────────┴──────────────────────────────┘   │
│                                                     │
│  dcc.Store(id="hover-x-store")                      │
│  dcc.Store(id="xaxis-range-store")                  │
└─────────────────────────────────────────────────────┘
```

### Callback フロー

```
[ユーザー操作]
      │
      ├─ グラフ上でホバー
      │       │
      │       ▼
      │  Callback 1: store_hover_x
      │    Input:  graph-{ch}.hoverData（全ch）
      │    Output: hover-x-store.data（X座標）
      │       │
      │       ├──▶ Callback 2: update_values
      │       │      Input:  hover-x-store.data
      │       │      Output: val-{ch}.children（全ch）
      │       │      処理:   df["time"] の idxmin で最近傍インデックスを取得
      │       │
      │       └──▶ Callback 4: update_graphs（triggered="hover-x-store"）
      │              → 全グラフに yref="paper" の縦線シェイプを Patch で追加
      │
      └─ ズーム・パン操作
              │
              ▼
         Callback 3: store_xaxis_range
           Input:  graph-{ch}.relayoutData（全ch）
           Output: xaxis-range-store.data（[x_min, x_max] or None）
              │
              ▼
         Callback 4: update_graphs（triggered="xaxis-range-store"）
           → 全グラフの xaxis.range を Patch で同期
```

### 主要実装ポイント

**チャンネル個別 Figure（`make_channel_fig`）**

```python
hoverinfo="none"    # ツールチップは非表示 / hoverData イベントは発火する
hovermode="x"       # X軸方向のホバー検出を有効化
```

> `hoverinfo="skip"` はイベント自体を無効化するため使用不可（bug_report_0_2 参照）

**ホバー値: DataFrame 逆引き**

```python
idx = (df["time"] - x_val).abs().idxmin()   # 最近傍インデックス
return [f"{df[ch].iloc[idx]:.4f}" for ch in channels]
```

**スパイクライン: `yref="paper"` で全グラフ高さを縦断**

```python
{"type": "line", "xref": "x", "yref": "paper", "y0": 0, "y1": 1, ...}
```

> Plotly のネイティブ spike は単一グラフ内にしか描画されないため、
> Dash Callback で shape として動的追加する方式を採用（bug_report_0_1 参照）

**差分更新: `Patch()`**

```python
p = Patch()
p["layout"]["shapes"] = [...]   # figureの該当箇所だけ更新
```

> figure 全体を返すと通信コストが大きいため Dash 2.9+ の `Patch()` を使用

**ズーム同期: `ctx.triggered_id` で起因を判別**

```python
triggered = ctx.triggered_id
if triggered == "hover-x-store":   # ホバー由来 → スパイクのみ更新
if triggered == "xaxis-range-store": # ズーム由来 → xaxis.range のみ更新
```

> 1つの Callback で2種のイベントを処理し、Output の競合（1 Output に複数 Callback 不可）を回避

**ツールバー設定**

```python
config={
    "scrollZoom": True,
    "displayModeBar": "hover",                            # マウスオーバー時のみ表示
    "modeBarButtonsToRemove": ["select2d", "lasso2d"],    # 不要ボタンを除外
}
```

---

## 4. 改善点

> ※ Proto #1 以降での実装候補として記録

| # | 改善点 | 理由・背景 |
|---|--------|-----------|
| 1 | **ファイル選択 UI がない** | データパスがハードコード。任意ファイルを読み込めない |
| 2 | **Y 軸スケールの単位が未表示** | チャンネル名のみで単位（V, A）がない |
| 3 | **チャンネル表示・非表示の切り替えがない** | 多チャンネル時に特定 ch を隠せない |
| 4 | **カーソル差分計算がない** | Δt・ΔV・ΔA の計算機能なし |
| 5 | **ホバー値がグラフ上ではなく左列のみ** | グラフ内インラインで見たいケースがある |
| 6 | **マウスアウト時に値が残る** | グラフからマウスが離れると `---` に戻らない |
| 7 | **データ前処理ユーティリティがない** | CSV/MATLAB → Parquet 変換機能なし |

---

## 5. 参照バグレポート

- [bug_report_0_1.md](./bug_report_0_1.md) — スパイクライン全パネル非連動（make_subplots 軸参照問題）
- [bug_report_0_2.md](./bug_report_0_2.md) — ホバー値・スパイク・ツールバー消失（hoverinfo/displayModeBar 設定ミス）
