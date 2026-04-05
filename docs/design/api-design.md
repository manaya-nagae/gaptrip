# GapTrip API設計書

## 概要

本書はGapTrip MVPのサーバーサイド処理（Server Actions・Route Handler・データ取得）の仕様を定義する。

### 責務の分担

| 種別 | 用途 | 認証チェック |
|------|------|-------------|
| Server Components + Prisma | データ取得（一覧・詳細・集計） | セッション取得で確認 |
| Server Actions | データ更新（作成・編集・削除・WENT記録・口コミ公開・認証） | セッション取得で確認（認証系を除く） |
| Route Handler | Google Places APIプロキシ、ファイルアップロード | 自身で `auth()` を呼んで確認 |

**多層防御について:** middleware はエッジでの一次防衛であり、認証済みユーザーのリクエストのみを `(main)` 配下のページに通す。ただし **API routes (`/api/*`) は middleware の対象外** とし、Route Handler 内で自ら `auth()` を呼んで認証確認する。Server Actions でも必ずセッションから userId を取得して確認すること。

### Server Action のエラー状態型

全ての `useActionState` 対応 Server Action は以下の型を返す。

```typescript
type ActionState = {
  error?: string
} | null
```

- 初期状態: `null`
- バリデーションエラー / 業務エラー: `{ error: "メッセージ" }`
- 成功時: `redirect()` するため戻り値はクライアントに届かない

`deleteSpot` のように確認ダイアログ経由で呼ばれるアクションは `useActionState` を使わず、失敗時はトースト表示とする。

---

## 1. Route Handler

### 1.1 Google Places Autocomplete プロキシ

```
GET /api/places/autocomplete?input=xxx
```

**クエリパラメータ:**
| パラメータ | 必須 | 説明 |
|-----------|------|------|
| input | Yes | 検索文字列（最低2文字） |

**処理:**
1. `auth()` で認証確認（未認証なら401）
2. `input` バリデーション（空文字・1文字以下は400）
3. サーバーサイドで `GOOGLE_PLACES_API_KEY` を使って Google Places Autocomplete API (New) を呼び出し
4. 結果をクライアントに返却（APIキーは露出しない）

**レスポンス（成功）:**
```json
{
  "suggestions": [
    {
      "placeId": "ChIJ...",
      "mainText": "東京タワー",
      "secondaryText": "東京都港区芝公園"
    }
  ]
}
```

**エラー:**
| ステータス | 理由 |
|-----------|------|
| 400 | input パラメータなし / 短すぎる |
| 401 | 未認証 |
| 502 | Google Places API の呼び出し失敗 |

---

### 1.2 Google Places 写真プロキシ

```
GET /api/places/photo?reference=xxx
```

**クエリパラメータ:**
| パラメータ | 必須 | 説明 |
|-----------|------|------|
| reference | Yes | Place.photoReference の値 |

**処理:**
1. `auth()` で認証確認（未認証なら401）
2. `reference` バリデーション（空文字は400）
3. サーバーサイドで `GOOGLE_PLACES_API_KEY` を使って Google Places Photo API を呼び出し、画像バイナリを取得
4. 画像データをそのままレスポンスとして返却（Content-Type: image/jpeg）

**エラー:**
| ステータス | 理由 |
|-----------|------|
| 400 | reference パラメータなし |
| 401 | 未認証 |
| 502 | Google Places API の呼び出し失敗 |

**備考:**
- Google Places API (New) では Photo URI が Place Details レスポンスに含まれる場合がある。`resolvePlace` 時にPhoto URIを直接取得・保存できる場合は、このプロキシは不要になる。実装時にAPIレスポンスを確認して判断する

---

### 1.3 画像アップロード

```
POST /api/upload
```

**リクエスト:**
- Content-Type: `multipart/form-data`
- フィールド名: `file`
- 受け付けるMIME: `image/jpeg`, `image/png`
- 最大サイズ: 5MB

**レスポンス（成功）:**
```json
{
  "imagePath": "/uploads/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.jpg"
}
```

**エラー:**
| ステータス | 理由 |
|-----------|------|
| 400 | ファイルなし / 不正なMIMEタイプ / サイズ超過 |
| 401 | 未認証（`auth()` で確認。middleware の対象外のため自身でチェック必須） |
| 500 | ファイル書き込み失敗 |

**備考:**
- ファイル名は `crypto.randomUUID()` で生成し（小文字hex、ハイフン区切り36文字）、`public/uploads/` に保存
- 画像の削除（Spot削除時）はServer Action内で `fs.unlink` を直接呼ぶ。Route Handlerは不要

---

## 2. Server Actions

### 2.1 認証

#### login

ログイン。成功時はマップ画面へリダイレクト。

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| email | string | Yes | メール形式 |
| password | string | Yes | 空でないこと |

**処理:**
1. NextAuth v5 の `signIn("credentials", { email, password, redirectTo: "/" })` を呼び出し
2. 成功時: `/` へリダイレクト（NextAuth が内部でリダイレクトをスローする）
3. 失敗時: `AuthError` をキャッチして `{ error: "メールアドレスまたはパスワードが正しくありません" }` を返す

