# E2Eテスト確認ポイント（Opaque Token + Introspection 構成）

本ドキュメントは、Hydra を用いた Opaque Token + Introspection 認証構成で WebAPI / SPA / MCP が共通アクセスするアプリを E2E 試験する際に、実装固有ではなく普遍的に確認すべき観点を整理したものです。

## 1. インフラ・ネットワーク
- [ ] Docker Compose 全サービスが起動し、Hydra Public/Admin・WebAPI・SPA・Identity UI へブラウザ/CLI から到達できる。
- [ ] `config/config.json` の external/internal URL が正しく参照され、SPA からの CORS エラーや WebAPI → Hydra の名前解決エラーが発生しない。
- [ ] Hydra Admin `/health/ready` が安定して 200 を返し、クライアント登録処理がリトライなしで完了する。

## 2. SPA（PKCE）フロー
- [ ] 「ログイン開始」で `state`・`code_verifier` が sessionStorage に格納され、Hydra `/oauth2/auth` に正しいクエリ（PKCE/S256）が付与されている。
- [ ] Identity UI で login → consent を経て `callbackUrl` に `code`／`state` が返却される。
- [ ] `state` の検証失敗や `error` パラメータ付与時に、SPA が日本語メッセージで異常を通知し再ログインを案内する。
- [ ] `/oauth2/token` から取得した `access_token` が Opaque 形式で保存され、API 呼び出し後にセッションの機密情報（verifier 等）が破棄される。
- [ ] ログアウトでアクセストークンと API レスポンスがクリアされ、再ログインなしに保護 API を叩けない。

## 3. WebAPI + Introspection
- [ ] `/api/hello` 等の保護エンドポイントは Bearer 未指定時に 401、範囲不足時に 403 を返し、日本語ログを出力する。
- [ ] Introspection エラー（Hydra への到達不可や 5xx）を 500 としてクライアントに返し、詳細はサーバーログへ記録する。
- [ ] `req.user` へ `sub`・`client_id`・`scope` が格納され、レスポンスや監査ログに反映される。
- [ ] Hydra クライアント登録処理（upsert）が重複実行でも整合性を保ち、設定差分を確実に反映する。

## 4. Identity UI
- [ ] `skip=true` のチャレンジで自動 Accept される一方、通常フローではログイン画面 → コンセント画面が必ず表示される。
- [ ] 誤認証時に 401 相当メッセージを出しつつ再入力できる。
- [ ] Consent 拒否時には Hydra へ `access_denied` が送られ、SPA 側がそのエラーを受け取れる。

## 5. MCP（Client Credentials）
- [ ] CLI が `grant_type=client_credentials` でトークンを取得し、`scope` が設定値と一致する。
- [ ] CLI から WebAPI を呼び出した結果が SPA からのレスポンスと同一構造になる。
- [ ] 失効したクレデンシャルや誤パラメータでトークン取得が失敗するケースをハンドリングできる。

## 6. セキュリティ・エラーハンドリング
- [ ] Hydra からのリダイレクト／トークン取得に TLS が要求される環境へ移行しても構成が破綻しない（`localhost` 固定値が残っていない）。
- [ ] 例外発生時に機密情報（client_secret 等）がログへ出力されない。
- [ ] 管理 API（4445）へのアクセスが外部公開されていない、もしくは必要最低限に制限されている。

## 7. 観察ポイント
- [ ] WebAPI・Hydra・Identity・SPA のログが時系列で追跡でき、リクエスト ID 等で突合できる。
- [ ] トークン失効やスコープ変更など運用イベント時に、再ログイン／再発行で期待通り動作が復旧する。

以上を満たすことで、Opaque Token + Introspection 構成の普遍的な品質を確保できる。
