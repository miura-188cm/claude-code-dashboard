# Claude Code ローカルログ — 取得可能データ一覧

Claude Code が `~/.claude/` に保存しているデータを網羅的に整理する。

---

## 1. データソース一覧

| # | データソース | パス | 形式 | 概要 |
|---|---|---|---|---|
| 1 | セッションログ | `~/.claude/projects/{project-id}/{session-id}.jsonl` | JSONL | 全メッセージ・ツール呼び出し・token使用量の完全な記録 |
| 2 | 統計キャッシュ | `~/.claude/stats-cache.json` | JSON | 日次アクティビティ集計（メッセージ数、セッション数等） |
| 3 | コマンド履歴 | `~/.claude/history.jsonl` | JSONL | ユーザーが入力したコマンド・スラッシュコマンド |
| 4 | デバッグログ | `~/.claude/debug/{session-id}.txt` | テキスト | セッションごとの詳細デバッグログ（タイムスタンプ付き） |
| 5 | テレメトリ | `~/.claude/telemetry/` | JSON | 送信失敗したテレメトリイベント |
| 6 | Teams設定 | `~/.claude/teams/{team-name}/config.json` | JSON | チーム構成・メンバー・モデル設定 |
| 7 | Teams Inbox | `~/.claude/teams/{team-name}/inboxes/{agent}.json` | JSON | エージェント間メッセージ通信 |
| 8 | Tasks | `~/.claude/tasks/{team-or-session-id}/*.json` | JSON | タスク定義（subject, description, status） |
| 9 | Plans | `~/.claude/plans/*.md` | Markdown | 実装計画ドキュメント |
| 10 | Todos | `~/.claude/todos/{session-id}-agent-{id}.json` | JSON | セッション内Todoリスト |
| 11 | File History | `~/.claude/file-history/{session-id}/` | バイナリ | ファイル変更のスナップショット（undo用） |
| 12 | Settings | `~/.claude/settings.json` | JSON | ユーザー設定 |

---

## 2. セッションログ詳細（最重要データソース）

`~/.claude/projects/{project-id}/{session-id}.jsonl` の各行は以下のいずれかの type を持つ。

### 2.1 メッセージタイプ一覧

| type | 説明 | 主な用途 |
|---|---|---|
| `user` | ユーザーメッセージ | ユーザー入力の分析、メッセージ数集計 |
| `assistant` | アシスタント応答 | ツール呼び出し・token使用量・応答分析 |
| `system` | システムメッセージ | compact_boundary, turn_duration, local_command |
| `progress` | 進捗イベント | hook実行状況 |
| `queue-operation` | キュー操作 | セッション開始・終了の検出 |
| `file-history-snapshot` | ファイル履歴 | ファイル変更追跡 |
| `last-prompt` | 最終プロンプト | セッション復帰用 |

### 2.2 メッセージ共通フィールド

| フィールド | 型 | 説明 | 分析用途 |
|---|---|---|---|
| `type` | string | メッセージタイプ（上表参照） | 分類 |
| `uuid` | string | メッセージ固有ID | 一意識別 |
| `parentUuid` | string\|null | 親メッセージID | 会話ツリー構築 |
| `sessionId` | string | セッションID | セッション単位集計 |
| `timestamp` | string (ISO8601) | タイムスタンプ | 時系列分析 |
| `userType` | string | `"external"`（メイン）/ `"internal"` | sub agent判定 |
| `isSidechain` | boolean | サイドチェーンか否か | sub agent判定 |
| `cwd` | string | 作業ディレクトリ | プロジェクト判定 |
| `gitBranch` | string | Gitブランチ名 | ブランチ別分析 |
| `slug` | string | セッションスラグ | セッション識別 |
| `version` | string | Claude Code バージョン | バージョン別分析 |
| `permissionMode` | string | 権限モード | 権限モード別集計 |

### 2.3 assistant メッセージの `message.content` ブロック

`message.content` は配列で、以下のブロックタイプを含む。

| ブロックタイプ | フィールド | 説明 | 分析用途 |
|---|---|---|---|
| `text` | `text` | テキスト応答 | 応答内容分析 |
| `tool_use` | `name`, `input`, `id` | ツール呼び出し | ツール利用分析 |
| `tool_result` | `tool_use_id`, `content` | ツール実行結果 | ツール結果分析 |

