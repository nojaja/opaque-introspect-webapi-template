# 📄 **GitHub Copilot 実装指示書（Opaque Token + Introspection 認証対応 WebAPI + SPA/MCP 共通アクセス）**

## 🎯 **目的**

Hydra（ORY Hydra）を OAuth2 / OIDC Provider として利用し、
**Opaque Access Token + Token Introspection** に基づいて SPA と MCP の両方が共通の WebAPI を利用できる構成を Node.js（TypeScript）で実装する。

実行環境は Docker Compose を利用し、以下の構成を作る：

* **Hydra**（OAuth2/OIDC Provider）
* **Postgres**（Hydra データストア）
* **Identity UI**（Hydra Login/Consent 代理 UI）
* **SPA Client**（認可コード + PKCE、Vue3/Pinia）
* **MCP Client**（Client Credentials で CLI 実行）
* **WebAPI（Resource Server）** … SPA と MCP 双方が共通アクセス
* **Adminer**（必要に応じて DB 管理 UI）

WebAPI は以下の認証方法を持つ：

* Bearer Opaque Token を受け取り
* Hydra の `/oauth2/introspect` API へ問い合わせ
* Token の有効性・スコープ・クライアントIDを検証
* 有効であれば API を実行

構成情報は `config/config.json` を共有して管理し、CLI や各サービスは `CONFIG_PATH` 経由で同一値を参照する。

**⚠️ ホスト名の役割分離に注意**：ブラウザ経由でアクセスされる URL（例: `http://localhost:4444`）と、Compose ネットワーク内で参照する URL（例: `http://hydra:4444`）を必ず区別し、`config.json` には **external/internal** の両方を定義する。Hydra 設定ファイル（`hydra/config.yaml`）では、Issuer / Login / Consent URL をブラウザ視点（`localhost`）に合わせないと `invalid_client` やリダイレクトループが発生するため、今回の仕様では最初から `localhost` に統一する。

---

# 📦 **システム構成（ディレクトリ）**

```
/
├─ docker-compose.yml
├─ config/
│   └─ config.json
├─ hydra/
│   ├─ config.yaml
│   └─ migrations/
├─ identity/
│   ├─ Dockerfile
│   └─ src/
├─ webapi/
│   ├─ Dockerfile
│   ├─ src/
│   │   ├─ server.ts
│   │   ├─ middlewares/auth.ts
│   │   ├─ services/introspect.ts
│   │   ├─ services/clientRegistry.ts
│   │   └─ routes/
│   │       └─ example.ts
│   ├─ scripts/mcp-client.ts
│   ├─ package.json
│   └─ tsconfig.json
├─ spa/
│   ├─ Dockerfile
│   ├─ src/
│   └─ scripts/copy-config.mjs
└─ spec/
    └─ spec.md
```

---

# 🧱 **要求仕様（Copilot が生成すべき内容）**

## 1. Docker Compose 環境の構築

Copilot は `docker-compose.yml` を生成すること。

* `postgres`（データストア、ボリューム `postgres-data`）
* `hydra-migrate`（マイグレーション用）
* `hydra`（oryd/hydra:latest）
* `identity`（Hydra Login/Consent UI）
* `webapi`（node:18-alpine）
* `spa`（Vue3 + webpack dev server）
* `adminer`（DB 管理 UI）

各サービスは同一ネットワーク上で稼働し、`config/config.json` をボリュームマウントして設定を共有する。必要に応じて環境変数で上書きできるようにする。

### 1.1 サービス間依存

* `hydra-migrate` は `postgres` 完了後に `oryd/hydra migrate sql --yes` を実行する。
* `hydra` は `hydra-migrate` に依存し、`hydra/config.yaml` を `/etc/config/hydra.yaml` にマウントして `serve all --dev` を実行する。
* `identity` は `hydra` へ依存し、Express を `PORT=3000` で起動する。
* `webapi` は `hydra` へ依存し、`PORT=3001`、`CORS_ORIGIN=http://localhost:4173` を想定。起動時に Hydra クライアント登録を自動化する。
* `spa` は `hydra` と `identity` に依存し、起動前に `scripts/copy-config.mjs` で `app-config.json` を生成してから `npm run start -- --host 0.0.0.0 --port 4173` を実行する。
* `adminer` は `postgres` に依存し `8080` で UI を提供する。

#### Hydra コンテナ起動時の注意

- `oryd/hydra` イメージはエントリポイントが既に `hydra` になっている。Compose の `command` で `hydra serve all ...` のようにバイナリ名を重ねると `unknown command "hydra"` が発生するため、`command: ["serve", "all", ...]` のようにサブコマンドのみを渡すこと。
- v25 以降は `oauth2.skip_jwt_bearer_tokens` が無効プロパティとして拒否される。`hydra/config.yaml` に同プロパティを残さず、必要なら別手段で挙動を調整する。

