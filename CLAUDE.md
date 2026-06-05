# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

東京の海面更正気圧（`pressure_msl`）を1時間ごと・7日先まで表示し、急な気圧低下を警告する単一ファイルの静的 Web アプリ。加えて、愛犬 Cobu の体調不良をグラフから記録し一覧で振り返る機能を持つ。`index.html` に HTML / CSS / JavaScript がすべて含まれ、ビルドツール・パッケージマネージャは使わない。気圧表示まわりは依存ライブラリ非使用。体調記録の複数端末同期は任意で、Firebase が設定されているときだけ Firestore SDK を CDN から**動的 import** する（未設定時は localStorage にフォールバックし、気圧グラフは Firebase の有無に関係なく動く）。`<script>` は ES module（`type="module"`）。ホスティングは GitHub Pages。

## 開発・実行

ビルド・テスト・lint の仕組みは無い。動作確認はブラウザで `index.html` を開くだけだが、`file://` で直接開くと Open-Meteo API への通信がブラウザにブロックされることがある。確実に動かすにはローカルサーバ経由で開く:

```sh
python3 -m http.server 8000
# ブラウザで http://localhost:8000/index.html を開く
```

## アーキテクチャ

`<script type="module">` 内の処理は次の流れ。すべてモジュールスコープの関数で、状態管理ライブラリは使わない。

- `load()` — `ENDPOINTS`（`jma` → `forecast` → `gfs` の順にフォールバック）を順に fetch し、最初に成功したレスポンスで `render()` を呼ぶ。全滅した場合は `#content` に原因別のエラー UI を描画する。いずれも `past_days=7`（直近7日の実績）＋ `forecast_days=7`（予報）で取得する。
- `detectDrops(times, vals)` — 気圧低下区間を検出する。「6時間で3hPa以上の低下」を全点で判定し、近接する検出（インデックス差 ≤ 3）を1つの区間にまとめて返す。警告カードとグラフ上の赤いハイライト領域の両方に使われる。
- `render(data)` — 現在 / 6時間後 / 24時間後の気圧と変化量、7日間の範囲のカード、警告 / 安全メッセージを組み立てて `#content` に流し込み、最後に `drawChart()` を呼ぶ。
- `drawChart(times, vals, unit, drops)` — グラフを SVG 文字列で生成する（描画ライブラリ不使用）。

## 体調記録の実装

- **保存層の抽象化（`store`）** — `store.add` / `store.update` / `store.remove` と購読の4つだけを持つ薄いオブジェクト。`initStore()` が `firebaseEnabled()`（`firebaseConfig.projectId` が `YOUR_` を含まないか）で実装を選ぶ。設定済みなら Firebase を動的 import して `onSnapshot` でリアルタイム購読（コレクション `cobu_logs`、`createdAt` 降順）、未設定/初期化失敗なら `localStore()`（`localStorage` キー `cobu_logs_v1`）にフォールバック。どちらの経路でも記録が変わるたび `LOGS` を更新して `renderLogs()` を呼ぶ。
- **記録1件** — `{ time, memo, pressure, series, createdAt }`。`time` は datetime-local と同じ壁時計文字列。`pressure` は `nearestPressure()` が取得範囲内の最近傍値（範囲外は `null`）を入れたスナップショット。`series` は `snapshotSeries()` が記録日を含む3カレンダー日（`miniWindow()` の `[前々日0:00, 翌日0:00)`）の気圧列を `{ start, vals }` で切り出したスナップショット。どちらも保存時に確定するため、後日データ取得範囲（直近7日）から外れた古い記録でも、気圧の値とミニグラフを復元できる（履歴の自前保持）。`series` を持たない旧記録は `ALL` 復元にフォールバックし、範囲外なら「取得範囲外」になる。
- **入力 UI** — グラフ下の `#recBar`（「記録」ボタン `#recBtn`）は常時表示。`drawChart` 内 `move()` がグラフのなぞりで `selectedTime` を更新しラベルを「選択中: …」に変える。「記録」ボタンは選択の有無にかかわらず押せ、`openRecordDialog(selectedTime)` が `<dialog id="recDialog">` を開く（日時は編集可、初期値は選択時刻、未選択なら現在時刻）。`drawChartView()` のモード切替時は `selectedTime` をリセットしラベルを未選択案内へ戻す。
- **一覧と3日ミニグラフ** — `renderLogs()` が `#logsPanel`（`#content` の外なので気圧取得失敗時も生きる）にカードを描画。カードのタップはパネルへのイベント委譲で処理（展開時は先に表示→`drawMiniChart()` の順で、コンテナ幅を確定させてから描く）。カードを開くと `drawMiniChart()` が記録日を含む3カレンダー日（`miniWindow()`、記録日が右端）の静的 SVG（ツールチップなし）を描き、記録時刻に縦マーカーを打つ。**メイングラフと同一縮尺**（`H=320`・横密度 4.6px/h・共通 `yRange`・メインと同じ `pressure-line`/`gridline`/`axis-label`/`day-label` クラス）で描くので低下傾斜を直接見比べられ、カードに収まらない時だけ横スクロール（`.log-mini-chart { overflow-x:auto }`、初期位置は記録日付近へ寄せる）。データは保存済み `series` を優先し、無ければ `ALL` から切り出す（どちらも不可なら「取得範囲外」）。メモの編集はカードを開いた下部の「✏️ メモを編集」から `<dialog id="editDialog">` を開き、`store.update(id, { memo })` で本文だけ更新する（日時・`pressure`・`series` のスナップショットは変えない＝過去の気圧履歴を壊さない）。削除はその隣の「🗑 記録を削除」から `<dialog id="delDialog">` で確認してから `store.remove()` を実行する。

