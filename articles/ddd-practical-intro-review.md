---
title: "「作りながら学ぶ！DDD実践入門」を読んで実装しながらまとめた"
emoji: "🏗️"
type: "tech"
topics: ["ddd", "typescript", "オニオンアーキテクチャ", "イベント駆動", "設計"]
published: false
---

## はじめに

「作りながら学ぶ！ドメイン駆動設計実践入門」を読みながら、書籍のコード例を実際に手を動かして実装した。

本記事はその読書まとめ＋感想。書籍で学んだ設計パターンを、実際のTypeScriptコードと対応させながら振り返る。

対象となる実装は **書籍カタログサービス**（書籍の登録・レビュー・推薦）という題材で、章を追うごとにアーキテクチャが積み上がっていく構成になっている。

---

## 書籍全体の構成

書籍は「なぜDDDが必要か」という問いから始まり、戦略設計・業務知識の獲得を経て、戦術設計の実装へと進む。最終的にはイベント駆動アーキテクチャ・イベントソーシング＋CQRSまで発展する。

```
【戦略設計】
DDDの基礎・必要性
    ↓
戦略的設計（コアドメイン・サブドメインの分類）
    ↓
イベントストーミング（業務知識の獲得・ユビキタス言語）
    ↓
ドメインモデルの可視化（集約・値オブジェクトの特定）

【戦術設計〜実装】
    ↓
値オブジェクト・エンティティ・集約
    ↓
リポジトリ・アプリケーションサービス
    ↓
プレゼンテーション層（Express）
    ↓
ESLint依存制約・DIコンテナ（tsyringe）

【イベント駆動】
    ↓
イベント駆動アーキテクチャ
    ↓
Outboxパターン
    ↓
イベントソーシング・CQRS
```

---

## 戦略設計：「何を作るか」を決める

実装の前に、**どの問題に集中すべきか**を決める章が序盤に置かれている。

**サブドメインの分類**では、ドメイン全体を3種類に分ける。

| 種類 | 説明 | 対応方針 |
|---|---|---|
| コアドメイン | 競争優位性の源泉 | 内製・DDD適用 |
| 支援サブドメイン | コアを補助する業務 | 内製 or 外注 |
| 汎用サブドメイン | 一般的な機能 | 既存SaaSで代替 |

本書の題材であるオンライン書店では「レビュー内容からの書籍推薦」がコアドメインに当たる。

**イベントストーミング**は、ドメインの業務知識をチームで共有・可視化するワークショップ手法。ドメインイベント（「書籍が登録された」「レビューが投稿された」）を付箋に書き出し、そこからコマンド・集約・ポリシーを特定していく。このプロセスで**ユビキタス言語**（開発者とビジネス側が共有する言葉）が形成される。

戦略設計の章を読んで、コードを書く前に「どこに力を入れるべきか」を整理する工程がDDDの出発点だということが分かった。実装に入る前にここで認識を揃えることで、後の戦術設計の判断（何を値オブジェクトにするか、どこが集約境界か）に迷いが少なくなる。

---

## 値オブジェクト：プリミティブ型への不満から始まる

最初に「なぜ `number` や `string` のままではいけないのか」という問いから始まる。

たとえば `rating: number` のままでは、1〜5の範囲チェックをアプリ中の至る所に書かなければならない。

```typescript
// ❌ プリミティブのまま
function updateRating(rating: number) {
  if (rating < 1 || rating > 5) throw new Error("...");
  // このチェックがあちこちに散らばる
}

// ✅ 値オブジェクト化
export class Rating extends ValueObject<number, "Rating"> {
  static readonly MAX = 5;
  static readonly MIN = 1;

  protected validate(value: number): void {
    if (!Number.isInteger(value)) throw new Error("評価は整数値でなければなりません。");
    if (value < Rating.MIN || value > Rating.MAX) {
      throw new Error(`評価は${Rating.MIN}から${Rating.MAX}までの整数値でなければなりません。`);
    }
  }
}
```

ビジネスルールが `Rating` クラスの中に閉じ込められ、**「Ratingはどこで使っても1〜5の整数が保証されている」** という事実がコードで表現される。

TypeScriptの構造的型付けに起因する型混同を防ぐため、`_type: U` によるブランド化もこの章で学んだ。`Rating` と `Price` が同じ `number` ベースでも別の型として扱われる。

---

## 集約：トランザクション境界を設計する

集約は「一緒に変わるべきオブジェクトのまとまり」であり、トランザクション境界と一致させる。

