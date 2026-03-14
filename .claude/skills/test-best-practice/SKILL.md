---
name: test-best-practice
description: テストコード（Vitest / Playwright）を書く・レビューする際に、ベストプラクティスに沿っているか確認し、改善提案を行う。AAA パターン、本質的なテストの書き方、ロジックテストとコンポーネントテストの設計指針を網羅。
user-invocable: true
---

# Test Best Practices

このプロジェクト（Vitest + Testing Library + Playwright）におけるテストのベストプラクティス。

---

## 1. AAA パターン (Arrange / Act / Assert)

すべてのテストは **AAA パターン** で構造化する。各セクションは空行で視覚的に区切る。

```tsx
test("スコアが閾値を超えたらランクが上がる", () => {
  // Arrange — テストの前提条件を準備する
  const user = createUser({ score: 99, rank: "silver" });

  // Act — テスト対象の操作を1つだけ実行する
  const result = promoteIfEligible(user);

  // Assert — 期待する結果を検証する
  expect(result.rank).toBe("gold");
});
```

### Arrange
- テストに必要なデータ・状態・モックを準備する。
- ファクトリ関数やヘルパーを活用して簡潔に書く。
- テストに無関係なフィールドはデフォルト値に任せる。

### Act
- **1テスト = 1アクション**。テスト対象の操作は1つだけにする。
- Act が複数行になる場合、テストの粒度が大きすぎる可能性がある。

### Assert
- **1テスト = 1つの関心事** を検証する。`expect` が5個以上あるなら分割を検討する。
- アサーションは具体的に書く。`toBeTruthy()` より `toBe(true)` や `toHaveLength(3)` を使う。

---

## 2. 本質的なテストを書く

### テストすべきもの / テストすべきでないもの

| テストする | テストしない |
|-----------|-------------|
| ビジネスロジックの振る舞い | フレームワークの内部実装 |
| 境界値・エッジケース | 定数の値 |
| エラーケース | private メソッドの個別テスト |
| ユーザーから見た振る舞い | 実装の詳細（state の値、内部関数の呼出し回数） |

### ロジックテスト — 「どんな入力に対してどんな出力を返すか」

純粋関数やビジネスロジックは **入力 → 出力** の関係をテストする。

```tsx
// Good: 振る舞いをテストしている
test("日付が同じ月なら同一週としてグループ化される", () => {
  const sessions = [
    createSession({ date: "2026-03-01" }),
    createSession({ date: "2026-03-07" }),
  ];

  const grouped = groupByWeek(sessions);

  expect(grouped).toHaveLength(1);
  expect(grouped[0].sessions).toHaveLength(2);
});

// Bad: 実装の詳細をテストしている
test("groupByWeek が getWeekNumber を呼ぶ", () => {
  const spy = vi.spyOn(dateUtils, "getWeekNumber");
  groupByWeek(sessions);
  expect(spy).toHaveBeenCalledTimes(2); // 実装が変わると壊れる
});
```

### コンポーネントテスト — 「何をしたら何が起きるか」

コンポーネントは **ユーザー操作 → 画面の変化** をテストする。

```tsx
// Good: ユーザー視点でテストしている
test("フィルタを選択すると該当するデータのみ表示される", () => {
  // Arrange
  render(<Dashboard data={mockData} />);

  // Act
  await userEvent.selectOptions(
    screen.getByRole("combobox", { name: "期間" }),
    "今週",
  );

  // Assert
  expect(screen.getByText("セッション: 5")).toBeInTheDocument();
  expect(screen.queryByText("セッション: 120")).not.toBeInTheDocument();
});

// Bad: 実装の詳細をテストしている
test("フィルタ選択で setState が呼ばれる", () => {
  const setState = vi.fn();
  vi.spyOn(React, "useState").mockReturnValue(["今週", setState]);
  render(<Dashboard data={mockData} />);
  fireEvent.change(select, { target: { value: "今週" } });
  expect(setState).toHaveBeenCalledWith("今週"); // 内部実装に依存
});
```

---

## 3. テストの命名

テスト名は **条件と期待結果** を日本語で明確に記述する。

```tsx
// Good: 何が起きるか分かる
test("トークン使用量が上限を超えたら警告バッジが表示される", () => {});
test("未認証ユーザーはダッシュボードにアクセスできない", () => {});
test("空のデータセットでは「データなし」メッセージが表示される", () => {});

// Bad: 曖昧・技術用語だけ
test("badge works", () => {});
test("test auth", () => {});
test("handles empty", () => {});
```

### describe でグルーピング