> **注意:** `signIn()` は成功時に `NEXT_REDIRECT` をスローする。`try/catch` で `AuthError` のみをキャッチし、それ以外のエラー（`NEXT_REDIRECT` 含む）は再スローすること。

---

#### register

ユーザー登録。成功時はマップ画面へリダイレクト。

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| name | string | No | 最大100文字 |
| email | string | Yes | メール形式、未使用であること |
| password | string | Yes | 8文字以上 |

**処理:**
1. メール重複チェック
2. bcryptでパスワードハッシュ化
3. User作成
4. NextAuth v5 の `signIn("credentials", { email, password, redirectTo: "/" })` を呼び出し

> **注意:** `signIn()` は成功時に `NEXT_REDIRECT` をスローする。ユーザー作成のエラーハンドリング（メール重複等）は `signIn()` 呼び出しの前に完結させること。`try/catch` で `NEXT_REDIRECT` を握りつぶさないこと。

---

#### logout

セッション破棄。`/login` へリダイレクト。

**入力:** なし

---

### 2.2 Place操作

#### resolvePlace（ヘルパー関数 — `lib/google-places.ts`）

Google Places placeId からPlaceをDBにgetOrCreateする。Server ActionsやServer Componentsから呼ばれるヘルパー関数。`"use server"` ファイルには含めない。

**引数:** `googlePlaceId: string`

**処理:**
1. `googlePlaceId` でPlaceを検索
2. 存在すればそのPlaceを返す
3. 存在しなければ Google Places API (New) の Place Details を呼び出し、name, address, lat/lng, prefectureCode, googleTypes, photoReference を取得してPlaceを作成
4. 作成したPlaceを返す

**prefectureCode導出:** Google Places の住所コンポーネントから都道府県名を取得し、`PREFECTURES` 定数と照合してコードを導出する。照合できない場合は null。

**エラー時:** Google Places API呼び出しが失敗した場合（ネットワークエラー、クォータ超過、無効なplaceId等）は例外をスローする。呼び出し元（`navigateToPlace` 等）がキャッチしてユーザー向けエラーメッセージに変換する。

#### navigateToPlace

探す画面でGoogle候補をタップした際に呼ばれるServer Action。PlaceをgetOrCreate後、Place詳細画面へリダイレクトする。

**引数:** `googlePlaceId: string`

**処理:**
1. セッションからuserId取得（未認証なら `/login` へリダイレクト）
2. `resolvePlace(googlePlaceId)` でPlaceをgetOrCreate
3. Google Places API呼び出しが失敗した場合は `{ error: "場所情報の取得に失敗しました" }` を返す
4. `/places/[place.id]` へリダイレクト

> **呼び出しパターン:** 探す画面（Client Component）からフォームsubmitではなくプログラム的に呼び出す。クライアント側では `startTransition` 内でServer Actionを呼び出し、`redirect()` による `NEXT_REDIRECT` をNext.jsに正しくハンドリングさせること。エラー時は `ActionState` で返却するため、呼び出し元で表示処理を行う。

> 探す画面からServer Actionとして呼び出すことで、`resolvePlace` のサーバーサイド処理（Google API呼び出し・DB操作）をクライアントに露出させない。

---

### 2.3 スポット操作

#### addSpotFromPlace

Place詳細画面の「行きたい！」からWANTスポットを新規作成。成功時はスポット一覧へリダイレクト。

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| placeId | string | Yes | DBクエリで存在確認 |
| categoryId | string | Yes | DBクエリで存在確認 |
| expectation | string | Yes | "1"〜"5" |
| memo | string | No | 最大1000文字 |

> Place詳細画面から呼ばれる時点でPlaceはDB上に存在するため、`placeId`（DB ID）を受け取る。`googlePlaceId` は不要。

**処理:**
1. セッションからuserId取得（未認証なら `/login` へリダイレクト）
2. バリデーション
3. PlaceのDB存在確認
4. 重複チェック: `(userId, placeId)` でSpotが既に存在する場合は `{ error: "この場所は既に登録されています" }` を返す
5. Spot作成（status=WANT、placeId紐づけ）
6. `/spots` へリダイレクト

---

#### updateSpot

登録済みSpotを更新。WENT時は満足度・感想・訪問日も含む。

**引数:** `spotId: string`

**入力 (FormData):**
| フィールド | 型 | 必須 | 備考 |
|-----------|-----|------|------|
| categoryId | string | Yes | DBクエリで存在確認 |
| expectation | string | Yes | |
| memo | string | No | |
| satisfaction | string | WENT時のみ必須 | "1"〜"5" |
| review | string | No | 最大1000文字 |
| visitedAt | string | No | `YYYY-MM-DD` 形式。未来日も許可する |

