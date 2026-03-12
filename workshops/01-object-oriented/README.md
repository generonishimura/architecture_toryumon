# 第1回：オブジェクト指向

> 変更に強いコードの設計原則を学ぶ

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | TypeScriptの基本文法（クラス、インターフェース、型） |
| 到達目標 | SOLID原則を理解し、責務の分離と依存の制御を意識した設計ができる |

## この回のキーメッセージ

> 「変更に強いコードは、責務が明確で依存が制御されている」

## 講義内容

### 1. オブジェクト指向の本質（15分）

- データと振る舞いをまとめる意味
- カプセル化 — 内部状態を隠蔽し、インターフェースで操作する
- ポリモーフィズム — 同じインターフェースで異なる振る舞いを実現する
- 「継承よりもコンポジション」という現代的な考え方

#### カプセル化の具体例

カプセル化とは「内部の実装詳細を隠し、外部には安全な操作だけを公開する」ことです。

```typescript
// Bad: 内部状態が丸見え — 不正な状態を許してしまう
class BankAccount {
  public balance: number = 0;
}

const account = new BankAccount();
account.balance = -1000; // 残高がマイナスに！ビジネスルール違反

// Good: カプセル化によってビジネスルールを守る
class BankAccount {
  private _balance: number = 0;

  get balance(): number {
    return this._balance;
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error('入金額は正の数である必要があります');
    this._balance += amount;
  }

  withdraw(amount: number): void {
    if (amount <= 0) throw new Error('出金額は正の数である必要があります');
    if (amount > this._balance) throw new Error('残高不足です');
    this._balance -= amount;
  }
}
```

カプセル化のポイントは「クラスの利用者がルールを知らなくても、不正な状態にならない」こと。`balance` を直接変更させず、`deposit()` / `withdraw()` 経由でのみ操作させることで、残高がマイナスになるバグを**構造的に**防げます。

#### 「継承よりコンポジション」の実例

継承は強力ですが、親子関係が密結合を生みます。TypeScriptではコンポジション（委譲）で柔軟性を保つことが推奨されます。

```typescript
// 継承の問題: FlyingSwimmingAnimal を作れない（多重継承不可）
class Animal { move(): void { /* ... */ } }
class FlyingAnimal extends Animal { fly(): void { /* ... */ } }
class SwimmingAnimal extends Animal { swim(): void { /* ... */ } }

// コンポジション: 機能を自由に組み合わせられる
interface Flyable { fly(): void; }
interface Swimmable { swim(): void; }

class Duck implements Flyable, Swimmable {
  constructor(
    private readonly flyBehavior: Flyable,
    private readonly swimBehavior: Swimmable
  ) {}

  fly(): void { this.flyBehavior.fly(); }
  swim(): void { this.swimBehavior.swim(); }
}
```

> **実務での指針**: 「is-a（〜は〜である）」の関係が本当に成立する場合のみ継承を使い、「has-a（〜は〜を持つ）」の関係にはコンポジションを使いましょう。迷ったらコンポジションを選ぶのが安全です。

### 2. SOLID原則（50分）

#### S — 単一責任の原則 (Single Responsibility Principle)

- クラスが変更される理由は1つだけであるべき
- 「責務」の粒度をどう判断するか
- TypeScript例：レポート生成と出力フォーマットの分離

> **よくある質問: 「責務」の粒度はどう決めるの？**
> Robert C. Martin は「変更の理由」で判断することを提案しています。例えば `UserService` に「ユーザー作成」「メール送信」「ログ出力」が含まれている場合、メールの仕様変更・ログフォーマットの変更・ユーザー登録ロジックの変更という3つの異なる理由で変更が発生します。変更の理由が複数あるクラスは分割の候補です。
>
> ただし、責務を細分化しすぎると逆にクラス数が爆発して理解しにくくなります。「今このコードに3つ以上の変更理由があるか？」を目安にするのが実用的です。

#### O — 開放閉鎖の原則 (Open/Closed Principle)

- 拡張に対して開いている、修正に対して閉じている
- Strategy パターンによる拡張ポイントの設計
- TypeScript例：割引計算ロジックの拡張

```typescript
// Bad: 新しい割引タイプを追加するたびに既存コードを修正する必要がある
class PriceCalculator {
  calculate(basePrice: number, discountType: string): number {
    if (discountType === 'member') return basePrice * 0.9;
    if (discountType === 'sale') return basePrice * 0.8;
    if (discountType === 'coupon') return basePrice * 0.85;
    return basePrice;
  }
}

// Good: 新しい割引を追加しても既存コードを変更しない
interface DiscountStrategy {
  apply(basePrice: number): number;
}

class MemberDiscount implements DiscountStrategy {
  apply(basePrice: number): number { return basePrice * 0.9; }
}

class SaleDiscount implements DiscountStrategy {
  apply(basePrice: number): number { return basePrice * 0.8; }
}

// 新しい割引を追加 → 新クラスを作るだけ。既存コードの修正は不要
class CouponDiscount implements DiscountStrategy {
  constructor(private readonly rate: number) {}
  apply(basePrice: number): number { return basePrice * (1 - this.rate); }
}

class PriceCalculator {
  calculate(basePrice: number, discount: DiscountStrategy): number {
    return discount.apply(basePrice);
  }
}
```

