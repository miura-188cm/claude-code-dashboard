# データ収集方法

Claude Code の Hooks 機能を使い、会話中のイベントを自動で API に POST する。

## 方針

- 各 PC の `~/.claude/settings.json` に Hooks を設定するだけ
- Claude Code が通常通り動くたびに自動で発火し、async で API に送信
- ユーザーの操作には一切影響しない

## 使用する Hook イベント

| Hook | 発火タイミング | 送るデータ | 目的 |
|---|---|---|---|
| `PostToolUse` | Claude がツールを使い終わった直後 | `tool_name`, `tool_input` | Skills / Sub Agents / Teams / MCP の利用記録 |
| `UserPromptSubmit` | ユーザーがメッセージを送信した瞬間 | `session_id`, `timestamp` | メッセージ数のカウント |
| `Stop` | Claude が応答を書き終わった瞬間 | `usage` (token数) | Token 使用量（input/output/cache） |
| `SessionEnd` | `/clear`、`exit`、ウィンドウを閉じた時 | `session_id`, `duration_seconds` | セッションの長さ |

## settings.json の設定例

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Skill|Agent|TeamCreate|SendMessage|TeamDelete|mcp__.*",
        "hooks": [
          { "type": "http", "url": "https://<your-api>/post-tool-use", "async": true }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "http", "url": "https://<your-api>/user-prompt-submit", "async": true }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "http", "url": "https://<your-api>/stop", "async": true }
        ]
      }
    ],
    "SessionEnd": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "http", "url": "https://<your-api>/session-end", "async": true }
        ]
      }
    ]
  }
}
```

## 会話中の発火フロー

```
ユーザー「このファイル修正して」       ← UserPromptSubmit 発火
  ↓
Claude: Read 実行                     ← PostToolUse (matcher 外なのでスキップ)
Claude: Agent(Explore) 実行            ← PostToolUse 発火 → POST
Claude: Edit 実行                     ← PostToolUse (matcher 外なのでスキップ)
Claude:「修正しました」                ← Stop 発火 → POST (token usage)
  ↓
ユーザー「ありがとう」                 ← UserPromptSubmit 発火
  ↓
Claude:「どういたしまして」             ← Stop 発火 → POST (token usage)
  ↓
ユーザー: exit                         ← SessionEnd 発火 → POST (duration)
```

## PostToolUse で取得できるデータの詳細

### Skills

```json
{ "tool_name": "Skill", "tool_input": { "skill": "create-components-best-practice" } }
```

### Sub Agents

```json
{ "tool_name": "Agent", "tool_input": { "subagent_type": "Explore", "description": "型の使用箇所を調査" } }
```

### Teams

```json
{ "tool_name": "TeamCreate", "tool_input": { "team_name": "epic-optimistic", "description": "調査チーム" } }
{ "tool_name": "SendMessage", "tool_input": { "recipient": "planner", "summary": "..." } }
{ "tool_name": "TeamDelete", "tool_input": {} }
```

### MCP

```json
{ "tool_name": "mcp__chrome-devtools__take_screenshot", "tool_input": {} }
{ "tool_name": "mcp__chrome-devtools__evaluate_script", "tool_input": { "function": "..." } }
```

`tool_name` を `mcp__{server}__{tool}` で分割すると、サーバー名・ツール名を個別に集計できる。

## PostToolUse の matcher について

`"matcher": "Skill|Agent|TeamCreate|SendMessage|TeamDelete|mcp__.*"` で分析対象のツールだけに絞っている。

全ツール（Read, Edit, Bash 等）も記録したい場合は `"matcher": ".*"` に変更する。
ただしリクエスト数が大幅に増える（1セッションで数十〜数百回）。

## 各 POST に共通で付与すべきフィールド

API 側でユーザー・セッションを識別するために、以下を共通で送る：

| フィールド | 取得元 | 用途 |
|---|---|---|
| `session_id` | stdin の JSON | セッション識別 |
| `timestamp` | stdin の JSON or 自前生成 | 時系列分析 |
| `user_id` | 環境変数 or hostname | ユーザー識別・ランキング |