```tsx
describe("TokenUsageCard", () => {
  describe("使用量が上限以内の場合", () => {
    test("通常のバッジが表示される", () => {});
    test("プログレスバーが緑色になる", () => {});
  });

  describe("使用量が上限を超えた場合", () => {
    test("警告バッジが表示される", () => {});
    test("プログレスバーが赤色になる", () => {});
  });
});
```

---

## 4. Testing Library のクエリ優先順位

要素の取得は **ユーザーがアクセスする方法に近い順** で選ぶ。

| 優先度 | クエリ | 用途 |
|--------|--------|------|
| 1 | `getByRole` | ボタン、リンク、テキストボックス等 |
| 2 | `getByLabelText` | フォーム要素 |
| 3 | `getByPlaceholderText` | label がない場合 |
| 4 | `getByText` | 表示テキストで取得 |
| 5 | `getByTestId` | **最終手段**。他で取得できない場合のみ |

```tsx
// Good
screen.getByRole("button", { name: "送信" });
screen.getByLabelText("メールアドレス");

// Bad
screen.getByTestId("submit-button");
document.querySelector(".btn-primary");
```

---

## 5. モック・スタブの原則

### モックは最小限にする
- 外部依存（API、DB、タイマー）のみモックする。内部モジュールのモックは避ける。
- モックが多いテストは設計の問題を示している可能性がある。

### モックの配置
- 共通のモックは `beforeEach` で設定し、`afterEach` で `vi.restoreAllMocks()` する。
- テスト固有のモックは各テストの Arrange で設定する。

```tsx
beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
  vi.restoreAllMocks();
});
```

### MSW で API をモックする
- `fetch` や `axios` を直接モックせず、MSW (Mock Service Worker) でネットワークレベルでインターセプトする。

---

## 6. テストデータ

### ファクトリ関数を使う
- テストデータはファクトリ関数で生成する。テストに関係するフィールドだけ上書きする。

```tsx
function createSession(overrides: Partial<Session> = {}): Session {
  return {
    id: crypto.randomUUID(),
    userId: "user-1",
    startedAt: new Date("2026-03-15T09:00:00"),
    messageCount: 10,
    tokenUsage: 1500,
    ...overrides,
  };
}

// 使用例: テストに関係する部分だけ明示
test("メッセージ数が0のセッションは非アクティブとみなされる", () => {
  const session = createSession({ messageCount: 0 });
  expect(isActive(session)).toBe(false);
});
```

### マジックナンバーを避ける
- テストデータの値には意図を持たせる。なぜその値なのか分かるようにする。

```tsx
// Good: 意図が明確
const THRESHOLD = 1000;
const session = createSession({ tokenUsage: THRESHOLD + 1 });

// Bad: なぜ 1001 なのか分からない
const session = createSession({ tokenUsage: 1001 });
```

---

## 7. E2E テスト (Playwright)

### ユーザーシナリオ単位でテストする
- E2E は**ユーザーの操作フロー全体**をテストする。単体テストの延長にしない。

```tsx
test("ユーザーがログインしてダッシュボードの統計を確認できる", async ({ page }) => {
  // Arrange
  await page.goto("/login");

  // Act
  await page.getByLabel("メールアドレス").fill("test@example.com");
  await page.getByLabel("パスワード").fill("password");
  await page.getByRole("button", { name: "ログイン" }).click();

  // Assert
  await expect(page).toHaveURL("/dashboard");
  await expect(page.getByRole("heading", { name: "ダッシュボード" })).toBeVisible();
  await expect(page.getByText("セッション数")).toBeVisible();
});
```

### E2E テストの注意点
- テスト間の依存を排除する。各テストは独立して実行可能にする。
- セレクタは `getByRole`, `getByLabel`, `getByText` を優先する。CSS セレクタは避ける。
- `waitForSelector` より `expect(...).toBeVisible()` のような自動リトライ付きアサーションを使う。

---

## 8. アンチパターン

### やってはいけないこと

```tsx
// 1. テスト間の状態共有
let sharedData: Data; // テスト間で共有すると順序依存が生まれる

// 2. 条件分岐のあるテスト
test("データの表示", () => {
  if (featureFlag) { /* ... */ } // テスト内で分岐しない。別テストに分ける
});

// 3. sleep による待機
await new Promise((r) => setTimeout(r, 1000)); // Playwright の自動待機を使う

// 4. スナップショットテストの乱用
expect(component).toMatchSnapshot(); // 変更のたびに壊れる。振る舞いをテストする

// 5. テストの中でテスト対象のロジックを再実装
test("合計を計算する", () => {
  const items = [{ price: 100 }, { price: 200 }];
  const expected = items.reduce((sum, i) => sum + i.price, 0); // ロジックの再実装
  expect(calcTotal(items)).toBe(expected); // これでは何もテストしていない
  expect(calcTotal(items)).toBe(300); // 具体的な値で検証する
});
```