### 1.2 ポートおよびホストアクセス

| サービス | ホストポート | コンテナポート |
| --- | --- | --- |
| Postgres | 5432 | 5432 |
| Hydra Public/Admin | 4444 / 4445 | 4444 / 4445 |
| Identity UI | 3000 | 3000 |
| WebAPI | 3001 | 3001 |
| SPA | 4173 | 4173 |
| Adminer | 8080 | 8080 |

### 1.3 共有設定ファイル `config/config.json`

```jsonc
{
  "hydra": {
    "auth": { "external": "http://localhost:4444/oauth2/auth" },
    "token": {
      "external": "http://localhost:4444/oauth2/token",
      "internal": "http://hydra:4444/oauth2/token"
    },
    "admin": {
      "external": "http://localhost:4445",
      "internal": "http://hydra:4445"
    },
    "introspection": {
      "internal": "http://hydra:4445/oauth2/introspect"
    },
    "userinfo": {
      "internal": "http://hydra:4444/userinfo"
    },
    "callbackUrl": "http://localhost:4173/callback",
    "scope": ["openid", "read"],
    "spaOrigin": "http://localhost:4173"
  },
  "clients": {
    "spa": { "clientId": "spa-client" },
    "mcp": { "clientId": "mcp-client", "clientSecret": "mcp-secret", "scope": "read" },
    "introspection": { "clientId": "introspection-client", "clientSecret": "introspection-secret" }
  },
  "webapi": {
    "externalBaseUrl": "http://localhost:3001",
    "internalBaseUrl": "http://webapi:3001",
    "requiredScope": "read"
  }
}
```

* すべての Node プロセスは `CONFIG_PATH`（例: `/config/config.json`）で上記ファイルを参照する。
* `.env` やサービス固有の環境変数で値を上書きできるようにしておく。

### 1.4 Hydra の設定例（config.yaml）

```yaml
serve:
  public:
    port: 4444
    cors:
      enabled: true
      allowed_origins:
        - http://localhost:4173
      allowed_methods:
        - POST
        - GET
      allowed_headers:
        - Authorization
        - Content-Type
  admin:
    port: 4445

urls:
  self:
    issuer: http://localhost:4444/
  consent: http://localhost:3000/consent
  login: http://localhost:3000/login

oauth2:
  expose_internal_errors: true
```

* Issuer / Login / Consent URL を **ブラウザ視点の `localhost`** にしておくことで、Hydra からのリダイレクト先が SPA/Identity の実際のホストと一致し、`invalid_client`/`invalid_state` を防げる。
* CORS で SPA のオリジンを許可し、`/oauth2/token` をブラウザから直接叩けるようにしておく。
* 内部アクセスは `config.json` の `internal` URL を利用する（例: WebAPI → Hydra Introspection）。
* **注意**：Hydra Admin (`http://localhost:4445`) は管理プレーン用 API であり、ブラウザから直接呼び出さない。Identity サービスなどバックエンドから内部的にアクセスし、必要な情報だけを同一オリジン API として公開する。

---

## 2. WebAPI（Node.js + TypeScript）の実装仕様

### 必須 npm modules

```
express
axios
dotenv
body-parser
cors (CORS_ORIGIN 利用時)
```

TypeScript:

```
typescript
ts-node
@types/express
@types/node
```

---

## 3. WebAPI の実装要件

> **ビルド出力パスの取り扱い**
>
> TypeScript の `rootDir` はプロジェクトルート (`.`)、`outDir` は `dist`（デフォルト構成）とし、`src/server.ts` など `src/` 配下のファイルは `dist/src/server.js` として出力される前提に統一する。このため、`npm start` や Dockerfile の `CMD` では必ず `node dist/src/server.js` を実行すること。`dist/server.js` を参照すると `MODULE_NOT_FOUND: /app/dist/server.js` でコンテナが即終了するため、`tsconfig.json` を変更する場合は **出力パス・起動コマンド・COPY 対象** をセットで更新し、整合性を崩さないようにする。

### ▶ **3.1 Express サーバ**

`webapi/src/server.ts`

* `.env` と `config/config.json` を読み込み、設定値を優先順位付きで解決する。
* `Authorization: Bearer <token>` を受け取る API 群に対し認証ミドルウェアを適用する。
* `cors` は `CORS_ORIGIN` が設定されている場合のみ許可する。
* `body-parser.json` / `body-parser.urlencoded` を登録し、`/healthz` を `{ status: 'ok' }` で応答させる。
* `/api` プレフィックスに `exampleRouter` をマウントする。
* 起動時に `ensureHydraClients()` を実行して SPA/MCP/Introspection クライアントを Hydra Admin API に upsert し、失敗時はプロセスを終了させる。