### 2.4 Token 使用量（`message.usage`）

assistant メッセージに付与される。**コスト計算の核心データ**。

| フィールド | 型 | 説明 |
|---|---|---|
| `input_tokens` | number | 入力トークン数 |
| `output_tokens` | number | 出力トークン数 |
| `cache_creation_input_tokens` | number | キャッシュ作成トークン数 |
| `cache_read_input_tokens` | number | キャッシュ読み取りトークン数 |
| `cache_creation.ephemeral_1h_input_tokens` | number | 1時間キャッシュ作成トークン |
| `cache_creation.ephemeral_5m_input_tokens` | number | 5分キャッシュ作成トークン |
| `server_tool_use.web_search_requests` | number | Web検索リクエスト数 |
| `server_tool_use.web_fetch_requests` | number | Web取得リクエスト数 |
| `service_tier` | string | サービスティア（`"standard"` 等） |
| `speed` | string | 速度設定（`"standard"` 等） |
| `inference_geo` | string | 推論地域 |

### 2.5 system メッセージのサブタイプ

| subtype | 追加フィールド | 説明 |
|---|---|---|
| `compact_boundary` | `compactMetadata.trigger`, `compactMetadata.preTokens` | コンテキスト圧縮イベント |
| `turn_duration` | `durationMs` | ターンの所要時間（ミリ秒） |
| `local_command` | `content` | ローカルコマンド実行結果 |

---

## 3. ツール呼び出し詳細

### 3.1 利用可能ツール一覧

セッションログの `tool_use` ブロックの `name` から取得可能。

| ツール名 | 説明 | 取得可能パラメータ |
|---|---|---|
| `Read` | ファイル読み取り | `file_path`, `offset`, `limit` |
| `Write` | ファイル書き込み | `file_path` |
| `Edit` | ファイル編集 | `file_path`, `old_string`, `new_string` |
| `Bash` | コマンド実行 | `command`, `description`, `timeout` |
| `Glob` | ファイル検索 | `pattern`, `path` |
| `Grep` | コンテンツ検索 | `pattern`, `path`, `glob` |
| `Agent` | サブエージェント起動 | `subagent_type`, `description`, `model`, `run_in_background`, `isolation` |
| `Skill` | スキル実行 | `skill`, `args` |
| `WebFetch` | Web取得 | URL等 |
| `WebSearch` | Web検索 | クエリ等 |
| `AskUserQuestion` | ユーザーへの質問 | 質問内容 |
| `ToolSearch` | ツール検索 | `query` |
| `TodoWrite` | Todo管理 | タスク内容 |
| `NotebookEdit` | Notebook編集 | セル操作 |
| `TaskCreate` | タスク作成 | タスク定義 |
| `TaskUpdate` | タスク更新 | ステータス更新 |

### 3.2 Agent ツール詳細（sub agent分析用）

| パラメータ | 説明 | 分析用途 |
|---|---|---|
| `subagent_type` | エージェントタイプ（`Explore`, `Plan`, `general-purpose` 等） | sub agent種別分布 |
| `description` | タスク概要 | 作業内容分類 |
| `model` | モデル指定（`sonnet`, `opus`, `haiku`, 未指定=inherit） | モデル使い分け分析 |
| `run_in_background` | バックグラウンド実行か | 並列実行パターン |
| `isolation` | `"worktree"` で隔離実行 | 隔離実行の頻度 |

### 3.3 Skill ツール詳細（スキル分析用）

| パラメータ | 説明 |
|---|---|
| `skill` | スキル名（`commit`, `simplify`, `create-pr`, カスタムスキル等） |
| `args` | スキルへの引数 |

---

## 4. Teams データ詳細

### 4.1 config.json

| フィールド | 型 | 説明 |
|---|---|---|
| `name` | string | チーム名 |
| `description` | string | チームの目的・説明 |
| `createdAt` | number (epoch ms) | 作成日時 |
| `leadAgentId` | string | リーダーエージェントID |
| `leadSessionId` | string | リーダーセッションID |
| `members[]` | array | メンバー一覧（下表） |

### 4.2 members[]