## バックアップと復元（外部 GAS・任意）

Firestore の体調記録のバックアップ／復元は、**このリポジトリ外の Google Apps Script（GAS）**で運用する。コードはリポジトリに含めない（リポジトリ内を探さないこと）。認証はサービスアカウント（Firestore REST API＋自前 JWT 署名）で、鍵は GAS のスクリプトプロパティ `SA_EMAIL` / `SA_PRIVATE_KEY` に置く（コードに鍵は書かない）。アプリ側（`index.html`）はバックアップ／復元の機能を持たない。

- **バックアップ** — 週次（毎週月曜 6 時台のトリガー）に `cobu_logs` を全件読み出し、スクリプトと同じ Drive フォルダへ `cobu_logs_backup.json` として保存。同名上書きで**最新 1 個だけ**を残す。
- **復元** — `restoreCobuLogs` を手動実行。その JSON を**元のドキュメント ID のまま** Firestore に書き戻す（REST の `PATCH`＝`setDoc` 相当の全フィールド上書き）。ID 単位の上書きなので**冪等なマージ**（二重登録なし・既存は消えない）。記録は `{ time, memo, pressure, series, createdAt }` と `index.html` と同じスキーマなので、`series` を含めてそのまま復元できる。
- **IAM ロール** — バックアップ（読み取り）は `Cloud Datastore 閲覧者`、復元（書き込み）は `Cloud Datastore ユーザー` が必要。閲覧者のまま復元すると 403。

## グラフ実装の要点

- **固定Y軸 + 横スクロール本体の2レイヤー構成**。`#yAxisFixed`（固定 SVG、左端 `padL=48px` 幅、不透明背景で本体を隠す）に目盛りラベル、`#chartScroll` 内の本体 SVG（幅 `W = max(820, n*4.6)`）にグリッド線・折れ線・面・日付境界・低下ハイライトを描く。Y軸ラベルだけが横スクロールに追従しない。
- **Y軸の値域はメイングラフとミニグラフで共通**。`yRange(vals)` が基準域 `Y_BASE_LO=985`〜`Y_BASE_HI=1025`(hPa) を必ず確保し、データがはみ出たら2hPa刻みで上下に自動拡張する。両グラフが同じ関数で値域を決めるので、記録カードのミニグラフとメイングラフを同じスケールで見比べられる。
- **タッチ操作の出し分け**: `touchmove` で横移動が大きい（`dx > 10 && dx > dy`）ときはブラウザの横スクロールに任せ、ほぼ静止（`dx < 8 && dy < 8`）のときだけツールチップ読み取りモードに切り替えて `preventDefault()` する。スクロールとツールチップ表示を両立させるための分岐なので、ここを変更するときは両挙動を必ず確認すること。

## 設定値

- 地点は `index.html` 冒頭の `LAT = 35.68`, `LON = 139.69`（東京）。変更すれば他地点に対応できる。
- 警告のしきい値は `detectDrops()` 内のハードコード（6時間で3hPa）。README とフッターの注記（24時間で約8hPa の記述）と整合させて変更すること。
- 体調記録を複数端末で共有する場合は、`index.html` 冒頭の `firebaseConfig` に Firebase コンソールの値を貼る。未設定（`YOUR_...` のまま）だと localStorage 保存になる。Firestore のセキュリティルールと併せた手順は README「体調記録」節を参照。認証なしルールのリスクも同節に記載。
- 悪用防止に App Check（reCAPTCHA v3）を使う。`index.html` 冒頭の `RECAPTCHA_SITE_KEY` に reCAPTCHA v3 のサイトキーを設定すると、`initStore()` が `firebase-app-check.js` を動的 import して `initializeAppCheck` を呼ぶ（未設定＝`YOUR_...` なら初期化しない）。reCAPTCHA のシークレットキーは Firebase コンソール側にのみ入れ、コードには置かない。設定〜enforce の手順と App Check の限界（人は区別しない）は README「不正な書き込みを防ぐ」節を参照。
