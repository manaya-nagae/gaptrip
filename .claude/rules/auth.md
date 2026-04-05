# 認証ルール (NextAuth.js)

## セッション取得

**Server Component** では `auth()` を使う：
```ts
import { auth } from '@/auth'

export default async function Page() {
  const session = await auth()
  if (!session) redirect('/login')
  // ...
}
```

**Client Component** では `useSession()` を使う：
```ts
'use client'
import { useSession } from 'next-auth/react'

export function UserMenu() {
  const { data: session } = useSession()
  // ...
}
```

## 保護されたルート

- Server Component でセッションチェック → 未認証なら `redirect('/login')`
- middleware.ts で一括保護する方法もある（ルートが増えたら検討）

## セキュリティ原則

- **サーバーサイドで必ず検証する**。クライアントのセッション状態だけを信頼しない
- パスワードは bcrypt でハッシュ化（生パスワードを DB に保存しない）
- `NEXTAUTH_SECRET` は強力なランダム文字列を使う（`openssl rand -base64 32`）
- セッショントークンをログに出力しない

## 環境変数

```
NEXTAUTH_SECRET=...     # 必須・サーバーのみ
NEXTAUTH_URL=...        # 本番 URL
```

`NEXT_PUBLIC_` を絶対につけない（クライアントに露出する）。

## やってはいけないこと

- クライアント側のセッション確認だけでアクセス制御する
- `session.user.id` を信頼してサーバー側で再検証しない
- 認証エラーを握りつぶして `null` を返す