| フィールド | 型 | 説明 |
|---|---|---|
| `agentId` | string | `{name}@{team-name}` |
| `name` | string | エージェント名（`team-lead`, `planner`, `explorer-xxx`等） |
| `agentType` | string | タイプ（`team-lead`, `Plan`, `Explore` 等） |
| `model` | string | 使用モデル（`claude-opus-4-6`, `inherit` 等） |
| `prompt` | string | エージェントに与えたプロンプト |
| `joinedAt` | number (epoch ms) | 参加日時 |
| `cwd` | string | 作業ディレクトリ |
| `backendType` | string | `"in-process"` 等 |

### 4.3 Inbox メッセージ

| フィールド | 型 | 説明 |
|---|---|---|
| `from` | string | 送信元エージェント名 |
| `text` | string | メッセージ本文 |
| `summary` | string | メッセージ要約 |
| `timestamp` | string (ISO8601) | 送信日時 |
| `read` | boolean | 既読フラグ |

---

## 5. コマンド履歴（history.jsonl）

| フィールド | 型 | 説明 |
|---|---|---|
| `display` | string | ユーザー入力コマンド（`/mcp`, `/clear`, テキスト等） |
| `timestamp` | number (epoch ms) | 入力日時 |
| `project` | string | プロジェクトパス |
| `sessionId` | string | セッションID（存在する場合） |
| `pastedContents` | object | ペースト内容 |

---

## 6. 統計キャッシュ（stats-cache.json）

| フィールド | 型 | 説明 |
|---|---|---|
| `version` | number | スキーマバージョン |
| `lastComputedDate` | string | 最終計算日 |
| `dailyActivity[]` | array | 日次統計（`date`, `messageCount`, `sessionCount`, `toolCallCount`） |
| `dailyModelTokens[]` | array | 日次モデル別token統計 |
| `modelUsage` | object | モデル別使用状況 |
| `totalSessions` | number | 総セッション数 |
| `totalMessages` | number | 総メッセージ数 |
| `longestSession` | object | 最長セッション情報（`sessionId`, `duration`, `messageCount`） |
| `firstSessionDate` | string | 初回セッション日時 |
| `hourCounts` | object | 時間帯別利用回数 |

---

## 7. デバッグログ（debug/*.txt）

テキスト形式のタイムスタンプ付きログ。以下の情報を含む。

| パターン | 取得できる情報 |
|---|---|
| `[onSubmit]` | ユーザー入力イベント |
| `hook` 関連 | Hook実行（PreToolUse, SessionStart, SessionEnd等） |
| `LSP Diagnostics` | LSP診断情報 |
| `AutoUpdaterWrapper` | インストールタイプ（npm-global等） |
| `claudeai-mcp` | MCPサーバー接続状況 |
| `Full reset` | 画面リセットイベント |

---

## 8. 実データからの観測値

参考として、実際のデータから観測された値。

### ツール利用頻度（上位）

| ツール | 用途 |
|---|---|
| `Read` | 最も頻繁（コード理解） |
| `Bash` | 2番目（コマンド実行） |
| `Glob` | ファイル検索 |
| `Edit` | コード編集 |
| `Write` | ファイル作成 |
| `Grep` | コンテンツ検索 |
| `Agent` | sub agent（全プロジェクトで計192回） |
| `WebFetch` | Web取得（1セッションで37回のケースあり） |
| `Skill` | スキル実行（計9回） |

### Sub Agent タイプ分布

| subagent_type | 用途 |
|---|---|
| `Explore` | 最も多い。コードベース調査 |
| `general-purpose` | 汎用タスク |
| `Plan` | 実装計画策定 |

### Sidechain メッセージ量

| 規模 | 説明 |
|---|---|
| 0 | sub agent未使用セッション |
| 30〜80 | 軽量なsub agent利用 |
| 100〜180 | ヘビーなsub agent利用 |

### Skill 利用

| skill名 | 回数 |
|---|---|
| `simplify` | 5 |
| `create-components-best-practice` | 3（カスタムスキル） |
| `create-pr` | 1 |

### スラッシュコマンド履歴（history.jsonl）

| コマンド | 回数 |
|---|---|
| `/mcp` | 15 |
| `/clear` | 13 |
| `/login` | 4 |
| `/context` | 4 |
| `/fast` | 3 |
| `/model` | 2 |
| `/review` | 2 |
