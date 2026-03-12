# 第4回：デザインパターン

> 設計の「型」を知り、適切に適用する判断力を養う

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第1回（オブジェクト指向）の内容、特にSOLID原則 |
| 到達目標 | 実務で頻出するデザインパターンを理解し、問題に応じて適用判断ができる |

## この回のキーメッセージ

> 「パターンは道具。問題に合わせて選択し、過剰適用を避ける」

## 講義内容

### 1. デザインパターンとは（10分）

- GoF（Gang of Four）の23パターンとその背景
- パターンは「発明」ではなく「発見」— 繰り返し現れる解決策の命名
- パターンを学ぶ意義 — 共通語彙、設計判断の引き出し
- アンチパターン：パターンの過剰適用（パターン病）

### 2. 生成に関するパターン（20分）

#### Factory Method / Abstract Factory

- オブジェクト生成のロジックを分離する
- 「どのクラスを生成するか」の判断をカプセル化
- TypeScript例：通知チャネル（Email / Slack / SMS）のファクトリ

```typescript
// Factory パターン: 生成ロジックをカプセル化
interface Notification {
  send(to: string, message: string): Promise<void>;
}

class EmailNotification implements Notification {
  async send(to: string, message: string): Promise<void> {
    // SMTP経由でメール送信
  }
}

class SlackNotification implements Notification {
  async send(to: string, message: string): Promise<void> {
    // Slack API経由で送信
  }
}

// Factoryが「どのクラスを使うか」の判断を引き受ける
class NotificationFactory {
  static create(channel: 'email' | 'slack' | 'sms'): Notification {
    switch (channel) {
      case 'email': return new EmailNotification();
      case 'slack': return new SlackNotification();
      case 'sms':   return new SmsNotification();
    }
  }
}

// 利用側は具象クラスを知らなくてよい
const notification = NotificationFactory.create('email');
await notification.send('user@example.com', 'Hello!');
```

> **Factory vs コンストラクタ直接呼び出し**: `new EmailNotification()` を直接書くのと何が違うのか？Factoryを使うと、利用側は具象クラスに依存しなくなります。通知手段の追加・変更があってもFactoryクラスだけ修正すればよく、利用側のコードは変わりません。

#### Builder

- 複雑なオブジェクトを段階的に構築する
- メソッドチェインによる可読性の向上
- TypeScript例：クエリビルダー、設定オブジェクトの構築

```typescript
// Builder パターン: 複雑なオブジェクトを段階的に構築
class QueryBuilder {
  private _table: string = '';
  private _conditions: string[] = [];
  private _orderBy: string | null = null;
  private _limit: number | null = null;

  from(table: string): this {
    this._table = table;
    return this; // this を返すことでメソッドチェインが可能
  }

  where(condition: string): this {
    this._conditions.push(condition);
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this._orderBy = `${column} ${direction}`;
    return this;
  }

  limit(count: number): this {
    this._limit = count;
    return this;
  }

  build(): string {
    let query = `SELECT * FROM ${this._table}`;
    if (this._conditions.length > 0) {
      query += ` WHERE ${this._conditions.join(' AND ')}`;
    }
    if (this._orderBy) query += ` ORDER BY ${this._orderBy}`;
    if (this._limit) query += ` LIMIT ${this._limit}`;
    return query;
  }
}

// メソッドチェインで直感的に組み立てられる
const query = new QueryBuilder()
  .from('orders')
  .where('status = "COMPLETED"')
  .where('created_at > "2025-01-01"')
  .orderBy('created_at', 'DESC')
  .limit(10)
  .build();
```

> **Builderを使うべきサイン**: コンストラクタの引数が4つ以上ある、オプション引数が多い、構築の順序に柔軟性が欲しい — これらの場合にBuilderパターンが効果的です。

### 3. 構造に関するパターン（20分）

#### Adapter

- 既存のインターフェースを期待されるインターフェースに変換する
- 外部ライブラリやレガシーコードとの接続に活用
- TypeScript例：外部決済APIのアダプター

```typescript
// Adapter パターン: 外部APIのインターフェースを自分のシステムに合わせて変換

// 自分のシステムが期待するインターフェース
interface PaymentGateway {
  charge(amount: number, currency: string, token: string): Promise<{ transactionId: string }>;
}

// 外部のStripe API（自分のシステムとはインターフェースが違う）
class StripeApi {
  async createCharge(params: {
    amount: number;
    currency: string;
    source: string;
    description?: string;
  }): Promise<{ id: string; status: string }> {
    // Stripe APIの実際の呼び出し
    return { id: 'ch_xxx', status: 'succeeded' };
  }
}

// Adapter: Stripe APIを自分のインターフェースに変換
class StripePaymentAdapter implements PaymentGateway {
  constructor(private readonly stripe: StripeApi) {}

  async charge(amount: number, currency: string, token: string) {
    const result = await this.stripe.createCharge({
      amount,
      currency,
      source: token,
    });
    return { transactionId: result.id };
  }
}

// 将来PayPalに切り替える場合も、PayPalPaymentAdapter を作るだけ
```

