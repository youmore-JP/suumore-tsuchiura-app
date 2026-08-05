# AGENTS.md — SUUMORE TSUCHIURA 電子記録 PWA 運用ガイド

このファイルは opencode が次回このアプリを更新するときに自動で読む手順書です。

## プロジェクト概要

- アプリ: 共同生活援助(グループホーム)「SUUMORE TSUCHIURA」電子記録
- 実体: 単一ファイル `index.html` + `manifest.json` + `sw.js` + `icon-192.png` + `icon-512.png`
- データ: **すべて `localStorage` に保存**(サーバー不要・端末内保存)
  - `RECORDS_KEY`=打刻/記録, `DAILY_KEY`=日誌, `INCIDENT_KEY`=事故報告, `STAFF_KEY`=名簿, `KASAN_KEY`=加算, `SUMMARY_KEY`=まとめ, `SUPPORT_KEY`=支援, `USER_KEY`=利用者, `ADMIN_PW_KEY`=管理者パスワード
  - GitHub に上がるのはプログラムのみ。実データは一切含まれない
- 運用: **GitHub Pages**(無料・常時HTTPS)で公開 → グループホームのタブレットでPWAインストール → オフライン動作
- インストール済みタブレットは、最初の1回読み込み以降は完全オフラインで動く(Service Workerキャッシュ)

## 更新フロー(毎回必ずこの順で)

1. ファイル編集(`index.html` 等)
2. `git add -A && git commit -m "変更内容の説明"`
3. `git push`(DNS不調に注意、下記参照)
4. GitHub Pages のビルド完了を待つ: `gh api repos/youmore-JP/suumore-tsuchiura-app/pages --jq '.status'` が `built` になるまで(10秒間隔で最大60秒)
5. **sw.js のキャッシュバージョンを上げる**: `var CACHE = "sumore-cache-vN"` のNを+1して commit+push(ステップ4を再度)
   - 必須。これがないとタブレットが古いキャッシュを見続ける
6. 実ブラウザ(Playwright)で動作確認 + オフライン確認

## 重要な注意点

- **DNS不安定**: このWSL環境は `github.com` の名前解決が頻繁に失敗する。`git push` が `Could not resolve host` で失敗したら、以下で回避(実証済み):
  ```
  GIT_SSL_NO_VERIFY=true git -c http.extraHeader="Host: github.com" push https://youmore-JP:$(gh auth token)@20.27.177.113/youmore-JP/suumore-tsuchiura-app.git main
  ```
  コミット前に必ずリモートと同期確認すること(`git ls-remote origin`)
- **アイコン名**: manifest.json / sw.js / index.html は `icon-192.png`・`icon-512.png` を参照。実ファイルも同名(リネームしないこと。旧名 `suumore-icon-*.png` は不整合の元)
- **タブレットのキャッシュ**: SWは stale-while-revalidate。キャッシュバージョンを上げないと更新が届かない。タブレット側でアイコン再インストール不要(データは消えない)
- **localStorage は URL(オリジン)ごと**: データは https://youmore-jp.github.io/suumore-tsuchiura-app/ に紐付く。URLを変えるとデータが見えなくなるので URL は絶対に変えない
- データの保存キー名や形式を変えると古いデータが読めなくなるので、後方互換を保つこと
- 記録データの操作はしない(修正はindex.htmlのUI/ロジックのみ)

## GitHub 情報

- リポジトリ: https://github.com/youmore-JP/suumore-tsuchiura-app (公開)
- 公開URL: https://youmore-jp.github.io/suumore-tsuchiura-app/
- gh CLI 認証済み(youmore-JP)。Playwright MCP 利用可能

## 画面構成

- `screen-main`: メイン(打刻/記録)
- `screen-manage-entry`: 管理者パスワード入力 → `screen-manage-menu`(管理者メニュー)
- `screen-daily`: 業務日誌 / `screen-records`: 実績 / `screen-summary`: まとめ
- `screen-incident` / `screen-incident-form`: 事故報告
- `screen-kasan`: 加算 / `screen-support`: 支援 / `screen-user`: 利用者 / `screen-staff`: 職員
- `screen-export`: PDF出力 / `screen-backup`: データバックアップ / `screen-password`: パスワード変更

## 動作確認方法(Playwright)

- ローカル確認: `cd /home/cocololo/suumore-tsuchiura-app && python3 -m http.server 8123` → http://localhost:8123/index.html
- 本番確認: https://youmore-jp.github.io/suumore-tsuchiura-app/
- キャッシュを消す時: `caches.keys()` を全削除 + `serviceWorker.getRegistrations()` を unregister してから再読み込み
- オフライン確認: `context.setOffline(true)` で再読み込み → 表示されること
- 検証後は必ずテストデータを localStorage から削除すること
