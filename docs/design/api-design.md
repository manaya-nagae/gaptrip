# GapTrip API設計書

## 概要

本書はGapTrip MVPのサーバーサイド処理（Server Actions・Route Handler・データ取得）の仕様を定義する。

### 責務の分担

| 種別 | 用途 | 認証チェック |
|------|------|-------------|
| Server Components + Prisma | データ取得（一覧・詳細・集計） | セッション取得で確認 |
| Server Actions | データ更新（作成・編集・削除・WENT記録・認証） | セッション取得で確認（認証系を除く） |
| Route Handler | ファイルアップロード | 自身で `auth()` を呼んで確認 |

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

### 1.1 画像アップロード

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
- ファイル名はUUIDで生成し、`public/uploads/` に保存
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

### 2.2 スポット操作

#### createSpot

WANTスポットを新規作成。成功時はスポット一覧へリダイレクト。

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| name | string | Yes | 最大100文字 |
| prefectureCode | string | Yes | `PREFECTURES` 定数に含まれるコードであること |
| categoryId | string | Yes | DBクエリで存在確認 |
| expectation | string | Yes | "1"〜"5" |
| memo | string | No | 最大1000文字 |

**処理:**
1. セッションからuserId取得（未認証なら `/login` へリダイレクト）
2. バリデーション
3. Spot作成（status=WANT）
4. `/spots` へリダイレクト

---

#### updateSpot

登録済みSpotを更新。WENT時は満足度・感想・訪問日も含む。

**引数:** `spotId: string`

**入力 (FormData):**
| フィールド | 型 | 必須 | 備考 |
|-----------|-----|------|------|
| name | string | Yes | |
| prefectureCode | string | Yes | `PREFECTURES` 定数に含まれるコードであること |
| categoryId | string | Yes | DBクエリで存在確認 |
| expectation | string | Yes | |
| memo | string | No | |
| satisfaction | string | WENT時のみ必須 | "1"〜"5" |
| review | string | No | 最大1000文字 |
| visitedAt | string | No | `YYYY-MM-DD` 形式。未来日も許可する |

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`）。存在しない場合は404を返す
3. 取得したSpotの `status` が WENT の場合、`satisfaction` の存在チェックを行う
4. バリデーション
5. Spot更新
6. `/spots/[id]` へリダイレクト

---

#### deleteSpot

Spotを削除。関連画像ファイルも削除する。

**引数:** `spotId: string`

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`）。存在しない場合は404を返す
3. Spotレコードを物理削除（DB削除を先に行う。DB削除が失敗した場合はここで停止し、画像は削除しない）
4. `imagePath` が存在する場合は `fs.unlink` でファイルを物理削除（ファイルが存在しなくてもエラーにしない。失敗はログに残すがユーザーエラーとしては扱わない）
5. `/spots` へリダイレクト

---

#### recordWent

WANTスポットをWENTに切り替える（初回記録専用）。

**引数:** `spotId: string`

**入力 (FormData):**
| フィールド | 型 | 必須 | バリデーション |
|-----------|-----|------|--------------|
| satisfaction | string | Yes | "1"〜"5" |
| review | string | No | 最大1000文字 |
| imagePath | string | No | 後述のパターン検証を行う |
| visitedAt | string | No | `YYYY-MM-DD` 形式。未来日も許可する |

**前提:**
- 画像は事前にクライアントサイドJSで `POST /api/upload` へ送信済み。このActionにはファイルではなくパス文字列のみ渡す
- このフローはクライアントサイドJSが必須（progressive enhancement なし）
- WANTスポットには画像がないため、旧画像の削除は発生しない

**処理:**
1. セッションからuserId取得
2. DBからSpotを取得（`WHERE id=spotId AND userId=userId`）。存在しない場合は404を返す
3. `status` が WANT であることを確認（WENT済みの場合は詳細画面にリダイレクト）
4. `imagePath` が指定されている場合、`^/uploads/[0-9a-f-]{36}\.(jpg|png)$` にマッチするか検証する（不正なパスを拒否）
5. バリデーション
6. Spot更新（status=WENT、満足度・感想・imagePath・訪問日を保存）
7. `/spots/[id]` へリダイレクト

---

## 3. Server Componentsでのデータ取得

Server Componentsから直接Prismaを呼ぶ。関数は `lib/` や専用の `queries.ts` にまとめる。

### 3.1 スポット一覧

```
対象: ログインユーザーのSpotのみ（WHERE userId = currentUserId）
フィルタ: prefectureCode（任意）、status（任意）
ソート: updatedAt DESC / expectation DESC / prefectureCode ASC
```

**フィルタ・ソートの状態管理:**
URLクエリパラメータで管理する（例: `/spots?prefecture=13&status=WANT&sort=expectation`）。
プルダウン変更時はClient Componentが `router.push` でURLを更新し、Server Componentが `searchParams` から読み取って再レンダリングする。

### 3.2 スポット詳細

```
対象: WHERE id = spotId AND userId = currentUserId
所有者不一致 / 存在しない場合: 404 を返す（リソースの存在を他ユーザーに知らせない）
```

### 3.3 制覇率集計（マップ用）

```
対象: WHERE userId = currentUserId AND status = WENT
取得: prefectureCode の一覧（重複排除）
制覇数: 上記の都道府県数
制覇率: 制覇数 / 47
```

### 3.4 カテゴリ一覧

```
対象: 全Category（シードデータ）
ソート: sortOrder ASC
```
