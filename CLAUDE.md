# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

東京の海面更正気圧（`pressure_msl`）を1時間ごと・7日先まで表示し、急な気圧低下を警告する単一ファイルの静的 Web アプリ。`index.html` に HTML / CSS / JavaScript がすべて含まれ、依存ライブラリ・ビルドツール・パッケージマネージャは一切使わない。ホスティングは GitHub Pages。

## 開発・実行

ビルド・テスト・lint の仕組みは無い。動作確認はブラウザで `index.html` を開くだけだが、`file://` で直接開くと Open-Meteo API への通信がブラウザにブロックされることがある。確実に動かすにはローカルサーバ経由で開く:

```sh
python3 -m http.server 8000
# ブラウザで http://localhost:8000/index.html を開く
```

## アーキテクチャ

`<script>` 内の処理は次の流れ。すべてグローバル関数で、状態管理ライブラリは使わない。

- `load()` — `ENDPOINTS`（`forecast` → `gfs` の順にフォールバック）を順に fetch し、最初に成功したレスポンスで `render()` を呼ぶ。全滅した場合は `#content` に原因別のエラー UI を描画する。
- `detectDrops(times, vals)` — 気圧低下区間を検出する。「6時間で3hPa以上の低下」を全点で判定し、近接する検出（インデックス差 ≤ 3）を1つの区間にまとめて返す。警告カードとグラフ上の赤いハイライト領域の両方に使われる。
- `render(data)` — 現在 / 6時間後 / 24時間後の気圧と変化量、7日間の範囲のカード、警告 / 安全メッセージを組み立てて `#content` に流し込み、最後に `drawChart()` を呼ぶ。
- `drawChart(times, vals, unit, drops)` — グラフを SVG 文字列で生成する（描画ライブラリ不使用）。

## グラフ実装の要点

- **固定Y軸 + 横スクロール本体の2レイヤー構成**。`#yAxisFixed`（固定 SVG、左端 `padL=48px` 幅、不透明背景で本体を隠す）に目盛りラベル、`#chartScroll` 内の本体 SVG（幅 `W = max(820, n*4.6)`）にグリッド線・折れ線・面・日付境界・低下ハイライトを描く。Y軸ラベルだけが横スクロールに追従しない。
- **タッチ操作の出し分け**: `touchmove` で横移動が大きい（`dx > 10 && dx > dy`）ときはブラウザの横スクロールに任せ、ほぼ静止（`dx < 8 && dy < 8`）のときだけツールチップ読み取りモードに切り替えて `preventDefault()` する。スクロールとツールチップ表示を両立させるための分岐なので、ここを変更するときは両挙動を必ず確認すること。

## 設定値

- 地点は `index.html` 冒頭の `LAT = 35.68`, `LON = 139.69`（東京）。変更すれば他地点に対応できる。
- 警告のしきい値は `detectDrops()` 内のハードコード（6時間で3hPa）。README とフッターの注記（24時間で約8hPa の記述）と整合させて変更すること。
