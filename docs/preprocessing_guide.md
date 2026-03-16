# Panel_y 前処理スクリプト 実行手順書

データ（CSV / MAT）を Panel_y で読み込める Parquet 形式に変換する手順。

| スクリプト | 用途 |
|-----------|------|
| `utils/converter.py` | 単体ファイルの変換（CLI） |
| `multiconverter.py` | フォルダ内 .mat を一括変換（CLI） |

---

## 前提条件

- Python 3.13
- `proto_1/.venv` が作成済みであること

venv が未作成の場合は先にセットアップを行う:

```bash
cd /Users/sugusokothx/newspace/panel-y/proto_1
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

---

## 変換処理の流れ

```
入力ファイル（CSV / MAT / Parquet）
  │
  ├─ [1] ファイル読み込み
  ├─ [2] 時間カラム検出（自動 or 手動指定）
  ├─ [3] 時間単位検出・秒単位に正規化 → カラム名を "time" に統一
  ├─ [4] 非数値カラム除外
  └─ [5] 等間隔チェック → 非等間隔なら線形補間でリサンプリング
         │
         ▼
     出力 Parquet（time + 数値チャンネル）
```

---

## 実行方法

### 1. venv を有効化

```bash
cd /Users/sugusokothx/newspace/panel-y/proto_1
source .venv/bin/activate
```

### 2. 変換コマンドを実行

#### CSVファイルの変換（基本）

```bash
python -m utils.converter input.csv data/output.parquet
```

#### CSVファイル — 時間カラムを明示指定

時間カラムが `time`, `t`, `Time`, `TIME`, `timestamp`, `elapsed`, `ECUTIME` 以外の名前の場合:

```bash
python -m utils.converter input.csv data/output.parquet --time-col <カラム名>
```

#### CSVファイル — 時間単位を明示指定

自動推定が不正確な場合（例: 値が `1000` 未満だが単位が ms のとき）:

```bash
python -m utils.converter input.csv data/output.parquet --time-unit ms
```

| `--time-unit` の値 | 意味 |
|--------------------|------|
| `s`                | 秒（デフォルト扱い） |
| `ms`               | ミリ秒 |
| `us`               | マイクロ秒 |

#### MATファイルの変換（MATLAB）

MATLAB v7.2 以前 / v7.3（HDF5）の両形式に対応:

```bash
python -m utils.converter data.mat data/output.parquet
```

#### 全オプション同時指定

```bash
python -m utils.converter input.csv data/output.parquet \
  --time-col elapsed \
  --time-unit ms
```

---

## 一括変換（multiconverter）

フォルダ直下の `.mat` ファイルをまとめて変換する場合に使用する。

### 実行方法

```bash
cd /Users/sugusokothx/newspace/panel-y/proto_1
source .venv/bin/activate

# 基本
python multiconverter.py --input ./input --output ./output

# 既存の .parquet を上書きする場合
python multiconverter.py --input ./input --output ./output --force
```

### オプション

| オプション | 必須 | 説明 |
|-----------|------|------|
| `--input` | ○ | 変換対象の .mat が置かれたフォルダ |
| `--output` | ○ | Parquet の出力先フォルダ（存在しなければ自動作成） |
| `--force` | — | 同名 .parquet が既にある場合でも上書きする |

### 動作仕様

- スキャン対象: `--input` フォルダ**直下**の `.mat` ファイルのみ（サブフォルダは対象外）
- 出力ファイル名: 入力のステム名をそのまま使用（`foo.mat` → `foo.parquet`）
- 時間カラムは自動検出（`time`, `t`, `Time`, `TIME`, `timestamp`, `elapsed`, `ECUTIME`）
- 1ファイル失敗しても残りのファイルの変換を継続する
- 失敗ファイルが1件以上ある場合は exit code 1 で終了

### 実行時の出力例

```
[スキャン完了] 3 件の .mat ファイルを検出

--- foo.mat ---
[読み込み完了] foo.mat  shape=(50000, 8)
[出力完了] output/foo.parquet
  チャンネル: ['iu', 'iv', 'iw', 'vu', 'vv', 'vw', 'torque']
  サンプル数: 50,000 点
  サンプリング: 10,000.0 Hz (Ts=0.1000 ms)

--- bar.mat ---
[スキップ] bar.mat → bar.parquet (既存)

--- baz.mat ---
[読み込み完了] baz.mat  shape=(30000, 6)
[出力完了] output/baz.parquet
  ...

========================================
変換サマリー
========================================
  成功  : 2 件
  スキップ: 1 件
  失敗  : 0 件
```

失敗ファイルがある場合はサマリーに一覧が表示される:

```
  失敗  : 1 件

失敗ファイル一覧:
  - broken.mat: MATLAB ファイルの読み込みに失敗しました。...
```

---

## 実行時の出力例（単体変換）

```
[読み込み完了] input.csv  shape=(50000, 8)
[時間変換] 'elapsed' を ms → s に変換します
[出力完了] data/output.parquet
  チャンネル: ['iu', 'iv', 'iw', 'vu', 'vv', 'vw', 'torque']
  サンプル数: 50,000 点
  サンプリング: 10,000.0 Hz (Ts=0.1000 ms)
```

非等間隔データの場合は追加で以下が表示される:

```
[警告] 非等時間間隔データを検出。線形補間でリサンプリングします
```

---

## トラブルシューティング

### 時間カラムを自動検出できない

**エラー:**
```
ValueError: 時間カラムを自動検出できませんでした。カラム一覧: ['...']
```

**対処:** `--time-col` でカラム名を明示する。

```bash
python -m utils.converter input.csv data/out.parquet --time-col <カラム名>
```

---

### 時間単位の自動推定が間違っている

**症状:** Parquet の `time` カラムの値が異常に大きい / 小さい

**原因:** 時間単位の自動推定は最大値で判定する（<1000 → s, <1,000,000 → ms）。
値の範囲と実際の単位が合わない場合は誤判定する。

**対処:** `--time-unit` で明示する。

---

### MAT ファイルの読み込みに失敗する

**エラー:**
```
ValueError: MATLAB ファイルの読み込みに失敗しました。
```

**確認事項:**
- `h5py` がインストールされているか: `pip install h5py`
- MATLAB でファイルを `-v7.3` 形式（HDF5）または `-v7` 以前で保存しているか

---

### 波形チャンネルが0本のエラー

**エラー:**
```
ValueError: 波形チャンネルが0本です（全て非数値 or 時間のみ）
```

**原因:** 数値型カラムが時間カラムしかない。CSVの文字列カラムが多い場合など。

**対処:** 入力データに数値型の波形チャンネルが含まれているか確認する。

---

## 出力 Parquet の仕様

| カラム | 型 | 内容 |
|--------|-----|------|
| `time` | float64 | 時間軸（秒、0始まりではなく元データに準ずる） |
| 各チャンネル名 | float64 | 波形データ（非数値カラムは除外済み） |

- 等間隔保証（非等間隔は線形補間済み）
- `proto_1/data/` ディレクトリに配置すると Panel_y アプリのパス補完で検索される
---