---

### ▶ **3.2 認証ミドルウェア**

`webapi/src/middlewares/auth.ts`

責務：

* Authorization header から bearer token を取り出す
* Hydra Admin API の `/oauth2/introspect` に問い合わせ
* レスポンス例：

```json
{
  "active": true,
  "sub": "user-id",
  "client_id": "client",
  "scope": "read write"
}
```

* `active=false` の場合 → 401 (`トークンが無効です` をログ)
* scope が不足している場合 → 403 (`必要なスコープが不足しています` をログ)
* `req.user` に introspection の結果を保存し、`src/types/express.d.ts` で型拡張する
* 想定外のエラーは `console.error` して 500 を返却する

---

### ▶ **3.3 Hydra Introspection 呼び出し**

`webapi/src/services/introspect.ts`

仕様：

* Hydra の Admin API（`http://hydra:4445/oauth2/introspect`）へ POST
* `Content-Type: application/x-www-form-urlencoded`
* client authentication は Basic 認証を使用（Hydra の client_secret を利用）
* `HYDRA_CLIENT_ID` / `HYDRA_CLIENT_SECRET` / `HYDRA_INTROSPECTION_URL` で上書き可能
* タイムアウトは 5 秒程度に設定し、失敗時は `Error('Hydraイントロスペクションに失敗しました')` を throw

例：

```ts
export async function introspect(token: string) {
  const res = await axios.post(
    "http://hydra:4445/oauth2/introspect",
    new URLSearchParams({ token }),
    {
      auth: {
        username: process.env.HYDRA_CLIENT_ID!,
        password: process.env.HYDRA_CLIENT_SECRET!
      },
      timeout: 5000
    }
  );
  return res.data;
}
```

---
        # The following line has been removed as it is no longer supported in v25 and above
## 4. ルーティング例

`webapi/src/routes/example.ts`

仕様：

### `/api/hello`

* 認証必須
* scope: `read` が必要 (`config.webapi.requiredScope` から取得)
* レスポンス例：

```json
{
  "message": "hello",
  "subject": req.user.sub,
  "client": req.user.client_id,
  "scope": req.user.scope
}
```

---

## 5. 簡易クライアント例（MCP）

Copilot は次のファイルも生成してよい：

`webapi/scripts/mcp-client.ts`

* `client_credentials` で Hydra から access_token を取得
* WebAPI を呼び出す
* `.env` および `config.json` から Hydra Token Endpoint、MCP クレデンシャル、WebAPI URL を読み込む
* `obtainToken()` で `grant_type=client_credentials` の `application/x-www-form-urlencoded` POST を行い、取得した `access_token` をログ出力する
* `callApi(token)` で `GET {webapi}/api/hello` に Bearer ヘッダーを付与し、レスポンスを標準出力する
* エラー発生時は Axios エラー詳細を出力して `process.exit(1)` を呼び出す

---

## 6. Identity サービス (`identity/`)

* Vue3 + Pinia（webpack 5 構成）の SPA として Hydra の Login/Consent UI を提供する。
* バックエンドは Express でシンプルな API を公開し、Vue アプリは同一サーバから `main.bundle.js` を配信して `#app` にマウントする。
* `.env` を読み込んだ後、`CONFIG_PATH` から Hydra Admin の **internal URL** を読み込み、サーバー側で Axios クライアントを構築する。Hydra Admin はブラウザに公開しない（CORS ブロックを避けるため）。
* Express サーバーは `/identity-config`（SPA から参照する設定。Hydra Admin URL は含めない）に加え、`/api/hydra/login-request` / `/api/hydra/login-accept` / `/api/hydra/consent-request` / `/api/hydra/consent-accept` などの **プロキシ API** を同一オリジンで提供する。これらの API が Hydra Admin へ内向きアクセスを代行する。
* ハードコード済みユーザー（例: alice/bob）を Pinia ストアで管理し、ログインフォームは Vue の双方向バインディングで入力値を保持する。
* `/login` と `/consent` の画面は Vue Router でルーティングし、各ページの `onMounted` で **同一オリジンのプロキシ API** からチャレンジ情報を取得する。`skip=true` のケースでは自動 Accept を実施し、結果のリダイレクト先を `window.location.href` で遷移させる。
* フォーム送信時は常に Express のプロキシ API を経由し、サーバー側で Hydra Admin と通信して accept / reject を行う。ブラウザから `http://localhost:4445` に直接アクセスさせないこと。
* スタイルは `identity/src/assets/style.css` をエントリでインポートし、webpack でバンドルしたうえで `/assets` 配下に出力する。
* **ビルド時の注意**：`Dockerfile` では `npm run build` で生成した Vue バンドル一式と静的アセットを `dist/` にコピーし、Express は `dist/index.html` を常に返すよう設定する。これにより本番コンテナでも同一構成で動作する。

