# Hooks stdin JSON スキーマ

各 Hook イベント発火時に Claude Code が stdin に流し込む JSON の定義。

## 全イベント共通フィールド

```json
{
  "session_id": "abc123-def456",
  "transcript_path": "/Users/<user>/.claude/projects/<project-id>/<session-id>.jsonl",
  "cwd": "/Users/<user>/Documents/project/xxx",
  "permission_mode": "default | plan | acceptEdits | dontAsk | bypassPermissions",
  "hook_event_name": "PostToolUse | UserPromptSubmit | Stop | SessionEnd"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `session_id` | string | セッション識別子 |
| `transcript_path` | string | セッション JSONL ファイルへの絶対パス |
| `cwd` | string | 発火時の作業ディレクトリ |
| `permission_mode` | string | 現在のパーミッションモード |
| `hook_event_name` | string | 発火した Hook イベント名 |

## PostToolUse

Claude がツールを使い終わった直後に発火。

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "permission_mode": "...",
  "hook_event_name": "PostToolUse",
  "tool_name": "Agent",
  "tool_input": {
    "subagent_type": "Explore",
    "description": "..."
  },
  "tool_response": {
    "success": true
  },
  "tool_use_id": "toolu_01ABC123...",
  "agent_id": "agent-xxx",
  "agent_type": "Explore"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `tool_name` | string | 実行されたツール名（`Skill`, `Agent`, `mcp__*` 等） |
| `tool_input` | object | ツールに渡された入力パラメータ |
| `tool_response` | object | ツールの実行結果 |
| `tool_use_id` | string | ツール実行の一意識別子 |
| `agent_id` | string? | sub agent 内で発火した場合のみ存在 |
| `agent_type` | string? | sub agent 内で発火した場合のみ存在 |

### tool_name ごとの tool_input 例

#### Skills

```json
{ "tool_name": "Skill", "tool_input": { "skill": "create-components-best-practice" } }
```

#### Sub Agents

```json
{ "tool_name": "Agent", "tool_input": { "subagent_type": "Explore", "description": "型の使用箇所を調査" } }
```

#### Teams

```json
{ "tool_name": "TeamCreate", "tool_input": { "team_name": "epic-optimistic", "description": "調査チーム" } }
{ "tool_name": "SendMessage", "tool_input": { "recipient": "planner", "summary": "..." } }
{ "tool_name": "TeamDelete", "tool_input": {} }
```

#### MCP

```json
{ "tool_name": "mcp__chrome-devtools__take_screenshot", "tool_input": {} }
{ "tool_name": "mcp__chrome-devtools__evaluate_script", "tool_input": { "function": "..." } }
```

`tool_name` を `mcp__{server}__{tool}` で分割するとサーバー名・ツール名を個別に集計できる。

## UserPromptSubmit

ユーザーがメッセージを送信した瞬間に発火。

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "permission_mode": "...",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "このファイル修正して"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `prompt` | string | ユーザーが入力したテキスト |

## Stop

Claude が応答を書き終わった瞬間に発火。

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "permission_mode": "...",
  "hook_event_name": "Stop",
  "stop_hook_active": false,
  "last_assistant_message": "修正しました。変更点は..."
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `stop_hook_active` | boolean | 無限ループ防止用フラグ |
| `last_assistant_message` | string | Claude の最後の応答テキスト |

## SessionEnd

セッション終了時に発火。

```json
{
  "session_id": "...",
  "transcript_path": "...",
  "cwd": "...",
  "permission_mode": "...",
  "hook_event_name": "SessionEnd",
  "reason": "clear"
}
```

| フィールド | 型 | 説明 |
|---|---|---|
| `reason` | string | 終了理由。`"clear"`, `"logout"`, `"prompt_input_exit"`, `"bypass_permissions_disabled"`, `"other"` |

## stdin に含まれないデータ

| データ | 取得方法 |
|---|---|
| **token usage** (input/output/cache) | `transcript_path` の JSONL 内 `message.usage` を読む |
| **セッション長** (duration) | Hook スクリプト内で `date` を使うか、JSONL のタイムスタンプ差分で計算 |
| **timestamp** | Hook スクリプト内で `date` コマンドで自前生成 |
