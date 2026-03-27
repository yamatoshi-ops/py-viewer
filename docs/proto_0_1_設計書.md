---
project: Panel_y
doc_type: 設計書
target: Proto #0.1 - Hello Dash
created: "2026-03-15"
updated: "2026-03-15"
status: 完成・動作確認済み
---

# Proto #0.1 設計書 — Hello Dash

---

## 1. 機能一覧

| # | 機能 | 概要 |
|---|------|------|
| 1 | **Parquetデータ読み込み** | 固定パスの `.parquet` ファイルをアプリ起動時に読み込む |
| 2 | **チャンネル自動検出** | `time` 列以外の全列をチャンネルとして自動認識 |
| 3 | **複数チャンネル波形表示** | チャンネル数に応じたサブプロットを動的に生成して表示 |
| 4 | **時間軸同期** | 全パネルのパン・ズームが連動（`shared_xaxes=True`） |
| 5 | **ホバー値表示** | カーソル位置の全チャンネル値を同時表示（X unified モード） |
| 6 | **スパイクライン** | ホバー時に垂直スパイクラインを全パネルに表示 |
| 7 | **スクロールズーム** | マウスホイールで時間軸を拡大・縮小 |
| 8 | **サンプルデータ生成** | モータ制御想定の2ch波形（電圧・電流）を Parquet で生成するスクリプト |

---

## 2. 使用方法

### 環境構築（初回のみ）

```bash
cd proto_0_1
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### サンプルデータ生成（初回のみ）

```bash
python generate_sample.py
# → sample_data/sample_waveform.parquet が生成される
```

### アプリ起動

```bash
python app.py
# → http://localhost:8050 をブラウザで開く
```

### 操作方法

| 操作 | 動作 |
|------|------|
| マウスホバー | カーソル位置の全チャンネル値を表示 |
| ドラッグ | 時間軸をパン（左右スクロール）。全パネル連動 |
| マウスホイール | 時間軸をズームイン・アウト。全パネル連動 |
| ダブルクリック | ズームリセット（全体表示に戻る） |
| 右上ツールバー | 拡大・縮小・保存（PNG）などのPlotly標準操作 |

---

## 3. 実装設計書

### ファイル構成

```
proto_0_1/
├── app.py                        # Dashアプリ本体
├── generate_sample.py            # サンプルParquet生成スクリプト
├── requirements.txt              # 依存パッケージ
├── sample_data/
│   └── sample_waveform.parquet   # 生成されたサンプルデータ（git除外）
└── .venv/                        # Python仮想環境（git除外）
```

### 依存パッケージ

| パッケージ | バージョン | 用途 |
|-----------|-----------|------|
| `dash` | >=2.16.0 | WebアプリフレームワークとUI |
| `plotly` | >=5.20.0 | グラフ描画 |
| `pandas` | >=2.0.0 | Parquetデータ読み込み・操作 |
| `numpy` | >=1.26.0 | サンプルデータ生成（正弦波計算） |
| `pyarrow` | >=15.0.0 | Parquetエンジン（pandas依存） |

### データ仕様

**Parquetフォーマット（`sample_waveform.parquet`）**

| カラム | 型 | 内容 |
|--------|-----|------|
| `time` | float64 | 時刻 [s] |
| `voltage_u` | float64 | U相電圧 [V] |
| `current_u` | float64 | U相電流 [A] |

**サンプルデータのパラメータ（`generate_sample.py`）**

| パラメータ | 値 | 内容 |
|-----------|-----|------|
| `SAMPLE_RATE` | 10,000 Hz | サンプリング周波数 |
| `DURATION` | 0.1 s（100ms） | 記録時間 |
| `FREQ` | 50 Hz | 基本波周波数 |
| 電圧振幅 | 200 V | 正弦波 |
| 電流振幅 | 10 A | 正弦波・30° 遅れ位相 |
| 総サンプル数 | 1,000 点 | |

### app.py 処理フロー

```
起動
 │
 ├─ Parquet読み込み（pd.read_parquet）
 │   └─ time列以外をチャンネルとして自動検出
 │
 ├─ Plotly Figure生成
 │   ├─ make_subplots（チャンネル数 × 1列、shared_xaxes=True）
 │   ├─ チャンネルごとに go.Scatter を追加
 │   ├─ X軸: spikemode="across"、最下段のみ "Time [s]" ラベル
 │   └─ layout: hovermode="x unified"、template="plotly_dark"
 │
 └─ Dashレイアウト定義
     ├─ html.H2（タイトル）
     └─ dcc.Graph（figure=fig、scrollZoom=True）
```

### 設計上のポイント

**チャンネル自動検出**
```python
channels = [col for col in df.columns if col != "time"]
```
Parquet のカラム名をそのままチャンネル名・Y軸ラベルとして使用。チャンネル数が変わっても `app.py` を修正不要。

**時間軸同期**
```python
make_subplots(shared_xaxes=True)
```
Plotly の `shared_xaxes` によりパン・ズームが全パネルで自動連動する。Callback 不要で実装。

**グラフ高さの動的計算**
```python
height=300 * len(channels)
```
チャンネル数に応じてグラフ全体の高さを自動調整。

---

## 4. 改善点

> ※ Proto #1 以降での実装候補として記録

| # | 改善点 | 理由・背景 |
|---|--------|-----------|
| 1 | **ファイル選択UIがない** | 現状はパスをハードコード。任意のParquetを読み込めない |
| 2 | **起動後にデータを変更できない** | Callback なし。データの差し替えにはサーバー再起動が必要 |
| 3 | **チャンネル表示・非表示の切り替えができない** | 凡例クリックで消えるが、パネル自体は残る |
| 4 | **カーソル差分計算がない** | Δt・ΔV・ΔAの計算機能なし |
| 5 | **単位がラベルに未反映** | Y軸ラベルがチャンネル名のみ（単位なし） |
| 6 | **データ前処理ユーティリティがない** | CSV/MATLABデータの変換機能なし |
| 7 | **大容量データへの対応がない** | ダウンサンプリングなし。点数が多いと描画が重くなる可能性あり |
