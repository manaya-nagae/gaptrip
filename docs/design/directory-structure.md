# GapTrip ディレクトリ構成

Next.js App Router の規約に従った構成。

## ツリー

```
gap-trip/
├── middleware.ts                       # 未認証ユーザーを /login にリダイレクト（NextAuth v5）
├── app/
│   ├── (auth)/                        # 認証不要ルートグループ（ボトムナビ非表示）
│   │   ├── login/
│   │   │   └── page.tsx               # ログイン画面
│   │   └── register/
│   │       └── page.tsx               # ユーザー登録画面
│   ├── (main)/                        # 認証必要ルートグループ
│   │   ├── layout.tsx                 # ボトムナビゲーション共通レイアウト（認証チェックはmiddlewareに委譲）
│   │   ├── page.tsx                   # マップ画面（/）
│   │   ├── search/
│   │   │   └── page.tsx               # 探す画面（/search）※Client Component必須（検索入力にクライアントサイドJSが必要）
│   │   ├── places/
│   │   │   └── [id]/
│   │   │       └── page.tsx           # Place詳細（/places/[id]）
│   │   ├── spots/
│   │   │   ├── page.tsx               # スポット一覧（/spots）
│   │   │   └── [id]/
│   │   │       ├── page.tsx           # スポット詳細（/spots/[id]）
│   │   │       ├── edit/
│   │   │       │   └── page.tsx       # スポット編集（/spots/[id]/edit）
│   │   │       └── record/
│   │   │           └── page.tsx       # 現実記録（/spots/[id]/record）※Client Component必須（画像アップロード+公開トグルにクライアントサイドJSが必要）
│   │   └── profile/
│   │       └── page.tsx               # プロフィール（/profile）
│   ├── api/
│   │   ├── places/
│   │   │   ├── autocomplete/
│   │   │   │   └── route.ts           # GET: Google Places Autocomplete プロキシ
│   │   │   └── photo/
│   │   │       └── route.ts           # GET: Google Places 写真プロキシ
│   │   └── upload/
│   │       └── route.ts               # POST: 画像アップロード（削除はServer Actionが直接fs.unlinkで行う）
│   ├── layout.tsx                     # ルートレイアウト（html/body、NextAuthProvider）
│   └── globals.css
│
├── actions/                           # Server Actions
│   ├── auth.ts                        # login / register / logout
│   ├── spots.ts                       # navigateToPlace / addSpotFromPlace / updateSpot / deleteSpot / recordWent
│   └── reviews.ts                     # publishReview / unpublishReview
│
├── lib/
│   ├── auth.ts                        # NextAuth v5設定（auth / signIn / signOut / handlers をエクスポート）
│   ├── prisma.ts                      # Prismaシングルトン
│   ├── google-places.ts               # Google Places API呼び出し + resolvePlace（getOrCreate）
│   ├── queries.ts                     # Server Components用データ取得関数（Prismaクエリをまとめる）
│   └── constants/
│       └── prefectures.ts             # 都道府県コード定数
│
├── components/
│   ├── ui/                            # 汎用コンポーネント（Button, Card, Input等）
│   ├── layout/
│   │   ├── AppHeader.tsx              # 共通ヘッダー（ロゴ + ユーザーアバター）
│   │   └── BottomNav.tsx              # ボトムナビゲーション（4タブ: マップ/探す/一覧/プロフィール）
│   ├── map/
│   │   └── JapanMap.tsx               # 日本地図SVGコンポーネント
│   ├── search/
│   │   └── SearchBar.tsx              # Google Places検索バー + 候補リスト
│   ├── places/
│   │   ├── PlaceInfo.tsx              # Place情報表示
│   │   ├── ReviewList.tsx             # 公開口コミ一覧
│   │   └── WantForm.tsx               # 「行きたい！」登録フォーム（カテゴリ・期待度・メモ）
│   └── spots/
│       ├── SpotCard.tsx               # スポットカード（一覧用）
│       ├── SpotForm.tsx               # スポット編集フォーム
│       └── StarRating.tsx             # ★評価入力UI
│
├── prisma/
│   ├── schema.prisma                  # DBスキーマ定義
│   └── seed.ts                        # Categoryシードデータ
│
├── public/
│   └── uploads/                       # 画像ファイル保存先（Docker永続化対象）
│
├── .env.local                         # ローカル環境変数（gitignore対象）
├── .env.example                       # 環境変数テンプレート（git管理）
├── docker-compose.yml
├── Dockerfile
├── next.config.ts
├── tailwind.config.ts
└── tsconfig.json
```

## ディレクトリ別の役割

### `middleware.ts`

NextAuth v5 の `auth` をそのままmiddlewareとしてエクスポートし、未認証ユーザーを `/login` へリダイレクトする。`matcher` で `(main)` 配下のルートを対象にする。認証チェックはここに集約し、各layout.tsxやServer Componentでは行わない。

```typescript
// middleware.ts のイメージ（実装時に確定）
export { auth as middleware } from "@/lib/auth"
export const config = {
  // 完全一致で除外（login → loginx のような誤マッチを防ぐ）
  matcher: ["/((?!login$|register$|api/|_next/|favicon\\.ico$).*)"],
}
```

### `app/(auth)/` と `app/(main)/`

Route Groupsで認証要否によってレイアウトを分ける。
- `(auth)`: ボトムナビなし。middlewareの対象外。
- `(main)`: ボトムナビあり。middlewareが未認証アクセスを弾く。layout.tsxは純粋にレイアウト提供のみ。

### `actions/`

Server Actionsをまとめる。`"use server"` ディレクティブをファイル先頭に記述。
- `auth.ts`: ログイン・ユーザー登録・ログアウト
- `spots.ts`: Place遷移（navigateToPlace）・Place経由でのスポット作成（addSpotFromPlace）・更新・削除・WENT記録
- `reviews.ts`: 口コミ公開（publishReview）・非公開（unpublishReview）

### `lib/`

サーバーサイドで共用するユーティリティ。
- `auth.ts`: NextAuth v5の設定ファイル。`NextAuth()` から `auth`, `signIn`, `signOut`, `handlers` をエクスポートする。middleware.ts からは `auth` を、Server Actions からは `signIn` / `signOut` を、Route Handlers からは `auth` を import する。**注意:** デフォルトではセッションに `userId` が含まれない。`jwt` コールバックで `token.sub`（= userId）をトークンに含め、`session` コールバックで `session.user.id` に渡す設定が必要。
- `prisma.ts`: PrismaClientのシングルトン（開発時のホットリロード対策込み）。
- `google-places.ts`: Google Places API (New) の呼び出しラッパー。Autocomplete・Place Details の呼び出しと `resolvePlace`（googlePlaceIdからPlaceのgetOrCreate）を提供。`GOOGLE_PLACES_API_KEY` はこのファイル内でのみ参照する。
- `queries.ts`: Server Components用のデータ取得関数をまとめる。スポット一覧、スポット詳細、Place詳細、制覇率集計、カテゴリ一覧等。
- `constants/prefectures.ts`: 都道府県コード定数。

### `components/`

UIコンポーネント。基本的にClient Componentsとして定義し、データはServer ComponentsからPropsで渡す。

### `public/uploads/`

アップロード画像の保存先。Next.jsの `public/` 配下なのでそのままURLでアクセス可能（例: `/uploads/xxx.jpg`）。
Docker環境では bind mount で永続化する（`environment-config.md` 参照）。
