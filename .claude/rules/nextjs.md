# Next.js 16 App Router ルール

## コンポーネント設計

- **Server Component がデフォルト**。`use client` は最小限にする
- `use client` が必要なのは: ブラウザ API・イベントハンドラ・useState/useEffect を使うとき
- `use client` はツリーの末端（葉）に置く。親を汚染しない

```
app/
  page.tsx          ← Server Component（データ取得）
  components/
    UserList.tsx    ← Server Component（表示）
    LikeButton.tsx  ← Client Component（インタラクション）
```

## データ取得

- Server Component 内で直接 Prisma を呼ぶか `fetch` する
- `useEffect` でデータ取得しない
- `loading.tsx` を置いて Suspense を活用する

```ts
// ✅ Server Component で直接取得
export default async function Page() {
  const users = await prisma.user.findMany()
  return <UserList users={users} />
}

// ❌ Client Component で useEffect 取得
```

## ファイル規約

| ファイル | 用途 |
|---------|------|
| `page.tsx` | ルートのページ |
| `layout.tsx` | 共有レイアウト |
| `loading.tsx` | Suspense フォールバック |
| `error.tsx` | エラーバウンダリ |
| `not-found.tsx` | 404 |

## Server Actions

- フォーム送信・データ変更は Server Actions で行う
- ファイル先頭に `'use server'` を付けるか、関数に付ける
- バリデーションは Server Actions 内で行う（クライアント信頼しない）

```ts
// src/app/actions/user.ts
'use server'

export async function createUser(formData: FormData) {
  const email = formData.get('email') as string
  // バリデーション → Prisma → revalidatePath
}
```

## 参照

詳細: `node_modules/next/dist/docs/01-app/`