> 場所名（name）・都道府県（prefectureCode）はPlace由来のため編集不可。

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`）。存在しない場合は404を返す
3. 取得したSpotの `status` が WENT の場合、`satisfaction` の存在チェックを行う
4. バリデーション
5. Spot更新
6. `/spots/[id]` へリダイレクト

---

#### deleteSpot

Spotを削除。関連するPublicReviewと画像ファイルも削除する。

**引数:** `spotId: string`

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`、PublicReviewも含む）。存在しない場合は404を返す
3. Spotレコードを物理削除（関連するPublicReviewはCASCADE削除。DB削除を先に行う。DB削除が失敗した場合はここで停止し、画像は削除しない）
4. `imagePath` が存在する場合は `fs.unlink` でファイルを物理削除（ファイルが存在しなくてもエラーにしない。失敗はログに残すがユーザーエラーとしては扱わない）
5. `/spots` へリダイレクト

---

#### recordWent

WANTスポットをWENTに切り替える（初回記録専用）。公開トグルON時はPublicReviewも同時作成。

**引数:** `spotId: string`

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| satisfaction | string | Yes | "1"〜"5" |
| review | string | No | 最大1000文字 |
| imagePath | string | No | 後述のパターン検証を行う |
| visitedAt | string | No | `YYYY-MM-DD` 形式。未来日も許可する |
| isPublic | string | No | "true" で公開。未指定時は非公開 |

**前提:**
- 画像は事前にクライアントサイドJSで `POST /api/upload` へ送信済み。このActionにはファイルではなくパス文字列のみ渡す
- このフローはクライアントサイドJSが必須（progressive enhancement なし）
- WANTスポットには画像がないため、旧画像の削除は発生しない

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`、Place情報も含む）。存在しない場合は404を返す
3. `status` が WANT であることを確認（WENT済みの場合は詳細画面にリダイレクト）
4. `imagePath` が指定されている場合、`^/uploads/[0-9a-f-]{36}\.(jpg|png)$` にマッチするか検証する（不正なパスを拒否）
5. バリデーション
6. Spot更新（status=WENT、満足度・感想・imagePath・訪問日を保存）
7. `isPublic` が "true" の場合、PublicReview を作成（Spotデータのスナップショット + User.nameをdisplayNameとして保存）
8. `/spots/[id]` へリダイレクト

---

### 2.4 口コミ操作（`actions/reviews.ts`）

#### publishReview

WENT済みSpotの記録をPublicReviewとして公開する。スポット詳細画面の「公開する」ボタンから呼ばれる。

**引数:** `spotId: string`

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId AND status=WENT`、Place情報も含む）。存在しない場合は404を返す
3. 既にPublicReviewが存在する場合は `{ error: "既に公開されています" }` を返す
4. PublicReview作成（Spot + Userのデータをスナップショットとしてコピー）
5. `/spots/[id]` へリダイレクト

---

#### unpublishReview

公開済みの口コミを非公開にする（PublicReviewを削除）。

**引数:** `spotId: string`

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`）。存在しない場合は404を返す
3. SpotのPublicReviewを取得。存在しない場合は詳細画面にリダイレクト
4. PublicReview を物理削除
5. `/spots/[id]` へリダイレクト

---

## 3. Server Componentsでのデータ取得

Server Componentsから直接Prismaを呼ぶ。関数は `lib/` や専用の `queries.ts` にまとめる。

### 3.1 スポット一覧

```
対象: ログインユーザーのSpotのみ（WHERE userId = currentUserId）
リレーション: Place を include（スポット名・都道府県はPlace由来）
フィルタ: Place.prefectureCode（任意）、status（任意）
ソート: updatedAt DESC / expectation DESC / Place.prefectureCode ASC (NULLS LAST)
```

**フィルタ・ソートの状態管理:**
URLクエリパラメータで管理する（例: `/spots?prefecture=13&status=WANT&sort=expectation`）。
プルダウン変更時はClient Componentが `router.push` でURLを更新し、Server Componentが `searchParams` から読み取って再レンダリングする。

### 3.2 スポット詳細

```
対象: WHERE id = spotId AND userId = currentUserId
リレーション: Place を include、PublicReview を include（公開状態の判定用）
所有者不一致 / 存在しない場合: 404 を返す（リソースの存在を他ユーザーに知らせない）
```

### 3.3 Place詳細

```
対象: WHERE id = placeId
リレーション: PublicReview を include（口コミ一覧表示用）
存在しない場合: 404 を返す
```

> Place詳細は全認証済みユーザーが閲覧可能（userId条件なし）。

### 3.4 Place口コミ一覧

```
対象: WHERE placeId = placeId
ソート: createdAt DESC
```

> 全認証済みユーザーが閲覧可能。

### 3.5 制覇率集計（マップ用）

```
対象: WHERE userId = currentUserId AND status = WENT
リレーション: Place を include（prefectureCodeはPlace由来）
取得: Place.prefectureCode の一覧（重複排除）
制覇数: 上記の都道府県数
制覇率: 制覇数 / 47
```

### 3.6 カテゴリ一覧

```
対象: 全Category（シードデータ）
ソート: sortOrder ASC
```

### 3.7 ユーザーのSpot取得（Place詳細用）

```
対象: WHERE userId = currentUserId AND placeId = placeId
返却: Spot レコード（id, status を含む）。存在しない場合は null
用途: Place詳細画面で「行きたい！」ボタン（Spot未登録時）/ 「登録済み」リンク（Spot存在時、Spot詳細へ遷移）の表示判定
```
