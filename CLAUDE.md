# Claude Code Dashboard

## What

Claude Code の利用ログを収集・可視化するダッシュボード。
ユーザー単位で「どれくらい Claude Code を活用できているか」を分析する。

## Why

1. 自分の活用度を把握したい
2. チームメンバーの活用度をランキング形式で比較し、使い方の分析・改善につなげたい

## 分析対象

- Skills: どのスキルがどれだけ呼ばれたか
- Sub Agents: 使用回数・タイプ別内訳（Explore / Plan / general-purpose 等）
- Teams: 利用回数・内容
- MCP: どのサーバーのどのツールをどれだけ使っているか
- セッション: 1日の数、どれくらいで終了するか
- メッセージ数
- Token 使用量

集計粒度はユーザー単位（プロジェクト単位・セッション単位ではない）。

## How

Claude Code Plugin として提供。`claude plugin install` だけで導入完了。
Plugin 内の Hooks（PostToolUse / UserPromptSubmit / Stop / SessionEnd）が会話中に自動発火し、
必要なフィールドだけ抽出して async で API に POST する。
セキュリティのため prompt や assistant の応答本文は送らない。

## データソース

Hook の stdin から取得。`tool_name` と `tool_input` の一部フィールドのみ送信。
詳細は `docs/howToPost.md` と `docs/hooksSchema.md` を参照。

## Tech Stack

- Runtime: Node.js (TypeScript), pnpm
- Framework: Next.js (App Router, fullstack)
- UI: yamada-ui, recharts
- Auth: auth.js（Supabase Auth は使わない）
- DB: Supabase (PostgreSQL), Drizzle ORM
- Validation: zod
- Lint / Format: oxlint, oxfmt
- Git hooks: husky
- Test: Vitest, Playwright
- UI開発: Storybook
- 日付操作: date-fns
- DB マイグレーション: drizzle-kit
- CI/CD: GitHub Actions
- Infra: Docker

## 現状

要件定義・データ設計フェーズ。