`Review` 集約は `ReviewId`・`BookId`・`Name`・`Rating`・`Comment` を内包し、集約ルートの `Review` を経由しないと内部状態を変更できない。

```typescript
export class Review {
  private constructor(
    private readonly _identity: ReviewIdentity,
    private readonly _bookId: BookId,
    private _name: Name,
    private _rating: Rating,
    private _comment?: Comment,
  ) {}

  static create(
    identity: ReviewIdentity,
    bookId: BookId,
    name: Name,
    rating: Rating,
    comment?: Comment,
  ): Review {
    return new Review(identity, bookId, name, rating, comment);
  }

  updateRating(rating: Rating): void {
    this._rating = rating;
  }
}
```

`private constructor` にして `static create()` ファクトリメソッドだけを公開するパターンは、生成時のバリデーションをかならず通過させるための定石として体に染み込んだ。

---

## アプリケーションサービス：「繋ぎ役」に徹する

アプリケーションサービスの章で最も印象に残ったのは「自分では何も計算・判断しない」という原則だ。

```typescript
// ❌ アプリケーションサービスにビジネスロジックが漏れている
async execute(command: RegisterBookCommand) {
  if (command.price < 0) throw new Error("価格は0以上...");
  if (command.isbn.length !== 13) throw new Error("ISBNは13桁...");
}

// ✅ ドメインオブジェクトに委譲
async execute(command: RegisterBookCommand) {
  const price = new Price({ amount: command.price, currency: "JPY" }); // Priceが検証
  const bookId = new BookId(command.isbn);                             // BookIdが検証
}
```

「アプリケーションサービスが太ってきたらドメイン層へ移す」という継続的なリファクタリングの感覚が身についた章でもある。

また、**DTOパターン** によってドメインオブジェクトを外に出さない意義も理解できた。`passwordHash` のような内部フィールドをうっかりAPIレスポンスに含めてしまうリスクをアーキテクチャレベルで防げる。

---

## ESLint依存制約：設計の崩壊を機械的に防ぐ

ESLintの `no-restricted-imports` を使えば、依存の方向違反をCIで自動検出できる。

```js
// ドメイン層がインフラ層をインポートしようとするとエラー
{
  files: ["src/Domain/**/*.ts"],
  rules: {
    "no-restricted-imports": ["error", {
      patterns: ["Application/*", "Infrastructure/*", "Presentation/*"]
    }]
  }
}
```

ESLintで書けるのか、便利だなと思った反面、Biomeにはこういう機能がないので使い分けが必要になる。「設計はドキュメントではなくツールで守る」という考え方自体は、レビューで指摘するのではなくコミットが通らない仕組みにするという点で気持ちよかった。

---

## DIコンテナ（tsyringe）：環境切り替えのコストをゼロに

手動DI（プレゼンテーション層でインスタンスを生成して注入する）では、依存が増えるたびにボイラープレートが肥大化する問題があった。tsyringe導入後は：

```typescript
// Before：手動でインスタンス化
const clientManager = new SQLClientManager();
const transactionManager = new SQLTransactionManager(clientManager);
const bookRepository = new SQLBookRepository(clientManager);
const service = new RegisterBookService(bookRepository, transactionManager);

// After：1行
const service = container.resolve(RegisterBookService);
```

テスト用コンテナを1つ用意するだけで、新しいテストを追加するたびに依存を手動で組む必要がなくなる点も実用的だった。

---

## イベント駆動アーキテクチャ

ドメインイベントをトリガーに別のコンテキストが非同期・疎結合に動く設計。集約はイベントを内部リストに蓄積し、アプリケーションサービスがトランザクション完了後にまとめて発行する。

```typescript
// アプリケーションサービス：トランザクション完了後に発行
const review = await this.transactionManager.begin(async () => {
  await this.reviewRepository.save(review); // ① 保存
  return review;
});

for (const event of review.getDomainEvents()) {
  this.domainEventPublisher.publish(event); // ② 発行（保存成功後）
}
```

保存が失敗したのにイベントが飛ぶ事故を防ぐための「蓄積して後で発行」という設計になっている。

---

## Outboxパターン：それでも残る「発行失敗」問題

ただし、上記の実装には抜け穴がある。①の保存が成功しても、②の発行がネットワーク障害やプロセスクラッシュで失敗するケースがある。

Outboxパターンはこれを「同一トランザクション内でoutboxテーブルにも書く」ことで解消する。

```typescript
// 保存とイベント記録を同一トランザクションでアトミックに
await this.transactionManager.begin(async () => {
  await this.reviewRepository.save(review);         // ビジネスデータ
  await this.outboxEventRepository.save(event);     // イベントを記録（同時にコミットされる）
});
// トランザクション外ではイベントを直接発行しない
```

