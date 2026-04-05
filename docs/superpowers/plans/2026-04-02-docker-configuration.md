# GapTrip Docker構成 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** GapTrip のローカル開発環境を Next.js + Prisma + PostgreSQL の2コンテナ構成で Docker 化する。

**Architecture:** Next.js アプリ(app コンテナ)と PostgreSQL(db コンテナ)の2コンテナ構成。Prisma を ORM として使用し、NextAuth.js で認証を実装する土台を作る。開発環境に特化し、ホットリロードとバインドマウントで快適な開発体験を提供する。

**Tech Stack:** Next.js 15 (App Router), TypeScript, Prisma, PostgreSQL 16, NextAuth.js (Auth.js), Node.js 20, Docker Compose

---

## ファイル構成

| ファイル | 操作 | 責務 |
|---|---|---|
| `.gitignore` | 作成 | Git追跡除外設定 |
| `.env` | 作成 | 環境変数(DB接続情報、NextAuth設定) |
| `.env.example` | 作成 | 環境変数テンプレート(git管理用) |
| `.dockerignore` | 作成 | Dockerビルド時の除外設定 |
| `Dockerfile` | 作成 | app コンテナのビルド定義 |
| `docker-compose.yml` | 作成 | 2コンテナ構成定義 |
| `package.json` | 作成 | Node.js 依存パッケージ定義 |
| `tsconfig.json` | 自動生成 | TypeScript設定(Next.js が生成) |
| `prisma/schema.prisma` | 作成 | Prisma スキーマ(初期User モデル) |
| `src/app/layout.tsx` | 自動生成 | Next.js ルートレイアウト |
| `src/app/page.tsx` | 自動生成 | Next.js トップページ |

---

### Task 1: .gitignore は Task 3 で create-next-app が生成するためスキップ

Task 3 の Step 3 で `.env` 関連の修正を行う。

---

### Task 2: 環境変数ファイルを作成する

**Files:**
- Create: `.env`
- Create: `.env.example`

- [ ] **Step 1: .env.example を作成する（git管理用テンプレート）**

```env
# Database
POSTGRES_USER=gap_user
POSTGRES_PASSWORD=gap_pass
POSTGRES_DB=gap_trip
DATABASE_URL=postgresql://gap_user:gap_pass@db:5432/gap_trip

# NextAuth
NEXTAUTH_SECRET=your-secret-key-here
NEXTAUTH_URL=http://localhost:3000
```

- [ ] **Step 2: .env を作成する（実際の値）**

`.env.example` をコピーして `.env` を作成する。内容は同一でOK（開発環境のため）。

```bash
cp .env.example .env
```

- [ ] **Step 3: .env が .gitignore で除外されていることを確認**

```bash
git status
```

Expected: `.env` が untracked files に表示されないこと。`.env.example` のみ表示される。

- [ ] **Step 4: コミット**

```bash
git add .env.example
git commit -m "chore: add .env.example template"
```

---

### Task 3: Next.js プロジェクトを初期化する

**Files:**
- Create: `package.json`（npx create-next-app が生成）
- Create: `tsconfig.json`（自動生成）
- Create: `src/app/layout.tsx`（自動生成）
- Create: `src/app/page.tsx`（自動生成）

- [ ] **Step 1: Next.js プロジェクトを初期化する**

ホストマシンで実行する（Docker構築前のため）。Node.js 20 がホストに入っている前提。

```bash
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm
```

注意: `.` で現在のディレクトリに作成する。既にファイルがある場合 `--yes` は使わず、対話プロンプトで進める。

- [ ] **Step 2: 生成されたファイルを確認**

```bash
ls -la src/app/
cat package.json
```

Expected: `src/app/layout.tsx`, `src/app/page.tsx` が存在すること。`package.json` に `next`, `react`, `react-dom`, `typescript` が含まれていること。

- [ ] **Step 3: .gitignore の .env 行を修正する**

create-next-app が生成した `.gitignore` の `.env*` は `.env.example` も除外してしまうため修正する。prisma 関連の追記は不要（SQLite ではなく PostgreSQL を使うため）。

`.gitignore` 内の以下を変更:

```diff
- .env*
+ .env
+ .env.local
+ .env.*.local
```

