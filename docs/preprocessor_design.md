---
project: Panel_y
doc_type: 設計書
target: 前処理ユーティリティ
created: "2026-03-15"
updated: "2026-03-16"
status: 実装済み
---

# 前処理ユーティリティ 設計書

---

## 1. 目的

異なる形式・品質の入力データを Panel_y 標準 Parquet 形式に変換する。
**Panel_y 本体は Parquet のみを受け付ける**という前提のもと、このユーティリティがすべての差異を吸収する。

---

## 2. 責務と非責務

| 責務（やること） | 非責務（やらないこと） |
|----------------|------------------|
| 時間カラムを `time`（秒）に正規化 | グラフ描画・可視化 |
| 非数値カラムの除外 | Panel_y 本体の起動 |
| 等時間間隔への補間・リサンプリング | データの解析・FFT |
| CSV / MAT → Parquet 変換 | ファイル保存先の管理 |
| 時間単位の変換（ms, us → s） | |

---

## 3. 実装機能一覧

### 機能 A: CSV → Parquet 変換

**入力**: CSV ファイル（任意のカラム構成）
**出力**: Panel_y 標準 Parquet

処理手順:
1. `pandas.read_csv()` で読み込み
2. 時間カラムを特定（後述: 時間カラム検出ロジック）
3. 時間カラムを秒単位の `time` に正規化
4. 非数値カラムを除外
5. 等時間間隔チェック → 非等間隔なら補間
6. `to_parquet(index=False)` で出力

---

### 機能 B: MATLAB .mat → Parquet 変換

**入力**: .mat ファイル（v7.2 以前 / v7.3 以降）
**出力**: Panel_y 標準 Parquet

処理手順:
1. バージョン判定
   - v7.2 以前: `scipy.io.loadmat()`
   - v7.3 以降: `h5py` で読み込み
2. 変数名一覧を取得し、時間変数を特定（後述）
3. 時間を秒単位の `time` に正規化
4. 数値配列のみ抽出してチャンネルとして登録
5. 等時間間隔チェック → 非等間隔なら補間
6. `to_parquet(index=False)` で出力

---

### 機能 C: 時間カラム検出・正規化

**目的**: 入力データが `time` という名前でなくても正しく時間軸を特定する

検出優先順位:
1. カラム名が `time`（完全一致）→ そのまま使用
2. カラム名が `t`, `Time`, `TIME`, `timestamp`, `elapsed`, `ECUTIME` → `time` にリネーム
3. 上記に該当しない場合 → **ユーザーに対話的に選択させる**（CLIオプション or 引数）

時間単位の自動検出（ヒューリスティック）:
| 判定条件 | 推定単位 | 変換 |
|---------|--------|------|
| 最大値 < 1000 かつ 最小値 >= 0 | 秒（s） | そのまま |
| 最大値 >= 1000 かつ < 1,000,000 | ミリ秒（ms） | ÷ 1,000 |
| 最大値 >= 1,000,000 | マイクロ秒（μs） | ÷ 1,000,000 |

> ユーザーが明示的に単位を指定した場合はそちらを優先する。

---

### 機能 D: 非数値カラムの除外

- `pandas.api.types.is_numeric_dtype()` で判定
- 非数値カラムは処理時にログに出力してから除外する
- 時間カラムは除外対象外

---

### 機能 E: 等時間間隔チェック・補間

**チェック**:
```python
diffs = time.diff().dropna()
is_uniform = (diffs.max() - diffs.min()) / diffs.mean() < 1e-6  # 相対誤差 1ppm 以内
```

**非等間隔の場合の処理**:
1. `time[0]` 〜 `time[-1]` の範囲で等間隔の新しい時間軸を生成
   - サンプリング周期: `(time[-1] - time[0]) / (len(time) - 1)`
2. 各チャンネルを `numpy.interp()` で線形補間
3. 変換したことをログに出力

---

## 4. ファイル構成

```
proto_1/
└── utils/
    ├── __init__.py
    ├── converter.py       # 変換エントリポイント（公開API）
    ├── reader_csv.py      # 機能A: CSV読み込み
    ├── reader_mat.py      # 機能B: MATLAB読み込み
    ├── time_normalizer.py # 機能C: 時間カラム検出・正規化
    ├── column_filter.py   # 機能D: 非数値除外
    └── resampler.py       # 機能E: 等間隔チェック・補間
```

---

## 5. 公開 API（`converter.py`）

```python
def convert_to_parquet(
    input_path: str | Path,
    output_path: str | Path,
    time_col: str | None = None,   # 時間カラム名を明示指定（省略時は自動検出）
    time_unit: str | None = None,  # 時間単位を明示指定: "s" | "ms" | "us"
) -> Path:
    """
    任意の入力ファイルを Panel_y 標準 Parquet に変換して保存する。
    Returns: 出力した Parquet ファイルのパス
    """
```

---

## 6. CLI インターフェース

### 単体変換（`python -m utils.converter`）

```bash
# 基本的な使い方
python -m utils.converter input.csv output.parquet

# 時間カラム・単位を明示
python -m utils.converter input.csv output.parquet --time-col elapsed --time-unit ms

# MAT ファイル
python -m utils.converter data.mat output.parquet
```

### 一括変換（`multiconverter.py`）

フォルダ直下の `.mat` ファイルをまとめて Parquet に変換する CLI。
内部で `converter.convert_to_parquet()` を呼び出す。

```bash
# 基本
python multiconverter.py --input ./input --output ./output

# 既存 .parquet を上書き
python multiconverter.py --input ./input --output ./output --force
```

| オプション | 必須 | 説明 |
|-----------|------|------|
| `--input` | ○ | 変換対象の .mat が置かれたフォルダ |
| `--output` | ○ | Parquet の出力先フォルダ（存在しなければ自動作成） |
| `--force` | — | 同名 .parquet が既にある場合でも上書きする |

---

## 7. エラーハンドリング方針

| 状況 | 対応 |
|------|------|
| `time` カラムが見つからない | エラーを出して停止 / --time-col で指定を促す |
| 波形チャンネルが 0 本（全て非数値 or 時間のみ） | エラーを出して停止 |
| 非数値カラムがある | 警告ログを出して除外し処理続行 |
| 非等間隔データ | 警告ログを出して補間し処理続行 |
| MAT バージョン不明 | v7.2 → v7.3 の順で試行。両方失敗でエラー |

---

## 8. 決定事項（旧・未決定事項）

| 項目 | 決定内容 |
|------|---------|
| 単位自動検出のヒューリスティック | 最大値ベース（<1000→s, <1M→ms, ≥1M→us）で実装。`--time-unit` で上書き可能 |
| 補間方式 | `numpy.interp()` による線形補間を採用 |
| CLI ツールの形式 | `python -m utils.converter`（argparse）を採用。追加で `multiconverter.py` も実装 |
