# バグ調査レポート: proto_3_1b 作図ロード後のチャンネル分散表示

- 調査日: 2026-04-16
- 対象: `code/proto_3_1b/app.py`
- 現象: 1行に複数チャンネルを設定して保存した `.pyc.json` をロードすると、重ね表示が崩れてチャンネルが分散して見える

## 結論（原因）

主因は **作図ロード時のコールバック競合** です。

`load_plotconfig()` が復元した波形を描いた直後に、`scroll-zoom-store` 更新をトリガとして `update_waveform_rows()` が再実行され、`waveform-container` を再上書きしています。  
この2つのコールバックが同じ出力先を持つため、後勝ちで表示が崩れる経路が存在します。

## 根拠（コード）

1. `load_plotconfig()` は復元時に以下を同時に更新
- `waveform-container`（直接復元描画）
- `row-groups-store`
- `scroll-zoom-store`

該当: `code/proto_3_1b/app.py:804-825`, `code/proto_3_1b/app.py:984-1002`

2. `update_waveform_rows()` は `scroll-zoom-store` を **Input** に持ち、同じ `waveform-container` を出力

該当: `code/proto_3_1b/app.py:1196-1211`

3. `update_waveform_rows()` は `row-dropdown` の `ALL` State から行グループを再構築して上書き

該当: `code/proto_3_1b/app.py:1232-1244`, `code/proto_3_1b/app.py:1270-1292`

## どう崩れるか

- ロード直後に `load_plotconfig()` で正しく復元されても、続いて `update_waveform_rows()` が実行される
- その時点の `ALL` State（旧UI状態/再マウント中の状態）で再構築されると、保存した行グループではなく別の並びで再描画される
- 結果として「1行重ね表示」が崩れ、チャンネルがばらけて見える

## 影響範囲

- `📂` 作図ロード時の波形表示
- 特に「1行に複数チャンネル」を使うケースで顕在化しやすい

## 補足

`journal/2026-04-13.md` にも、ロード時にパターンマッチ系コールバックが副作用発火しやすい旨の記録があります（同系統の設計リスク）。