> **Adapterが必要なサイン**: 外部API/ライブラリを使うとき、「自分のコードにそのAPIの型やメソッド名が広がっている」なら、Adapterで隔離すべきです。第6回（クリーンアーキテクチャ）では、この考え方をシステム全体に適用します。

#### Facade

- 複雑なサブシステムに対するシンプルなインターフェースを提供
- TypeScript例：注文処理のファサード（在庫確認 + 決済 + 配送手配）

```typescript
// Facade パターン: 複数のサブシステムをまとめたシンプルなインターフェース

class OrderFacade {
  constructor(
    private readonly inventory: InventoryService,
    private readonly payment: PaymentGateway,
    private readonly shipping: ShippingService,
    private readonly notification: NotificationService
  ) {}

  // 利用側はこの1メソッドを呼ぶだけでよい
  async placeOrder(order: OrderRequest): Promise<OrderResult> {
    // 1. 在庫確認
    await this.inventory.checkAndReserve(order.items);
    // 2. 決済
    const tx = await this.payment.charge(order.totalAmount, 'JPY', order.paymentToken);
    // 3. 配送手配
    const tracking = await this.shipping.arrange(order.shippingAddress, order.items);
    // 4. 通知
    await this.notification.send(order.customerId, `注文完了: ${tracking.trackingNumber}`);

    return { transactionId: tx.transactionId, trackingNumber: tracking.trackingNumber };
  }
}
```

> **Facade vs Service**: 「それって普通のサービスクラスでは？」と思うかもしれません。Facadeの本質は「複雑さを隠す」ことです。サブシステムの詳細を知らなくても、Facadeを通じて操作できるようにするのがポイントです。

### 4. 振る舞いに関するパターン（30分）

#### Strategy

- アルゴリズムをカプセル化し、実行時に切り替え可能にする
- 第1回で学んだ開放閉鎖の原則の実践
- TypeScript例：価格計算戦略（通常 / 会員 / セール）

#### Observer

- オブジェクト間の1対多の依存関係を定義
- 状態変化を自動通知する仕組み
- TypeScript例：注文状態の変化に応じた通知
- EventEmitter / RxJS との関連

```typescript
// Observer パターン: 状態変化を購読者に自動通知

interface OrderEventListener {
  onOrderStatusChanged(orderId: string, newStatus: string): void;
}

class OrderStatusNotifier {
  private listeners: OrderEventListener[] = [];

  addListener(listener: OrderEventListener): void {
    this.listeners.push(listener);
  }

  notify(orderId: string, newStatus: string): void {
    for (const listener of this.listeners) {
      listener.onOrderStatusChanged(orderId, newStatus);
    }
  }
}

// 購読者: メール通知
class EmailNotifier implements OrderEventListener {
  onOrderStatusChanged(orderId: string, newStatus: string): void {
    // 顧客にステータス変更メールを送信
  }
}

// 購読者: 在庫管理
class InventoryUpdater implements OrderEventListener {
  onOrderStatusChanged(orderId: string, newStatus: string): void {
    if (newStatus === 'CANCELLED') {
      // 在庫を戻す
    }
  }
}

// 新しい購読者を追加しても、通知元（OrderStatusNotifier）のコードは変わらない
```

> **Node.js での Observer**: Node.js の `EventEmitter` はまさにObserverパターンの実装です。第9回（イベント駆動）では、これをサービス間に拡張した形を学びます。

#### Repository

- データアクセスのロジックを抽象化する
- ドメインオブジェクトの永続化と取得のインターフェース
- 第1回の依存性逆転、第2回のRDB知識を統合
- TypeScript例：インメモリリポジトリとDB リポジトリの切り替え

### 5. パターンの適用判断（15分）

- 「問題があってパターンがある」— パターンありきで設計しない
- パターンを適用すべき兆候（コードの匂い）
- TypeScript / Node.js エコシステムにおけるパターンの現れ方
  - ミドルウェアパターン（Express / Koa）
  - プラグインパターン（各種ライブラリ）
  - DIコンテナ（NestJS）

#### パターンを適用すべき「コードの匂い」チェックリスト

| コードの匂い | 適用候補のパターン |
|------------|----------------|
| `if/else` や `switch` が長大で、新しいケースの追加が頻繁 | Strategy, Factory |
| オブジェクト生成のロジックが複雑で、あちこちにコピーされている | Factory, Builder |
| 外部API/ライブラリの呼び出しがビジネスロジックに直接埋まっている | Adapter, Repository |
| ある状態変化に対して、複数の処理を連鎖的に実行したい | Observer |
| 複数のサブシステムを組み合わせた処理が散在している | Facade |

