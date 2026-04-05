# GapTrip Docker構成 設計書

## 概要

GapTrip（旅行記録アプリ）のローカル開発環境をDocker化する。デプロイ先は未定のため、開発環境に特化した構成とする。

## 技術スタック

| レイヤー | 技術 | バージョン |
|---|---|---|
| フロントエンド / バックエンド | Next.js (App Router) | 最新安定版 |
| 言語 | TypeScript | 最新安定版 |
| ORM | Prisma | 最新安定版 |
| データベース | PostgreSQL | 16 |
| 認証 | NextAuth.js (Auth.js) | 最新安定版 |
| ランタイム | Node.js | 20 |

## コンテナ構成

2コンテナのシンプル構成。

| コンテナ名 | イメージ | ポート | 役割 |
|---|---|---|---|
| `app` | カスタムビルド (node:20-alpine) | `3000:3000`, `5555:5555` | Next.js dev server + Prisma CLI + Prisma Studio |
| `db` | postgres:16-alpine | `5433:5432` | PostgreSQL データベース |

ポート補足:
- `3000` — Next.js アプリケーション
- `5555` — Prisma Studio（ブラウザでDB確認用）
- `5433` — PostgreSQL（ホスト側。VSCode拡張等から接続用）

## ディレクトリ構成

```
gap-trip/
├── docker/
│   └── app/
│       └── Dockerfile      # app コンテナのビルド定義
├── docker-compose.yml
├── .dockerignore
├── .env                    # DB接続情報、NextAuth設定
├── docs/
│   └── basic_docs/         # 既存の設計ドキュメント
├── src/                    # Next.js アプリ (後で作成)
├── prisma/
│   └── schema.prisma       # Prisma スキーマ (後で作成)
├── package.json
└── tsconfig.json
```

## docker-compose.yml

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/app/Dockerfile
    ports:
      - "3000:3000"
      - "5555:5555"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - NEXTAUTH_URL=${NEXTAUTH_URL}
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
      - node_modules:/app/node_modules

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  node_modules:
  postgres_data:
```

### 設計判断

- **`node_modules` を Named Volume で分離**: macOS のバインドマウントはファイル数が多いと遅い。node_modules をコンテナ内 Volume にすることでパフォーマンスを確保する。
- **`healthcheck` で DB 起動待ち**: `depends_on` + `condition: service_healthy` により、PostgreSQL が接続可能になってから app コンテナが起動する。DB接続エラーによる起動失敗を防ぐ。
- **環境変数は `.env` で管理**: docker-compose.yml に直書きせず、`.env` ファイルで管理する。`.gitignore` に追加してリポジトリに含めない。

## Dockerfile (docker/app/Dockerfile)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npx prisma generate
EXPOSE 3000
CMD ["npm", "run", "dev"]
```

### 設計判断

- **`npm run build` は含めない**: 開発環境では dev server を使うため不要。ビルド時間を節約する。
- **`prisma generate` を実行**: Prisma Client をコンテナビルド時に生成する。これがないと Next.js から DB にアクセスできない。
- **alpine イメージ**: イメージサイズを最小化。

## .dockerignore

```
node_modules
.next
postgres_data
.git
*.md
docs/
```

## .env (テンプレート)

```
# Database
POSTGRES_USER=gap_user
POSTGRES_PASSWORD=gap_pass
POSTGRES_DB=gap_trip
DATABASE_URL=postgresql://gap_user:gap_pass@db:5432/gap_trip

# NextAuth
NEXTAUTH_SECRET=your-secret-key-here
NEXTAUTH_URL=http://localhost:3000
```

## 開発ワークフロー

```bash
# 初回起動
docker compose up -d --build

# Prisma マイグレーション実行
docker compose exec app npx prisma migrate dev

# Prisma Studio でDB確認 (localhost:5555)
docker compose exec app npx prisma studio

# ログ確認
docker compose logs -f app

# 停止
docker compose down

# DBデータも含めて完全リセット
docker compose down -v
```

## 将来の拡張ポイント

デプロイ先が決まった際に以下を追加検討:

- `docker-compose.prod.yml` の作成（本番用オーバーライド）
- マルチステージビルド（本番用 Dockerfile）
- Mailpit コンテナ（パスワードリセット等のメール機能実装時）
- Nginx リバースプロキシ（本番環境用）