#### L — リスコフの置換原則 (Liskov Substitution Principle)

- サブタイプは基底型の契約を守らなければならない
- 振る舞いの互換性と事前条件・事後条件
- TypeScript例：不適切な継承と正しい設計の比較

```typescript
// Bad: 有名な「正方形・長方形問題」
class Rectangle {
  constructor(protected width: number, protected height: number) {}

  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
  getArea(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  // 正方形は幅と高さが同じ → setWidth で height も変えてしまう
  setWidth(w: number): void { this.width = w; this.height = w; }
  setHeight(h: number): void { this.width = h; this.height = h; }
}

// Rectangle を期待するコードで Square を使うと壊れる
function doubleWidth(rect: Rectangle): void {
  const originalHeight = rect.getArea() / rect.getArea(); // ←意図と違う結果に
  rect.setWidth(rect.getArea() / originalHeight * 2);
}

// Good: 共通インターフェースで本当に互換性のある抽象を定義
interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  getArea(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private side: number) {}
  getArea(): number { return this.side * this.side; }
}
```

> **実務での教訓**: LSPに違反しているサインは「子クラスで親のメソッドが意味をなさない」「子クラスに型チェック（`instanceof`）が必要になる」場合です。こうしたケースでは、継承ではなくインターフェースで共通部分を抽出しましょう。

#### I — インターフェース分離の原則 (Interface Segregation Principle)

- クライアントが使わないメソッドへの依存を強制しない
- 大きなインターフェースを小さく分割する
- TypeScript例：`interface Readable` と `interface Writable` の分離

```typescript
// Bad: 1つの大きなインターフェース — 読み取り専用の用途でも write を実装させられる
interface FileHandler {
  read(path: string): string;
  write(path: string, content: string): void;
  delete(path: string): void;
  getPermissions(path: string): string[];
}

// Good: 用途ごとに分割 — 必要なインターフェースだけ実装すればよい
interface Readable {
  read(path: string): string;
}

interface Writable {
  write(path: string, content: string): void;
}

interface Deletable {
  delete(path: string): void;
}

// 読み取り専用のクライアントは Readable だけに依存
class ReportGenerator {
  constructor(private readonly reader: Readable) {}

  generate(path: string): string {
    const data = this.reader.read(path);
    return `Report: ${data}`;
  }
}
```

#### D — 依存性逆転の原則 (Dependency Inversion Principle)

- 上位モジュールは下位モジュールに依存すべきでない
- 抽象（インターフェース）に依存する
- TypeScript例：リポジトリインターフェースによるDB依存の逆転

> **なぜ「逆転」と呼ぶのか？**
> 通常、ビジネスロジック（上位）がDBアクセス（下位）を直接呼び出します。この場合、DBの変更がビジネスロジックに波及します。DIPでは**上位側がインターフェースを定義し、下位側がそれを実装する**ことで、依存の方向が逆転します。
>
> ```
> 【通常の依存】
> OrderService → PrismaClient（上位が下位に依存）
>
> 【依存性逆転後】
> OrderService → OrderRepository（interface）← PrismaOrderRepository
> （上位は抽象に依存、下位が抽象を実装）
> ```
>
> これにより、テスト時にはインメモリ実装に差し替えられ、将来DBを変更しても上位のコードは影響を受けません。この原則は第6回（クリーンアーキテクチャ）の核心概念になるため、しっかり理解しておきましょう。

### 3. 実務での適用（15分）

- 過度な抽象化の罠 — YAGNI（You Ain't Gonna Need It）との折り合い
- テスタビリティと設計品質の関係
- 既存コードのリファクタリングアプローチ

#### SOLID原則と「ちょうどよい設計」のバランス

SOLID原則は絶対的なルールではなく、設計のガイドラインです。以下の判断基準を持っておくと、実務で「やりすぎ」を避けられます。

| 状況 | 推奨アプローチ |
|------|--------------|
| 変更が見込まれる箇所 | SOLID原則をしっかり適用する |
| 安定していて変更されない箇所 | シンプルさを優先し、過度に抽象化しない |
| プロトタイプ・実験的なコード | まず動くものを作り、安定したら設計を改善する |
| チームの理解度が低い場合 | 段階的に導入し、まずSRP（単一責任）から始める |

> **実務Tip**: リファクタリングの順番として「①まずSRP（責務の分離）→ ②次にDIP（依存の逆転）→ ③必要に応じてOCP（拡張ポイントの設計）」の順で取り組むと、段階的に設計品質を改善できます。一度にすべてを適用しようとしないことが大切です。

#### テストしやすいコード = 設計品質の高いコード

テストの書きやすさは設計品質のバロメーターです。テストが書きにくいコードには、多くの場合以下の設計上の問題があります。

