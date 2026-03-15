# Panel_y

電力変換器・モータ制御の波形データをブラウザ上でインタラクティブに可視化・解析するツール。

## 技術スタック

- Python + Dash (Plotly社)
- Plotly Python
- Pandas / SciPy / NumPy
- 標準データ形式: Parquet

## ディレクトリ構成

```
panel-y/
├── proto_0_1/          # Proto #0.1 - Hello Dash（2チャンネル波形表示）
│   ├── app.py
│   ├── sample_data/
│   └── requirements.txt
├── proto_1/            # Proto #1 - 実用ビューア
│   ├── app.py
│   ├── utils/
│   │   └── converter.py    # CSV/MAT → Parquet 変換ユーティリティ
│   ├── data/               # 作業用データ置き場（.gitignore除外）
│   └── requirements.txt
└── docs/               # コード寄りの技術メモ
```

## 企画書・検討メモ

`~/life/.secretary/projects/panel-y/` に格納。

## 起動方法

```bash
# Proto #0.1
cd proto_0_1
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python app.py
# → http://localhost:8050
```
