# This is NOT the Next.js you know

Next.js 16.x は API・規約・ファイル構造が大幅に変更されています。
コードを書く前に `node_modules/next/dist/docs/` の関連ガイドを必ず参照すること。
フォールバック: https://nextjs.org/docs
deprecation 警告を無視しないこと。

# 技術スタック

- Next.js 16.2.2 (App Router)
- React 19
- TypeScript
- Tailwind CSS v4
- Prisma ORM (client output: src/generated/prisma)
- PostgreSQL 16 (Docker)
- NextAuth.js

# プロジェクト構造

```
src/app/           Next.js App Router のページ・レイアウト
prisma/            スキーマ・マイグレーション
docker/            Docker 設定 (app, db)
docs/              設計ドキュメント
src/generated/     Prisma 生成コード (git 管理外)
```

# Docker 環境

サービス:
- `app` — Next.js アプリ (port 3000, Prisma Studio 5555)
- `db` — PostgreSQL 16 (port 5433)

```bash
docker-compose up -d        # 起動
docker-compose down         # 停止
docker-compose logs -f app  # ログ確認
```

環境変数は `.env.local` で管理 (git 管理外)。
必要な変数: DATABASE_URL, NEXTAUTH_SECRET, NEXTAUTH_URL, POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB

# Prisma

```bash
npx prisma generate          # クライアント再生成 (スキーマ変更後)
npx prisma migrate dev       # マイグレーション作成・適用
npx prisma studio            # GUI (port 5555)
```

スキーマ: `prisma/schema.prisma`
クライアント出力先: `src/generated/prisma`

# コーディング規約

- コンポーネント: PascalCase (`UserCard.tsx`)
- ファイル名: kebab-case (`user-card.tsx`)
- Prisma 操作: `src/lib/` に集約する
- 環境変数: `.env.local` のみ (コードにハードコードしない)
- Server Component がデフォルト。クライアント側が必要なときだけ `'use client'` を使う
