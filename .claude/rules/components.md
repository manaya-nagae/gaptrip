# コンポーネント設計 (Atomic Design × App Router)

## 層の定義

App Router では templates / pages 層は不要。最初の3層を使う。

```
src/
  components/
    atoms/       ← 最小単位。単体で意味が完結する
    molecules/   ← atoms の組み合わせ。単一の機能を持つ
    organisms/   ← 画面の塊。複数の molecules / atoms で構成
  app/
    layout.tsx   ← Template 相当（骨格）
    page.tsx     ← Page 相当（実データ）
```

## 各層の基準

**Atoms** — これ以上分解できない部品
- Button, Input, Label, Badge, Icon, Avatar, Spinner
- props で見た目を制御する
- ビジネスロジックを持たない

**Molecules** — atoms を組み合わせた単一機能の部品
- FormField（Label + Input + ErrorMessage）
- SearchBar（Input + Button）
- UserAvatar（Avatar + 名前テキスト）
- 「1つのことをする」が基準

**Organisms** — 独立して機能する画面の塊
- Header, Footer, Sidebar
- UserCard, PostList, CommentSection
- データ取得の起点になることもある（Server Component として）

## ファイル命名

```
components/
  atoms/
    button.tsx          ← kebab-case
    input.tsx
  molecules/
    form-field.tsx
    search-bar.tsx
  organisms/
    user-card.tsx
    site-header.tsx
```

コンポーネント名（関数名・export名）は PascalCase：
```ts
// components/atoms/button.tsx
export function Button({ ... }) { ... }
```

## Server / Client の判断

- atoms・molecules は基本 Server Component でよい
- イベントハンドラ・useState が必要なら `use client`
- `use client` は可能な限り末端（atoms レベル）に押し込む

```
// ❌ organisms 全体を use client にする
// ✅ インタラクションが必要な atom だけ use client にする
```

## やってはいけないこと

- organisms に atoms を飛ばして直接 HTML タグを書く（再利用できなくなる）
- atoms にビジネスロジックを入れる
- `components/` 直下にファイルを置く（必ずいずれかの層に入れる）
