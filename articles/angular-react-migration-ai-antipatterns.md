---
title: "AIエージェントにAngular→React移行させて気づいたアンチパターン集"
emoji: "🔍"
type: "tech"
topics: ["react", "typescript", "angular", "ai", "migration"]
published: false
---

## はじめに

業務でAngular 12からReact 18 + Viteへの移行プロジェクトを担当した。規模が大きかったため、AIエージェントを活用して進めることにした。

ワークフローはこうだ。

1. 画面（または画面内の機能単位）でIssueを作成する
2. AIに移行計画を立てさせる
3. 人間がレビューしてOKを出す
4. AIに実装させる
5. 人間がコードレビューする

サブエージェントは**設計者・開発者・テスター**の3役割に分けた。特定のコードスタイルを強制するプロンプトは与えていない。

実装レビューを繰り返す中で、同じようなアンチパターンが繰り返し登場することに気づいた。本記事ではその内容をまとめる。

対象読者はAngularからReactに移行している人だけでなく、**AIエージェントを使ってReactを書いている人全般**を想定している。

---

## なぜAIは同じアンチパターンを出すのか

考えられる理由の一つは、「動くコードを生成すること」に最適化されやすい点だ。

プロジェクトの設計方針や「このパターンは避けてほしい」という暗黙のルールは、明示しない限り反映されにくい。また、学習データには古いStackOverflowの回答やアンチパターンを含むブログ記事も存在するため、React Hooksのような比較的新しい概念では古い書き方が出てくることがある。Angular移行の文脈では特に、Hooksに相当するAngularのパターンがないため、なんとなく動くコードを生成しやすいのではないかと推測している。

以降では実際に繰り返し観測されたアンチパターンを紹介する。

---

## 状態管理のアンチパターン

### 1. Props → useEffect → State の二重管理

最も頻繁に出たパターンがこれだ。

```tsx
// ❌ 悪い例
function Child({ value }: { value: string }) {
  const [localValue, setLocalValue] = useState(value);

  useEffect(() => {
    setLocalValue(value);
  }, [value]);

  return <span>{localValue}</span>;
}
```

`props` と `state` は更新タイミングが異なる。親が新しい値で再レンダリングされると props は即座に新しい値になるが、`useEffect` はレンダー・DOM反映の後に実行されるため、その間 props と state が一時的に別の値を持つ瞬間が生まれる。

```
1. 親が value="new" で再レンダリング
2. 子が再レンダリング → value（props）= "new"、localValue（state）= "old"  ← ズレ
3. DOMに反映
4. useEffect が実行 → setLocalValue("new")
5. 子が再レンダリング → やっと同期
```

状態の源泉が2か所になることでバグの温床になるうえ、同期のために余分な再レンダリングが毎回1回発生する。

```tsx
// ✅ 正しい対応
// 単なる表示なら props をそのまま使う
function Child({ value }: { value: string }) {
  return <span>{value}</span>;
}

// 子が独自に変更を持つ必要があるなら、初期値としてのみ使う
function EditableChild({ initialValue }: { initialValue: string }) {
  const [value, setValue] = useState(() => initialValue);
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```

Reactの公式ドキュメントでも「[You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)」として説明されているパターンだ。

---

### 2. useState と useRef の二重管理

```tsx
// ❌ 悪い例
const [count, setCount] = useState(0);
const countRef = useRef(0);

const handleClick = () => {
  setCount(c => c + 1);
  countRef.current += 1; // 両方を手動で同期
};
```

更新漏れによる同期ズレが発生し、「どちらが真の値か」が不明になる。

使い分けのルールはシンプルだ。

| 用途 | 使うもの |
|------|---------|
| レンダリングに反映する値 | `useState` |
| レンダリング不要な値（タイマーID、前回値など） | `useRef` |
| 同じ値を両方で持つ | **NG** |

---

## コード品質のアンチパターン

### 3. 既存の共通コンポーネントを使わない

プロジェクトに `<Button>`・`<Dialog>`・`<Input>` などのUIコンポーネントが整備されているにもかかわらず、素の `<button>` や `<div>` で実装するケースが出た。

原因は単純で、AIにコンテキストとして「このプロジェクトの共通コンポーネント一覧」を渡していなかったことだ。