#### Node.js / TypeScript エコシステムでのパターンの例

```
Express ミドルウェア    → Chain of Responsibility パターン
  app.use(cors());     → リクエストが複数のハンドラを順に通過
  app.use(auth());
  app.use(logging());

NestJS の DI           → Dependency Injection + Factory パターン
  @Injectable()        → フレームワークがオブジェクトの生成と注入を管理

Prisma Client          → Repository + Builder パターン
  prisma.user.findMany({ where: {...} })
                       → メソッドチェインでクエリを構築
```

## コード例の概要

```typescript
// Strategy パターン
interface PricingStrategy {
  calculate(basePrice: number): number;
}

class RegularPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice;
  }
}

class MemberPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice * 0.9; // 10% off
  }
}

class SalePricing implements PricingStrategy {
  constructor(private readonly discountRate: number) {}
  calculate(basePrice: number): number {
    return basePrice * (1 - this.discountRate);
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  calculatePrice(basePrice: number): number {
    return this.strategy.calculate(basePrice);
  }
}
```

```typescript
// Repository パターン
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// テスト用：インメモリ実装
class InMemoryUserRepository implements UserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }
  // ...
}

// 本番用：Prisma実装
class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } });
    return data ? UserMapper.toDomain(data) : null;
  }
  // ...
}
```

## ディスカッションテーマ

1. **パターンの過剰適用** — 「パターンを使いすぎた」経験や、見かけたコードはあるか？
2. **フレームワークとパターン** — Express、NestJS、Next.js 等で、どのデザインパターンが使われているか？
3. **パターンの命名効果** — チーム内で「このクラスはStrategyパターンだ」と言えることの価値は何か？

## 課題

### 復習問題

1. Factory Method パターンと Strategy パターンの違いを説明してください
2. Adapter パターンが必要になる具体的な場面を2つ挙げてください
3. Repository パターンを使うことで得られるテスト上の利点を説明してください

### コード課題

ECサイトの「配送料金計算」をデザインパターンを用いて設計してください。

要件：
- 配送方法は「通常配送」「速達」「コンビニ受取」の3種類
- 地域（都道府県）によって料金が異なる
- 一定金額以上の注文で送料無料になる
- 将来的に新しい配送方法の追加が見込まれる

成果物：
- どのパターンを適用したかと、その理由
- TypeScriptのコード（インターフェースと主要クラスの実装）

### 発展課題（任意）

NestJSのソースコードを読み、どのデザインパターンが使われているかを3つ以上特定し、それぞれの役割を説明してください。

## 事前準備

### 参加者

#### 前提知識の確認

- [ ] 第1回のSOLID原則（特に開放閉鎖の原則・依存性逆転の原則）を復習済み
- [ ] TypeScriptのインターフェースと実装の使い分けが理解できている
- [ ] 「抽象に依存する」という概念がイメージできる

#### 事前読み物（30分程度）

- [ ] 以下のキーワードについて、知っている範囲で概要を調べておく
  - Factory パターン、Strategy パターン、Observer パターン
  - 「デザインパターン」という考え方が生まれた背景（GoF）
- [ ] 自分が使っているフレームワーク（Express、NestJS、Next.js等）で「これはパターンっぽい」と感じる仕組みを1つ挙げてみる
  - 例：Expressのミドルウェア、NestJSのDIコンテナ

#### 事前ミニ課題（任意・15分程度）

以下のコードを読み、「新しい割引タイプを追加するとき何が大変か」を考えてメモしておいてください。

```typescript
class PriceCalculator {
  calculate(basePrice: number, discountType: string): number {
    if (discountType === 'member') {
      return basePrice * 0.9;
    } else if (discountType === 'sale') {
      return basePrice * 0.8;
    } else if (discountType === 'coupon') {
      return basePrice * 0.85;
    } else {
      return basePrice;
    }
  }
}
```

### 運営側

- [ ] 第3回の課題提出状況の確認
- [ ] 各パターンのコード例が動作確認済みであること
- [ ] パターンの適用前（Before）/ 適用後（After）を並べて比較できるスライドの準備
- [ ] Repository パターンのインメモリ実装とDB実装の切り替えデモの準備
- [ ] NestJS / Express での実際のパターン活用例の資料準備

## 参考資料

- 『Head First デザインパターン』Eric Freeman 他
- 『リファクタリング 既存のコードを安全に改善する（第2版）』Martin Fowler
- [Refactoring Guru — Design Patterns](https://refactoring.guru/design-patterns)

## 次回との接続

第4回で学んだRepository、Factory、Strategyパターンは、第5回（DDD）の戦術的設計パターンの基礎になります。DDDでは、これらのパターンをドメインモデルの構築に統合的に活用します。