```
テストしにくい                          設計上の問題
─────────────────────────────────────────────────────
外部API/DBに直接依存している    →  依存性逆転の原則（DIP）の違反
1つのメソッドが長すぎる          →  単一責任の原則（SRP）の違反
グローバル状態を参照している     →  カプセル化の不足
具象クラスに依存している        →  インターフェースの欠如
```

## コード例の概要

講義中に使用する主なTypeScriptコード例。

```typescript
// Bad: 複数の責務が混在
class OrderService {
  calculateTotal(order: Order): number { /* ... */ }
  formatReceipt(order: Order): string { /* ... */ }
  sendEmail(to: string, body: string): void { /* ... */ }
}

// Good: 責務を分離
class OrderCalculator {
  calculateTotal(order: Order): number { /* ... */ }
}

class ReceiptFormatter {
  format(order: Order): string { /* ... */ }
}

class EmailSender {
  send(to: string, body: string): void { /* ... */ }
}
```

```typescript
// 依存性逆転の原則
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// ビジネスロジックは抽象に依存
class OrderService {
  constructor(private readonly orderRepo: OrderRepository) {}

  async completeOrder(orderId: string): Promise<void> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new Error('Order not found');
    order.complete();
    await this.orderRepo.save(order);
  }
}
```

## ディスカッションテーマ

以下のテーマから1つ選んでグループで議論する。

1. **SOLID原則のトレードオフ** — SOLID原則を厳格に適用しすぎると、どんな問題が起きるか？
2. **実務のコード改善** — 自分のプロジェクトで「単一責任の原則に違反している」と思うコードはあるか？
3. **テスタビリティ** — テストしづらいコードの特徴は何か？オブジェクト指向の原則とどう関係するか？

## 課題

### 復習問題

1. SOLID原則の各原則を、自分の言葉で1文ずつ説明してください
2. 依存性逆転の原則が守られていないコード例を1つ挙げ、改善案を示してください
3. 「継承よりもコンポジション」が推奨される理由を説明してください

### コード課題

以下の要件を持つ「通知システム」をTypeScriptで設計してください。

- メール、Slack、SMS の3つの通知手段に対応する
- 通知の送信先と本文を受け取って送信する
- 将来的に新しい通知手段（LINE等）を追加可能にする
- SOLID原則を意識した設計にする

### 発展課題（任意）

自分のプロジェクトのコードから、SOLID原則に違反していると思われる箇所を1つ見つけ、リファクタリング案を書いてください。Before/Afterのコードと、どの原則を適用したかを説明してください。

## 事前準備

### 参加者

#### 前提知識の確認

- [ ] TypeScriptのクラス構文を理解している（`class`、`constructor`、`public`/`private`/`readonly`）
- [ ] TypeScriptのインターフェース構文を理解している（`interface`、`implements`）
- [ ] 「継承」と「インターフェースの実装」の構文上の違いが分かる

#### 事前読み物（30分程度）

- [ ] 自分が最近書いた（または読んだ）コードの中で「このクラス、いろいろやりすぎでは？」と感じた箇所を1つ思い出しておく
- [ ] 以下のキーワードについて、知っている範囲で意味を調べておく
  - カプセル化、ポリモーフィズム、継承
  - SOLID原則（概要レベルでOK）

#### 事前ミニ課題（任意・15分程度）

以下のコードを読み、「何が問題だと思うか」を自分の言葉でメモしておいてください。正解は講義中に扱います。

```typescript
class UserService {
  async createUser(name: string, email: string): Promise<void> {
    // バリデーション
    if (!email.includes('@')) throw new Error('Invalid email');
    // DBに保存
    await db.query('INSERT INTO users (name, email) VALUES (?, ?)', [name, email]);
    // ウェルカムメール送信
    await sendEmail(email, 'ようこそ！', `${name}さん、登録ありがとうございます。`);
    // 管理者に通知
    await sendSlack('#admin', `新規ユーザー登録: ${name}`);
    // ログ出力
    console.log(`User created: ${name} (${email})`);
  }
}
```

### 運営側

- [ ] 第0回の振り返り・参加者の反応の確認
- [ ] SOLID原則の各コード例の動作確認（TypeScript Playground等）
- [ ] 講義スライドの準備（Bad/Goodコードの比較が視覚的に分かる構成）
- [ ] ディスカッションのファシリテーション準備（グループ分け案の検討）
- [ ] 参加者の事前アンケート結果に基づく理解度の把握

## 参考資料

- 『オブジェクト指向設計実践ガイド』Sandi Metz
- 『Clean Code』Robert C. Martin
- [SOLID Principles in TypeScript](https://blog.logrocket.com/solid-principles-typescript/) （Web記事）

## 次回との接続

第1回で学んだ**依存性逆転の原則**と**インターフェースによる抽象化**は、第2回以降のすべてのテーマで繰り返し登場します。特に第2回（RDB）では、データアクセスの抽象化としてリポジトリパターンの具体例を見ていきます。