デザイン・挙動の一貫性が崩れ、共通コンポーネントに加えた修正が全体に反映されなくなる。

**対応策：** 実装前に `src/components/ui/` 配下を確認させるステップをプロンプトに明示する。

---

### 4. disabled で弾いているのに handleSubmit でも弾く

```tsx
// ❌ 悪い例
const isDisabled = !inputValue || hasError;

const handleSubmit = () => {
  if (!inputValue) {
    setError("入力してください"); // isDisabled が true の時点でボタンは押せない
    return;
  }
  submit();
};
```

`isDisabled` が `true` のときボタンは押せないため、`handleSubmit` 内の `!inputValue` チェックには到達しない。

```tsx
// ✅ シンプルにする
const isDisabled = !inputValue || hasError;

const handleSubmit = () => {
  submit(); // disabled で弾いているので二重チェック不要
};
```

一見防御的に見えるが、将来 `isDisabled` の条件だけ変更したときに `handleSubmit` 内のチェックと乖離し、意図が読めなくなる。UIレベルで弾いている条件はハンドラ内に書かない。

---

### 5. メモ化の乱用（useMemo / useCallback / React.memo）

```tsx
// ❌ 悪い例：軽い処理にメモ化を適用している
const label = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);

const handleClick = useCallback(() => {
  setOpen(true);
}, []);
```

`useMemo` / `useCallback` はキャッシュのための比較コストを持つ。処理が軽ければ素直に計算した方が速い。`React.memo` も props の浅い比較を毎回行うため、頻繁に更新される子では意味がなくなる。

さらに依存配列の管理ミスで古い値を参照するバグが生まれやすく、コードの見通しも悪くなる。

**使うべき状況：**

- `useMemo`：ループや重い計算（ソート・フィルタ・集計）を含む処理
- `useCallback`：`React.memo` でラップされた子コンポーネントに渡す関数
- `React.memo`：プロファイラで再レンダリングがボトルネックと確認できた場合

> 「遅い」と計測で確認してから最適化する。最初からメモ化しない。

---

## テストのアンチパターン

### 6. 全ファイルにテストを書く

AIはファイルが存在すると、そのファイルに対するテストを生成しようとする。`page`・`component`・`hooks`・`utils` のすべてに対してテストが出てくるが、「その層にテストが必要か」という判断は苦手だ。

問題は、テストを書きすぎると実装の変更のたびにテストが壊れ、保守コストが跳ね上がることだ。層ごとのトレードオフを理解した上で、どこにテストを置くかを判断する必要がある。

**書く価値が高いテスト**

- **pageのコンポーネントテスト**：ユーザーストーリーをそのまま検証できる。送信フローや画面遷移など、「ユーザーが何をするか」を起点に書く
- **複雑な表示ロジックを持つcomponentのテスト**：条件による表示/非表示の出し分けが複雑なら単体で検証する価値がある
- **純粋な変換ロジックのテスト**：ソート・フィルタ・計算などのロジックはhooksやcomponentに混ぜず、別ディレクトリに純粋関数として切り出してテストする

```ts
// ✅ ロジックは src/lib/ などに純粋関数として置く
export function filterUnread(notifications: Notification[]) {
  return notifications.filter(n => !n.read);
}

it("未読のみ返す", () => {
  expect(filterUnread(notifications)).toHaveLength(2);
});
```

```tsx
// ✅ ユーザーストーリーは page のテストで検証する
it("フォームを入力して送信するとAPIが呼ばれる", async () => {
  render(<FormPage />);
  await userEvent.type(screen.getByLabelText("名前"), "田中");
  await userEvent.click(screen.getByRole("button", { name: "送信" }));
  expect(mockApi).toHaveBeenCalledWith({ name: "田中" });
});
```

**省略できることが多いテスト**

- **hooks単体テスト**：ロジックが純粋関数に切り出されていれば不要になるケースが多い。単純なAPIコールやstateの更新はpageのテストでカバーできる
- **シンプルなcomponentのテスト**：propsを受け取って表示するだけのコンポーネントはpageのテストで十分

書きすぎたテストは変更への抵抗になる。「テストがあるから安全」ではなく、「何のためのテストか」を問いながら書く層を選ぶことが、AIに任せるときに特に意識すべき点だ。

