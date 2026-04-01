# GapTrip

旅行記録アプリ。日本地図で都道府県の制覇率を確認し、行きたい場所(妄想)と行った場所(現実)のギャップを記録する。

## 技術スタック

- Next.js 15 (App Router) / TypeScript / Tailwind CSS
- Prisma 7 (ORM) / PostgreSQL 16
- NextAuth.js (認証)
- Docker Compose (開発環境)

## 前提条件

- Docker Desktop がインストール済みであること
- Git がインストール済みであること

## セットアップ

```bash
# 1. リポジトリをクローン
git clone https://github.com/okomemoti/gaptrip.git
cd gaptrip

# 2. 環境変数ファイルを作成
cp .env.example .env

# 3. NEXTAUTH_SECRET を生成して .env に設定
openssl rand -base64 32
# 出力された文字列で .env の NEXTAUTH_SECRET= を置き換える

# 4. Docker コンテナを起動
docker compose up -d --build

# 5. データベースのマイグレーション実行
docker compose exec app npx prisma migrate dev
```

セットアップ完了後、http://localhost:3000 にアクセスできます。

## 開発コマンド

```bash
# コンテナ起動
docker compose up -d

# コンテナ起動(リビルドあり)
docker compose up -d --build

# ログ確認
docker compose logs -f app

# Prisma Studio (DBのGUI管理画面)
docker compose exec app npx prisma studio --port 5555 --browser none
# http://localhost:5555 でアクセス

# Prisma マイグレーション作成・実行
docker compose exec app npx prisma migrate dev --name <マイグレーション名>

# コンテナ停止
docker compose down

# コンテナ停止 + データ削除(DBリセット)
docker compose down -v
```

## ポート一覧

| ポート | 用途 |
|---|---|
| 3000 | Next.js アプリケーション |
| 5433 | PostgreSQL (ホスト側からの接続用) |
| 5555 | Prisma Studio (DB管理GUI) |
