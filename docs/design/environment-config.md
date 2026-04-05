# GapTrip 環境構成

## Docker Compose 構成

### サービス構成

| サービス名 | イメージ | ポート | 役割 |
|-----------|---------|--------|------|
| app | Dockerfile（Node.js） | 3000:3000 | Next.jsアプリケーション |
| db | postgres:16-alpine | 5432:5432 | PostgreSQL |

### ボリューム / マウント

| 対象 | 方式 | 目的 |
|------|------|------|
| PostgreSQLデータ | Docker volume | DB永続化 |
| `public/uploads/` | bind mount | 画像ファイル永続化。ホスト側でファイルを直接確認・バックアップできる。コンテナ再作成後も画像が消えない |

> uploads の永続化方式は **bind mount** を採用（個人ローカル開発環境での利便性を優先）。

---

## 環境変数

NextAuth v5 では変数名が v4 から変わっている。`NEXTAUTH_*` ではなく `AUTH_*` を使用すること。

### `.env.local`（ローカル開発用、gitignore対象）

```env
# DB接続文字列
# Docker Compose経由の場合はサービス名 "db" を使用
DATABASE_URL="postgresql://gaptrip:password@db:5432/gaptrip"

# NextAuth v5 設定
# AUTH_SECRET は openssl rand -base64 32 等で生成
AUTH_SECRET="your-secret-here"
AUTH_URL="http://localhost:3000"
```

### `.env.example`（テンプレート、git管理対象）

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE"
AUTH_SECRET="your-secret-here"
AUTH_URL="http://localhost:3000"
```

### 環境変数の説明

| 変数名 | 必須 | 説明 |
|--------|------|------|
| `DATABASE_URL` | Yes | PrismaのDB接続文字列。Docker Compose経由ならホストは `db` |
| `AUTH_SECRET` | Yes | JWTの署名・暗号化キー。32文字以上のランダム文字列。NextAuth v5の正式名（v4の `NEXTAUTH_SECRET` は使用しないこと） |
| `AUTH_URL` | Yes | アプリのベースURL。NextAuth v5の正式名（v4の `NEXTAUTH_URL` は使用しないこと） |

---

## Prisma設定

### schema.prisma のデータソース

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

### マイグレーション運用

| 操作 | コマンド |
|------|---------|
| マイグレーション作成・適用 | `npx prisma migrate dev --name <名前>` |
| 本番適用 | `npx prisma migrate deploy` |
| シードデータ投入 | `npx prisma db seed` |
| Prisma Client再生成 | `npx prisma generate` |

---

## セットアップ手順（初回）

1. `.env.local` を `.env.example` をもとに作成
2. `Dockerfile` に `RUN mkdir -p /app/public/uploads` を含めること（bind mount のマウント先ディレクトリが存在しないとコンテナ起動時にエラーになる）
3. `docker compose up -d` でコンテナ起動
4. `npx prisma migrate dev` でDB初期化
5. `npx prisma db seed` でカテゴリシードデータ投入
6. `npm run dev` で開発サーバー起動（または `docker compose` 内で起動）
