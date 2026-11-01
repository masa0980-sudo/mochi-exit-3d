# CLAUDE.md

## Project

「8番出口」型の **3D 一人称・異変発見ゲーム**（もち出口3D）。同じ通路を歩き、「いつもの通路」と違う所（異変）があれば
引き返し、無ければ奥へ進む。正解で出口番号+1、不正解で0、8で脱出。2D 版 `C:\claude\repos\mochi-exit` の世界観
（和菓子商店街・もち・かわいいが少し不穏）を引き継ぐ。ジャンプスケア・追跡者などのホラー演出は入れない（2026-11-01 ユーザー選択）。

構成: **`index.html` 一枚 + `vendor/three.module.min.js`（three.js r160 を同梱）**、ビルド無し、GitHub Pages。
three.js 同梱は姉妹作の「依存なし」方針の唯一の例外（ユーザーが CDN 読み込み・生 WebGL より同梱を選んだ）。
バージョンを上げるときは vendor のファイルを差し替えるだけ。CDN 参照に変えないこと。

## Commands

- ローカル実行: `python -m http.server 8490` → `http://localhost:8490/index.html`（ES module のため `file://` では動かない）
- 検証: Playwright headless は WebGL のため `--use-gl=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist` を付けて起動し、
  `window.__exit3d` を叩く（下記）
- デプロイ: `main` に push すると `.github/workflows/deploy-pages.yml`（mochi-exit と同じ）が Pages を更新

## Architecture（`index.html` 内の番号付きモジュール）

1. **Sfx** — WebAudio。効果音は必ず `master` ゲイン経由（ミュートを効かせるため）。110Hz 未満・打楽器は使わない（2D 版の音の苦情に倣う）
2. **Renderer** — 通路は z=+13〜-13・幅 3.2・高さ 2.8。スポーン z=10.5、前端トリガー z<-11.8、後端トリガー z>12.2。
   **縦長画面では横の視野が 62° を下回らないよう縦 FOV を広げる（上限 100°）**: 初版は縦持ちスマホで横の視野が狭く、入口から左右の壁の
   ポスターや扉が画面外だった（2026-11-01）
3. **テクスチャ** — 全て Canvas 生成で、乱数を使わない（異変の差分検証を決定的にするため）
4. **buildHall(signNum)** — シーン一式を毎回新規に作る。部品は `s.posters / s.doors / s.panels / s.lights / s.lanterns / s.extraLantern / s.mochi / s.wallL,R / s.sign(setSign)`
5. **ANOMALIES** — `{id, level, desc, apply(s)}`。**apply は s の部品を書き換えるだけ**（2D 版と同じ設計。描画側は異変を知らない）。
   普段は無い物（もち・追加の提灯）は build 時に `visible=false` で作っておき、異変で表示する。`desc` は画面に出さない
6. **Game** — `phase: title | intro | play | ended`。最初は異変なしの「いつもの通路」（判定なし）。`onEnd(dir)` が判定→フェード→`nextScene()`。
   **`G.busy` で二重判定を防ぐ**: フェード中もプレイヤーは端を越えたままなので、ガードが無いと毎フレーム判定され出口番号が 7, 14 と跳ねた（実際に踏んだ）。
   出題 `pickAnomaly` は 2D 版と同じ（出現率 0.6・一巡まで再出題なし・round で level 解放）
7. **入力** — PC: WASD/矢印＋マウス（クリックで Pointer Lock）。スマホ: 画面左半分=仮想スティック、右半分=見回し（pointerId で指を追跡）
8. **ループ** — 移動・足音・トリガー判定・描画

画面のトースト（`#toast`）は**1行で収める**（`white-space:nowrap`・文言は短く。ユーザー指摘で2行折り返しを禁止。2026-11-01）。
タイトルはロゴ＋はじめる＋あそびかたのみ（ユーザーの固定方針）。

## `window.__exit3d` — テストフック

`startGame()` / `skipIntro()` / `debugState()` / `forceAnomaly(id|null)`（次の nextScene で1回だけ）/ `teleport(z, x)` / `setYaw(y)` /
`renderOnly(id, yaw, z)`（判定に触れず、その異変だけの画像を返す）/ `anomalies`。
通しテスト: 各ラウンドで `debugState().anomaly` を見て、異変ありなら `teleport(12.3)`、なしなら `teleport(-11.9)`。各手の後 900ms 待つ。

## 異変の視認性（未解決・第2段階）

2D 版の「基準画像との差分 80px 以上」は 3D ではそのまま使えない。入口からの1枚では、ポスター・提灯・出口番号などの小物は
画面の 0.1% 前後しか変わらない（近づけば見える）。**物のそばを通る視点（位置×首振り）で測る**検証に作り直すこと。
測定スクリプトの考え方: 視点ごとに基準と異変をその場で続けて描画して比べる（基準を先にまとめて描くと値が狂った）。

## Scope（プロトタイプ段階）

通路1種・異変8種。図鑑・自己ベスト・共有・PlayCounts・テーマ追加・BGM は第2段階以降。mini-medley には組み込まない
（3D は短時間ミニ版に向かないため）。