- [ ] **Step 4: コミット**

```bash
git add .
git commit -m "feat: initialize Next.js project with TypeScript and Tailwind CSS"
```

---

### Task 4: Prisma をセットアップする

**Files:**
- Create: `prisma/schema.prisma`
- Modify: `package.json`（依存追加）

- [ ] **Step 1: Prisma をインストール**

```bash
npm install prisma --save-dev
npm install @prisma/client
```

- [ ] **Step 2: Prisma を初期化**

```bash
npx prisma init --datasource-provider postgresql
```

Expected: `prisma/schema.prisma` が生成される。

- [ ] **Step 3: prisma/schema.prisma を確認し、初期 User モデルを追加**

生成された `prisma/schema.prisma` を以下に置き換える:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

この User モデルは認証の土台として最小限のフィールドのみ定義する。GapTrip の本格的なテーブル設計（spots, reviews 等）は別タスクで追加する。

- [ ] **Step 4: コミット**

```bash
git add prisma/schema.prisma package.json package-lock.json
git commit -m "feat: add Prisma with initial User model"
```

---

### Task 5: Dockerfile を作成する

**Files:**
- Create: `docker/app/Dockerfile`

- [ ] **Step 1: docker/app/Dockerfile を作成する**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npx prisma generate
EXPOSE 3000
CMD ["npm", "run", "dev"]
```

- [ ] **Step 2: コミット**

```bash
git add docker/app/Dockerfile
git commit -m "chore: add Dockerfile for development"
```

---

### Task 6: .dockerignore を作成する

**Files:**
- Create: `.dockerignore`

- [ ] **Step 1: .dockerignore を作成する**

```dockerignore
node_modules
.next
postgres_data
.git
```

- [ ] **Step 2: コミット**

```bash
git add .dockerignore
git commit -m "chore: add .dockerignore"
```

---

### Task 7: docker-compose.yml を作成する

**Files:**
- Create: `docker-compose.yml`

- [ ] **Step 1: docker-compose.yml を作成する**

```yaml
services:
  app:
    build:
      context: .
      dockerfile: docker/app/Dockerfile
    ports:
      - "3000:3000"
      - "5555:5555"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - NEXTAUTH_URL=${NEXTAUTH_URL}
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
      - node_modules:/app/node_modules

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5433:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gap_user -d gap_trip"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  node_modules:
  postgres_data:
```

注意: healthcheck の `test` では環境変数展開が不安定な場合があるため、値を直書きしている（`gap_user`, `gap_trip`）。

- [ ] **Step 2: コミット**

```bash
git add docker-compose.yml
git commit -m "feat: add docker-compose.yml with app and db services"
```

---

### Task 8: Docker 起動と動作確認

**Files:** なし（実行確認のみ）

- [ ] **Step 1: Docker Compose でビルド・起動**

```bash
docker compose up -d --build
```

Expected: 2つのコンテナが起動すること。

- [ ] **Step 2: コンテナの状態を確認**

```bash
docker compose ps
```

Expected: `app` と `db` の両方が `running` (healthy) であること。

- [ ] **Step 3: Next.js アプリにアクセス**

ブラウザで `http://localhost:3000` を開く。

Expected: Next.js のデフォルトページが表示されること。

- [ ] **Step 4: DB接続を確認 — Prisma マイグレーション実行**

```bash
docker compose exec app npx prisma migrate dev --name init
```

Expected: マイグレーションが成功し、`prisma/migrations/` ディレクトリにマイグレーションファイルが生成されること。

- [ ] **Step 5: Prisma Studio の起動確認**

```bash
docker compose exec app npx prisma studio
```

Expected: ブラウザで `http://localhost:5555` にアクセスすると、Prisma Studio が表示され、User テーブルが確認できること。

確認後、Ctrl+C で Prisma Studio を停止する。

- [ ] **Step 6: マイグレーションファイルをコミット**

```bash
git add prisma/migrations/
git commit -m "feat: add initial database migration with User model"
```

- [ ] **Step 7: 停止と再起動の確認**

```bash
docker compose down
docker compose up -d
```

Expected: 再起動後も `http://localhost:3000` が正常に表示されること。DBのデータが保持されていること（postgres_data ボリュームにより永続化）。
