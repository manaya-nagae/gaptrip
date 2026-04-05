@AGENTS.md
@.claude/rules/nextjs.md
@.claude/rules/prisma.md
@.claude/rules/components.md
@.claude/rules/auth.md

# Skill Routing (gstack + superpowers)

ユーザーの依頼がスキルに合致するとき、必ず Skill ツールで起動する。
直接回答せず、スキルのワークフローに従う。

優先順位: バグがある場合は investigate を先に実行してから ship を検討する。

| 状況 | スキル |
|------|--------|
| バグ / エラー / 壊れてる / 500 | `investigate` |
| 動作確認 / テスト / これ動く？ | `qa` |
| PR 作成 / デプロイ / push（バグなし確認後） | `ship` |
| コードレビュー / diff 確認 | `review` |
| ドキュメント更新 | `document-release` |
| デザイン確認 / UI の問題 | `design-review` |
| デザインシステム / ブランド | `design-consultation` |
| 設計 / アーキテクチャ / 何を作るべきか | `office-hours` |
| アーキテクチャレビュー | `plan-eng-review` |
| 戦略レビュー / 機能の方向性 | `plan-ceo-review` |
| 週次振り返り | `retro` |
| コード品質チェック / 健全性確認 | `health` |
| 作業の区切り / 進捗保存 | `checkpoint` |
| ブレインストーミング | `superpowers:brainstorm` |
| 計画立案 | `superpowers:write-plan` |
| 計画の実行 | `superpowers:execute-plan` |

# Git 操作のルール

`git commit` と `git push` はユーザーが明示的に許可したときのみ実行する。
スキル・エージェントが自律的に判断してコミットしない。

コミットメッセージに `Co-Authored-By` や `🤖` などの Claude に関する記述を含めない。

# Git ワークフロー

コミットメッセージ形式: `<prefix>: <日本語の説明>`

| prefix | 用途 |
|--------|------|
| `[feat]` | 新機能 |
| `[fix]` | バグ修正 |
| `[chore]` | 設定・依存関係・ビルド |
| `[docs]` | ドキュメント |
| `[refactor]` | リファクタリング |
| `[test]` | テスト |

例: `[feat] ユーザー認証を追加`, `[fix] ログイン時のリダイレクトを修正`

ブランチ命名: `feature/xxx`, `fix/xxx`, `chore/xxx`
