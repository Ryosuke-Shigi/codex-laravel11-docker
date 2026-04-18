# Laravel 11 Docker 開発環境

Laravel 11 向けに、責務を分離した Docker Compose 構成です。

- `app`: PHP-FPM 常駐コンテナ
- `nginx`: Web サーバー
- `db`: MySQL 8.4
- `redis`: Redis
- `mailpit`: ローカルメール確認
- `composer` / `artisan` / `npm`: **実行専用サービス**（`docker compose run --rm ...` 前提）

## ディレクトリ構成

```txt
.
├── compose.yml
├── .env.example
├── docker
│   ├── app
│   │   └── Dockerfile
│   └── nginx
│       └── default.conf
└── src
    └── (Laravel アプリ本体を配置)
```

- `src/` に Laravel アプリ本体（Git 管理対象）を置きます。
- Docker 関連設定は `docker/` と `compose.yml` に分離しています。

## 初回セットアップ手順

1. 環境変数ファイルを作成

```bash
cp .env.example .env
```

2. 常駐コンテナを起動（app / nginx / db / redis / mailpit）

```bash
docker compose up -d
```

3. `src` へ Laravel プロジェクトを作成（未作成の場合）

```bash
docker compose run --rm composer create-project laravel/laravel . "^11.0"
```

4. Laravel 側 `.env` を作成

```bash
cp src/.env.example src/.env
```

5. `src/.env` の主要値を確認（Redis / Mailpit を利用）

```dotenv
APP_URL=http://localhost:8080
DB_HOST=db
REDIS_HOST=redis
QUEUE_CONNECTION=redis
CACHE_STORE=redis
SESSION_DRIVER=redis
MAIL_MAILER=smtp
MAIL_HOST=mailpit
MAIL_PORT=1025
```

6. アプリキー生成

```bash
docker compose run --rm artisan key:generate
```

## Laravel プロジェクト作成手順

既存 `src` が空の場合のみ実行します。

```bash
docker compose run --rm composer create-project laravel/laravel . "^11.0"
```

## 日常利用コマンド

### composer install 実行方法

```bash
docker compose run --rm composer install
```

### artisan 実行方法

```bash
docker compose run --rm artisan --version
docker compose run --rm artisan route:list
```

### npm install / build / dev 実行方法

```bash
docker compose run --rm npm install
docker compose run --rm npm run build
docker compose run --rm npm run dev
```

> `npm run dev` は監視用途なので、必要に応じて `--service-ports` を追加してください。
>
> 例: `docker compose run --rm --service-ports npm run dev -- --host`

### migration 実行方法

```bash
docker compose run --rm artisan migrate
docker compose run --rm artisan migrate:fresh --seed
```

## Mailpit の確認 URL

- Web UI: `http://localhost:8025`
- SMTP: `localhost:1025`

Laravel から送信したメールは Mailpit UI で確認できます。

## Redis を使う前提の説明

この構成は将来的な queue 利用を見据えて Redis 前提にしています。

- `QUEUE_CONNECTION=redis`
- `CACHE_STORE=redis`
- `SESSION_DRIVER=redis`
- `REDIS_CLIENT=phpredis`（app コンテナに `phpredis` 拡張を導入済み）

キューを使う場合は、将来的に `queue` サービス（`php artisan queue:work`）を追加しやすい構造です。

## コンテナ構成の役割説明

- **app (PHP-FPM)**
  - Laravel 実行基盤。
  - nginx から FastCGI 経由で利用されます。
- **nginx**
  - HTTP 入口。
  - `public/` 配下を配信し、PHP リクエストを app に中継します。
- **db (MySQL)**
  - Laravel 用 RDB。
  - データは `db-data` ボリュームに永続化します。
- **redis**
  - キャッシュ / セッション / キュー基盤。
  - データは `redis-data` ボリュームに永続化します。
- **mailpit**
  - ローカルメール受信・確認用。
- **composer / artisan / npm（実行専用）**
  - 常駐しません。
  - `docker compose run --rm ...` 実行時のみ起動し、終了後に停止します。

## 停止 / 後片付け

```bash
docker compose down
docker compose down -v
```

- `down -v` は DB/Redis ボリュームも削除します。
