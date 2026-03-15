---
name: react-best-practice
description: React / Next.js (App Router) のコードを書く・レビューする際に、ベストプラクティスに沿っているか確認し、改善提案を行う。コンポーネント設計、状態管理、パフォーマンス最適化、アクセシビリティなどを網羅。
user-invocable: true
---

# React Best Practices

このプロジェクト（Next.js App Router + TypeScript + yamada-ui）における React のベストプラクティス。

---

## 1. コンポーネント設計

### Server Components をデフォルトにする

- Next.js App Router では **Server Component がデフォルト**。クライアント側の機能が必要な場合のみ `"use client"` を付与する。
- `"use client"` が必要なケース: `useState`, `useEffect`, `useRef`, イベントハンドラ (`onClick` 等), ブラウザ API, yamada-ui のインタラクティブコンポーネント。

### Client Component の境界を最小化する

- ページ全体を `"use client"` にせず、インタラクティブな部分だけを小さな Client Component に切り出す。

```tsx
// Good: インタラクティブ部分だけ Client Component
// app/dashboard/page.tsx (Server Component)
import { InteractiveChart } from "./interactive-chart";

export default async function DashboardPage() {
  const data = await fetchData(); // サーバーで取得
  return (
    <div>
      <h1>Dashboard</h1>
      <InteractiveChart data={data} />
    </div>
  );
}
```

### 単一責任の原則

- 1 コンポーネント = 1 つの責務。表示・ロジック・データ取得を適切に分離する。
- 200 行を超えるコンポーネントは分割を検討する。

### Props の設計

- Props の型は明示的に定義する。`any` は禁止。
- children を活用した合成パターンを優先し、prop drilling を避ける。
- boolean props は肯定形にする（`isDisabled` ではなく `disabled`、ただし yamada-ui の API に従う場合は除く）。

```tsx
// Good
type ChartCardProps = {
  title: string;
  data: DataPoint[];
  children?: React.ReactNode;
};

// Bad
type ChartCardProps = {
  title: string;
  data: any;
  isNotHidden?: boolean;
};
```

---

## 2. 状態管理

### 状態の置き場所を正しく選ぶ

1. **URL (searchParams)** — フィルタ、ページネーション、ソート順など共有可能な状態
2. **Server State** — DB / API からのデータ（Server Component で fetch）
3. **Local State (`useState`)** — UI のトグル、フォーム入力など
4. **Context** — テーマ、認証情報など複数コンポーネントで共有する状態

### 不要な状態を作らない

- 既存の state や props から**計算できる値**は state にしない。

```tsx
// Bad
const [items, setItems] = useState<Item[]>([]);
const [count, setCount] = useState(0);
// items が変わるたびに setCount(items.length) を呼ぶ…

// Good
const [items, setItems] = useState<Item[]>([]);
const count = items.length; // 派生値
```

### `useReducer` の活用

- 関連する複数の state がある場合は `useReducer` を検討する。

---

## 3. パフォーマンス最適化

### 不要な再レンダリングを防ぐ

useMemo,useCallback は基本的に不要です．react compiler がついています．

- コンポーネントの分割で解決できないか先に検討する。

### リストレンダリング

- `key` には安定した一意の ID を使う。配列の index は**並べ替え・追加・削除がない場合のみ**許容。

```tsx
// Good
{
  users.map((user) => <UserCard key={user.id} user={user} />);
}

// Bad
{
  users.map((user, index) => <UserCard key={index} user={user} />);
}
```

### 画像の最適化

- `next/image` を使用する。`width`, `height` を必ず指定する。

### 動的インポート

- 初期表示に不要な重いコンポーネントは `next/dynamic` で遅延ロードする。

```tsx
import dynamic from "next/dynamic";

const HeavyChart = dynamic(() => import("./heavy-chart"), {
  loading: () => <Skeleton height="300px" />,
});
```

---

## 4. データ取得パターン (Next.js App Router)

### Server Component でのデータ取得

- `fetch` や DB クエリは Server Component 内で直接行う。
- `async/await` をそのまま使える。

```tsx
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const stats = await db.select().from(sessions);
  return <StatsDisplay stats={stats} />;
}
```

### Server Actions

- フォーム送信やミューテーションには Server Actions を使う。
- `"use server"` ディレクティブを関数またはファイルの先頭に記述する。
- 入力値は必ず zod でバリデーションする。

```tsx
"use server";

import { z } from "zod";

const schema = z.object({
  name: z.string().min(1),
});

export async function createItem(formData: FormData) {
  const parsed = schema.safeParse({
    name: formData.get("name"),
  });
  if (!parsed.success) {
    return { error: parsed.error.flatten() };
  }
  // DB 操作
}
```

### loading.tsx / error.tsx

- 各ルートセグメントに `loading.tsx` と `error.tsx` を配置して、Suspense / Error Boundary を活用する。

---

## 5. Hooks のルール

### カスタム Hooks

- 再利用するロジックは `use` プレフィックス付きのカスタム Hook に切り出す。
- カスタム Hook は **1 つの関心事** に集中させる。

```tsx
function useSessionStats(userId: string) {
  const [stats, setStats] = useState<Stats | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    // fetch logic
  }, [userId]);

  return { stats, isLoading };
}
```

### useEffect の注意点

- **useEffect はイベントハンドラで代替できないか**まず検討する。
- 依存配列は正確に記述する。ESLint の `exhaustive-deps` ルールに従う。
- クリーンアップ関数を忘れない（タイマー、サブスクリプション等）。

---

## 6. TypeScript との統合

### 型の付け方

- コンポーネントの型は関数の引数に直接記述する。`React.FC` は使わない。

```tsx
// Good
function UserCard({ name, score }: { name: string; score: number }) {
  return (
    <div>
      {name}: {score}
    </div>
  );
}

// Good (型が複雑な場合)
type UserCardProps = {
  name: string;
  score: number;
  onSelect?: (id: string) => void;
};

function UserCard({ name, score, onSelect }: UserCardProps) {
  return (
    <div>
      {name}: {score}
    </div>
  );
}
```

### イベントハンドラの型

- `React.MouseEvent<HTMLButtonElement>` など具体的な型を使う。
- インラインハンドラなら型推論に任せて OK。

### as / ! の使用禁止

- `as` による型アサーションや `!` (non-null assertion) は原則禁止。型ガードや zod でのパースで安全に絞り込む。

---

## 7. アクセシビリティ (a11y)

- セマンティックな HTML 要素を使う（`<button>`, `<nav>`, `<main>` 等）。`<div onClick>` は禁止。
- 画像には `alt` テキストを設定する。装飾画像は `alt=""` にする。
- yamada-ui のコンポーネントは a11y が組み込まれているのでそのまま活用する。
- フォーム要素には `<label>` を関連付ける。yamada-ui の `FormControl` を使う。
- キーボード操作を考慮する。フォーカス管理を適切に行う。

---

## 8. エラーハンドリング

- Error Boundary (`error.tsx`) でクラッシュを防ぐ。
- ユーザー向けのエラーメッセージは具体的かつ対処法を示す。
- API からのエラーは型安全にハンドリングする。

```tsx
// app/dashboard/error.tsx
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>エラーが発生しました</h2>
      <button onClick={reset}>再試行</button>
    </div>
  );
}
```

### 命名規則

- ファイル名: **kebab-case** (`stats-card.tsx`)
- コンポーネント名: **PascalCase** (`StatsCard`)
- Hooks: **camelCase** with `use` prefix (`useSessionStats`)
- 型名: **PascalCase** (`SessionStats`)