---

## 7. SPA (`spa/`)

* Vue3 + Pinia（webpack 5 構成）。`npm run build` で `dist/` を生成し、`npm run start -- --host 0.0.0.0 --port 4173` で webpack-dev-server を公開する。
* `scripts/copy-config.mjs` が `config/config.json` を `app-config.json` としてコピーし、ブラウザから取得できるようにする。
* `src/router/index.ts` は `/` と `/callback` を用意し、`HomePage.vue` が PKCE フローを完結させる。
* `src/stores/auth.ts` は `sessionStorage` に PKCE アーティファクトを保持し、`startLogin` / `handleCallback` / `callApi` / `logout` を提供する。`handleCallback` では `state` 不一致や Hydra から返る `error`/`error_description` を検出し、画面に日本語エラーを表示して次のログインを案内する。
* `src/services/appConfig.ts` が `APP_CONFIG_URL`（未指定時 `/app-config.json`）から設定をフェッチし、単一キャッシュを保持する。

---

## 8. Hydra クライアント登録 (`webapi/src/services/clientRegistry.ts`)

1. `ensureHydraClients`
   * `HYDRA_ADMIN_URL` → `config.hydra.admin.internal/external` の順に解決し、`/health/ready` を最大 30 回ポーリングして待機する。
2. `buildClientDefinitions`
   * SPA クライアント: `authorization_code + refresh_token`、`token_endpoint_auth_method: none`、`redirect_uris = [callbackUrl]`、`allowed_cors_origins = [spaOrigin]`。
   * MCP クライアント: `client_credentials`、`client_secret_post`、scope は `config.clients.mcp.scope`。
   * Introspection クライアント: `client_credentials`、scope `hydra.introspect`。
3. `upsertClient`
   * `PUT /clients/{client_id}` で更新し、404 の場合は `POST /clients` で作成する。
   * 空値は送信前に削除し、最新設定を必ず反映させる。

---

## 9. 動作シーケンス（Copilot は実装をこれに従わせる）

### 9.1 Docker 起動

1. `docker compose up --build` を実行。
2. `hydra-migrate` → `hydra` → `identity` / `webapi` / `spa` の順番でサービスが準備される。
3. `webapi` 起動時に Hydra 管理 API へ接続し、クライアントを upsert する。
4. SPA の `copy-config.mjs` が `config.json` を `app-config.json` として公開する。

### 9.2 SPA (Authorization Code + PKCE)

1. ユーザーが SPA を開き「ログイン開始」を押下。
2. `startLogin`
   * PKCE の verifier/state を生成し `sessionStorage` に保存。
   * Hydra Auth (`/oauth2/auth`) へ遷移。
3. Hydra → Identity UI
   * `/login` でユーザー認証、`/consent` で scope `read` を許可。
4. Hydra が `callbackUrl` へ `code` と `state` を付与してリダイレクト。
5. `handleCallback`
   * `state` を検証後、`token` エンドポイントに `authorization_code` + PKCE で POST。
   * 取得した `access_token` を保持し、アーティファクトを削除。
6. 「API 呼び出し」で WebAPI `/api/hello` を叩き、`req.user` を基にした JSON を表示する。

### 9.3 MCP (Client Credentials)

1. `docker compose exec webapi node ./dist/scripts/mcp-client.js` などで CLI を実行。
2. `obtainToken` が Hydra `/oauth2/token` に `grant_type=client_credentials` を送信する。
3. 取得したトークンで `/api/hello` を叩き、SPA と同じレスポンスにアクセスする。

---

## 10. 運用・確認ポイント

* ヘルスチェック: `http://localhost:3001/healthz`。
* Hydra 管理 API: `http://localhost:4445/clients`。
* Adminer: `http://localhost:8080` (System: PostgreSQL, Server: postgres, User/Pass: hydra/hydra)。
* SPA 設定が反映されない場合は `node spa/scripts/copy-config.mjs` を手動実行する。
* トークンが 401/403 の際は WebAPI ログの日本語メッセージで原因を確認できる。
* CORS エラーが出た場合は `hydra/config.yaml` の `serve.public.cors` 設定と `config.json` の `hydra.spaOrigin` が一致しているか確認する。

---

## 11. 参考：Hydra と WebAPI の連携フロー

1. SPA or MCP が access_token を取得
   * SPA：認可コード + PKCE → access_token（opaque）
   * MCP：client_credentials → access_token（opaque）
2. WebAPI にアクセス

```
Authorization: Bearer <opaque_access_token>
```

3. WebAPI で introspection 実行
   * API → Hydra：POST /oauth2/introspect
   * Hydra → `active=true` なら OK
4. API 実行
   * WebAPI はユーザー情報（sub, client_id, scope）を付与してレスポンス
