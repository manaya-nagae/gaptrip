# Prisma ルール

## クエリの配置

- Prisma クライアントの呼び出しは `src/lib/db/` に集約する
- コンポーネント・page.tsx に直接 `prisma.xxx` を書かない
- 関数名はドメイン単位で整理する（`getUserById`, `createUser` など）

```
src/lib/db/
  user.ts      ← User 関連のクエリ
  index.ts     ← prisma client のインスタンス
```

```ts
// src/lib/db/index.ts
import { PrismaClient } from '@/generated/prisma'

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma
```

## N+1 を避ける

- 関連データは `include` または `select` で一括取得する

```ts
// ✅ 一括取得
const users = await prisma.user.findMany({
  include: { posts: true }
})

// ❌ ループ内でクエリ
const users = await prisma.user.findMany()
for (const user of users) {
  const posts = await prisma.post.findMany({ where: { userId: user.id } })
}
```

## select で必要なフィールドだけ取得

```ts
// ✅ パスワードを返さない
const user = await prisma.user.findUnique({
  where: { id },
  select: { id: true, email: true, name: true }
})
```

## トランザクション

複数テーブルを同時に更新するときは `$transaction` を使う。

```ts
await prisma.$transaction([
  prisma.user.update({ where: { id }, data: { name } }),
  prisma.log.create({ data: { userId: id, action: 'update' } })
])
```

## マイグレーション手順

1. `prisma/schema.prisma` を編集
2. `npx prisma format` — フォーマット確認
3. `npx prisma migrate dev --name <変更内容>` — マイグレーション作成・適用
4. `npx prisma generate` — クライアント再生成（自動で走ることも多い）