別プロセス（メッセージリレー）がoutboxテーブルをポーリングしてブローカへ転送する。実装コストは上がるが、**「保存されたなら必ずイベントが届く」** という保証が得られる。

---

## イベントソーシング：過去を捨てない設計

最終章に向けて登場するイベントソーシングは、それまでの「状態を保存する」発想を根本から変える。

| アプローチ | 保存するもの |
|---|---|
| ステートベース | 現在の状態（`UPDATE`で上書き） |
| イベントソーシング | 状態を変化させたイベントの連続（追記のみ） |

「6ヶ月前と比べて推薦書籍からの売り上げが30%低下した。信頼性の低いレビューが悪影響しているのではないか」という仮説を検証したいとする。ステートベースでは現在の評価しか残っていないが、イベントソーシングなら `ReviewRatingUpdated` の履歴から「誰がいつ評価を変えたか」を追跡できる。

Outboxパターンで発生していた「集約の状態とイベントの二重管理」も、イベントを唯一の情報源とすることで解消される。

---

## CQRS：書き込みと読み取りを分離する

イベントソーシングで困るのが集約をまたぐ検索。`findAllByBookId` のようなクエリは、`aggregate_id`（= `reviewId`）でインデックスされたイベントストアとは相性が悪い。

CQRSはこれをCommand（書き込み）とQuery（読み取り）のモデルを分離することで解決する。

```
Command サイド：イベントストアへ追記（整合性保証）
    ↓ ReviewCreated を発行
Query サイド：Read Modelを更新（クエリ最適化）
    └─ review_read_model テーブルに bookId インデックスつきで保持
```

Read Modelはイベントを再生すれば**いつでも再構築可能**なため、クエリ要件が変わっても対応できる。

---

## 読んで感じたこと

### よかった点

**「なぜそう設計するのか」の動機が丁寧**
値オブジェクトもDIコンテナもイベントソーシングも、「何が嫌だったから」という問いから説明が始まる。概念を先に押し付けてくる本ではなく、問題を先に提示してから解決策として設計パターンが登場する構成が腹落ちしやすかった。

**実装コードが一本のシステムで積み上がる**
1章ごとに独立した例ではなく、書籍カタログサービスという一つのシステムに設計が積み重なっていく。前章で実装したコードに後章の概念が接続される感覚があって、「使いどころ」が実感できた。

**いつ使うか・使わないかを明示してくれる**
イベント駆動もイベントソーシングも「このシステムには向いている/向いていない」という判断基準をセットで書いてくれている。設計パターンは使い方より「使い時の判断」が難しいと思っているので、これは助かった。

### 難しかった点

**CQRSとイベントソーシングの組み合わせは複雑**
最終章に向かうにつれてアーキテクチャの複雑度が増す。イベントソーシング単体、CQRS単体はわかっても、両者を組み合わせたデータフローを頭に入れるのには時間がかかった。図を自分で書いて理解した。

**スキーマ変更・バージョニングは本書の範囲外**
イベントソーシングの本番運用で問題になる「保存済みイベントのスキーマを変えたい」ケースはほとんど触れられていない。実運用に乗せるにはこのあたりの戦略が別途必要だと感じた。

---

## 類書との比較

同ジャンルの本として「ドメイン駆動設計入門 ボトムアップでわかる！ドメイン駆動設計の基本」も良書として挙げられる。あちらはDDDの戦術設計（値オブジェクト・エンティティ・リポジトリ等）をわかりやすく解説しており、入門書として優れている。

一方で本書との違いとして感じたのは次の点だ。

- **戦略設計・イベント駆動設計の有無**：「ドメイン駆動設計入門」は戦術設計に集中しており、境界付けられたコンテキストの連携やイベント駆動アーキテクチャまでは踏み込まない。本書はそこまでカバーしている。
- **「なぜそう設計するのか」の丁寧さ**：本書は問題提起→解決策の流れが一貫していて、設計パターンを導入する動機が腹落ちしやすかった。

DDD自体が初めてであれば「ドメイン駆動設計入門」から入り、実装を積み上げながらアーキテクチャ全体を学びたくなったら本書、という順番がよさそうだと感じた。

---

## まとめ

「設計パターンの名前は知っているが使いどころがわからない」という状態に刺さる一冊だった。

書籍のコードをそのまま動かすだけでなく、各章の内容をMarkdownにまとめる作業を並行したことで理解が定着した。実装しながら読む系の本はこの進め方がおすすめ。