---

## 設計の考えどころ：Angular DI vs React Context

Prop drilling（propsのバケツリレー）はよく知られたアンチパターンだ。

```tsx
// ❌ 中間コンポーネントが関係ない props を経由する
<Page userId={userId}>
  <Layout userId={userId}>
    <Sidebar userId={userId}>
      <UserMenu userId={userId}>
        <Button userId={userId} /> {/* ここでしか使わない */}
```

解決策としてContextやJotaiなどのグローバルステートが挙げられる。しかし、子コンポーネントがContextやatomを直接参照することに違和感を覚える場合がある。子が「どこから値が来るか」という実装詳細に依存するためだ。

Angularと対比すると整理しやすい。

AngularはDIコンテナがサービスを管理するため、「ContextやJotaiへの直接依存」という問題はない。しかしよく考えると、AngularのコンポーネントもServiceクラスを直接知っている。

```typescript
@Component({...})
export class UserMenuComponent {
  constructor(private userService: UserService) {} // UserService を知っている
}
```

`UserService` を別の実装に差し替えたいとき、コンポーネントはその事実を知らずに済む――という意味ではAngularのDIは優れている。しかし「コンポーネントがServiceという存在に依存している」という構造自体は変わらない。汎用コンポーネントにドメイン固有のServiceを注入すれば、再利用しにくくなるのはReactのContextと同じだ。

ReactのCustom Hookは、AngularのServiceに相当する抽象化だと言える。どちらも「コンポーネントが依存する先」の名前が違うだけで、本質的な問題は同じだ。

現実的な落とし所は**コンポーネントの性質で使い分ける**ことだ。

| コンポーネント種別 | 状態参照 |
|-----------------|---------|
| `src/components/ui/Button` など汎用UI | propsのみ受け取る |
| `src/features/user/UserMenu` などfeature固有 | Context/Jotai直参照OK |

汎用コンポーネントはどのコンテキストでも使える必要があるのでpropsのみにする。feature固有のコンポーネントは再利用性よりも凝集度を優先してよい。

Angular移行の文脈では、この設計判断が最も議論が生まれる箇所だった。

---

## React Doctorで自動検知できるか

[React Doctor](https://github.com/millionco/react-doctor) というCLIツールが気になっている。60以上のルールでReactコードを静的解析し、0〜100のスコアを出力する。AIエージェントのSkillとして組み込める設計になっており、Claude CodeやCursorとの統合も想定されている。

本記事で紹介したアンチパターンのうち、「Props→useEffect→State」「デッドコード」「メモ化乱用の一部」は検知できる可能性がある。一方、「既存コンポーネントを使わない」「useState+useRef二重管理」はプロジェクト固有の知識が必要なため静的解析では難しいだろう。

まだ試せていないが、CIに組み込んでAI実装後の差分スキャンとして使うのは有望そうだ。

---

## まとめ

AIエージェントを使った移行で繰り返し出たアンチパターンを6つ紹介した。

| # | パターン | 防ぐ方法 |
|---|---------|---------|
| 1 | Props→useEffect→State二重管理 | プロンプトで明示 or レビュー |
| 2 | useState+useRef二重管理 | レビュー |
| 3 | 既存コンポーネントを使わない | 実装前にコンポーネント一覧を渡す |
| 4 | 到達不能なバリデーション | レビュー |
| 5 | メモ化の乱用 | プロンプトで「計測前はメモ化しない」と指定 |
| 6 | 実装詳細のテスト | レビュー |

プロンプトで防げるものと、人間のレビューが必要なものが混在している。AIに任せれば人間のレビューが不要になるわけではなく、**レビューの観点が変わる**のだと実感している。

Angular移行特有の話として、prop drillingを避けようとするとContext/Jotaiへの直接依存が生まれるというジレンマがある。AngularのDIも本質的には同じ構造で、依存先がServiceクラスに変わるだけだ。どちらのフレームワークでも「汎用コンポーネントはpropsのみ、feature固有はOK」という割り切りが現実的な落とし所になる。これはプロンプトで解決できる問題ではなく、チームで方針を決めるべき設計判断だ。
