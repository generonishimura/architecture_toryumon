# 第9回：イベント駆動アーキテクチャ

> 非同期は強力だが複雑。結果整合性のトレードオフを理解する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | 第5回（DDD、ドメインイベント）、第7回（マイクロサービス） |
| 到達目標 | イベント駆動アーキテクチャの利点とトレードオフを理解し、CQRS・Sagaパターンを説明できる |

## この回のキーメッセージ

> 「非同期は強力だが複雑。結果整合性のトレードオフを理解する」

## 講義内容

### 1. イベント駆動アーキテクチャとは（15分）

- 「コマンド」と「イベント」の違い
  - コマンド：「〇〇をしてください」（命令）
  - イベント：「〇〇が起こりました」（事実の通知）
- イベント駆動で解決できる問題
  - サービス間の疎結合
  - 非同期処理によるレスポンス改善
  - イベントによるシステム拡張性
- 第5回で学んだドメインイベントとの関係

### 2. メッセージングの基盤（20分）

#### メッセージブローカー

- メッセージキュー（RabbitMQ、Amazon SQS）
  - Point-to-Pointモデル
  - 1つのメッセージは1つのコンシューマーが処理
- イベントストリーム（Apache Kafka）
  - Publish/Subscribeモデル
  - 1つのイベントを複数のコンシューマーが処理
  - イベントの永続化と再生（リプレイ）

#### メッセージングのパターン

- Pub/Sub（パブリッシュ・サブスクライブ）
- メッセージの順序保証
- At-least-once / At-most-once / Exactly-onceデリバリー
- 冪等性（Idempotency）の重要性
- Transactional Outboxパターン：集約の保存とイベントの記録を、同じDBトランザクション内でoutboxテーブルに書き込む。別プロセス（リレー）がoutboxテーブルを読んでブローカーに送る。これで「DBには保存されたのにイベントが送られない」状態を防ぐ

### 3. CQRS（Command Query Responsibility Segregation）（25分）

#### CQRSの概念

- コマンド（書き込み）とクエリ（読み取り）の責務を分離
- なぜ分離するのか：読み書きの要件は大きく異なる
- 書き込みモデルと読み取りモデルを別々に最適化

#### CQRSの実装レベル

```
レベル1: コード内での分離
  → 同じDBだが、Write操作とRead操作のコードパスを分離

レベル2: モデルの分離
  → 同じDBだが、Write用モデルとRead用モデルを別に持つ

レベル3: データストアの分離
  → Write用DB（正規化RDB）とRead用DB（非正規化、全文検索等）を分離
  → イベントで同期
```

#### CQRSの適用判断

- 読み書きの負荷差が大きい場合
- 読み取りに複雑な結合が必要な場合
- CQRSはすべてのシステムに必要なわけではない。複雑さが増す点とのトレードオフで判断する

### 4. 結果整合性とSagaパターン（25分）

#### 結果整合性（Eventual Consistency）

- 強い整合性 vs 結果整合性
- 分散システムにおけるCAP定理（ネットワーク分断時にCとAのどちらを優先するか）
- 結果整合性を受け入れるための設計
- ユーザーへの見せ方の工夫

#### Sagaパターン

- 分散トランザクションの代替手法
- 複数サービスにまたがるビジネスプロセスの管理

#### コレオグラフィ vs オーケストレーション

> 注: ここでの「オーケストレーション」は、第8回のコンテナオーケストレーション（Kubernetesなど）とは別の概念である。Sagaのオーケストレーションは、業務処理の実行順序を1つのサービス（オーケストレーター）が指示する方式を指す。

```
コレオグラフィ（Choreography）:
  各サービスが前のサービスのイベントに反応して、次の処理を自分で始める

  [注文サービス] ──OrderPlaced──▶ [在庫サービス] 在庫引当
  [在庫サービス] ──StockReserved──▶ [決済サービス] 決済処理
  [決済サービス] ──PaymentCompleted──▶ [注文サービス] 注文の決済完了を記録する
                                    └─▶ [通知サービス] 確認メール送信

  在庫引当の失敗時: [在庫サービス] ──StockReservationFailed──▶ [注文サービス] 注文をキャンセル済みにする（補償）
  決済の失敗時:     [決済サービス] ──PaymentFailed──▶ [在庫サービス] 引当解除（補償）
                                                  └─▶ [注文サービス] 注文をキャンセル済みにする（補償）


オーケストレーション（Orchestration）:
  中央のオーケストレータがフローを制御する

  注文確定 → [注文サービス（オーケストレータ）]
                    │
                    ├──▶ [在庫サービス] 在庫引当
                    │         │
                    │    成功  │
                    ├──▶ [決済サービス] 決済処理
                    │         │
                    │    成功  │
                    └──▶ [通知サービス] 確認メール
```

- メール送信のように取り消せない処理は、補償が必要になる可能性がある処理がすべて成功したあとに置く

#### 補償トランザクション

- Sagaの途中のステップが失敗したとき、それまでに成功したステップの結果を打ち消すために実行する処理
- DBのロールバックとは違う。各サービスのローカルトランザクションはすでにコミット済みで、他の処理がその結果を読んでいる可能性がある。そのため、変更を無かったことにはできない。代わりに、逆向きの業務操作を新しく記録する（例: 決済に対して「返金」、在庫引当に対して「引当解除」）
- メール送信や外部APIへの通知のように打ち消せない処理もある。こうした処理はSagaの最後に置く

### 5. Node.jsでのイベント駆動実装（15分）

- EventEmitterを使ったプロセス内イベント
- Redis Streams（コンシューマーグループとACKで配送を管理できる）を使ったサービス間イベント
  - ※Redis Pub/Subはメッセージを保存しないため、購読側が止まっている間のイベントは失われる。画面のリアルタイム更新のように、取りこぼしてもよい通知に限って使う
- BullMQを使ったジョブキュー

## コード例の概要

```typescript
// ドメインイベントの発行と購読（プロセス内）
import { EventEmitter } from 'events';
import { randomUUID } from 'crypto';

// ドメインイベントの型定義
// 第5回の OrderPlaced（orderId, customerId, totalAmount, occurredAt）の拡張版。
// 第9回では eventId（冪等性の確保に使う）と items（在庫引当に使う）を追加する。
// そのため Order.confirm() は、明細から items を作って OrderPlaced に渡す。
class OrderPlaced {
  readonly eventId: string = randomUUID();
  readonly occurredAt: Date = new Date();

  constructor(
    readonly orderId: string,
    readonly customerId: string,
    readonly totalAmount: Money,
    readonly items: ReadonlyArray<{ productId: string; quantity: number }>
  ) {}
}

// 決済の完了を表すイベント（Saga の最後のイベント）
class PaymentCompleted {
  readonly eventId: string = randomUUID();
  readonly occurredAt: Date = new Date();

  constructor(
    readonly orderId: string,
    readonly customerId: string
  ) {}
}

// イベント名と型の対応表
type DomainEvents = {
  OrderPlaced: OrderPlaced;
  PaymentCompleted: PaymentCompleted;
};

// イベントバス
class DomainEventBus {
  private emitter = new EventEmitter();

  publish<K extends keyof DomainEvents>(name: K, event: DomainEvents[K]): void {
    this.emitter.emit(name, event);
  }

  subscribe<K extends keyof DomainEvents>(
    name: K,
    handler: (event: DomainEvents[K]) => Promise<void>
  ): void {
    this.emitter.on(name, (event: DomainEvents[K]) => {
      // EventEmitter は async リスナーの完了を待たないため、失敗をここで受け止める
      handler(event).catch((err) => console.error(`[${String(name)}] handler failed`, err));
    });
  }
}

// 購読側の例
const eventBus = new DomainEventBus();

// 在庫引当ハンドラ
eventBus.subscribe('OrderPlaced', async (event) => {
  for (const item of event.items) {
    await inventoryService.reserve(item.productId, item.quantity);
  }
});

// 通知ハンドラ
// 確認メールは取り消せないので、OrderPlaced ではなく PaymentCompleted を受けてから送る
eventBus.subscribe('PaymentCompleted', async (event) => {
  await notificationService.sendOrderConfirmation(
    event.customerId,
    event.orderId
  );
});
```

- EventEmitterはプロセス内だけで動く。プロセスが停止すると処理中のイベントは失われ、再送もされない

```typescript
// CQRS: 書き込みモデルと読み取りモデルの分離

// === Write Side ===
// 流れは第6回の PlaceOrderUseCase と同じ
class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly productRepo: ProductRepository,
    private readonly eventPublisher: DomainEventPublisher
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    const order = new Order(generateId(), input.customerId,
      Address.from(input.shippingAddress));

    // ビジネスロジックの実行
    for (const item of input.items) {
      const product = await this.productRepo.findById(item.productId);
      if (!product) throw new Error(`商品が見つかりません: ${item.productId}`);
      order.addItem(product.id, item.quantity, product.price);
    }

    // 注文確定。confirm() が返すドメインイベントを受け取る
    const event = order.confirm();

    // 正規化されたDBに保存
    await this.orderRepo.save(order);

    // イベントを発行（Read側の更新をトリガー）
    // 簡略化のため直接 publish している。本番では保存と同じトランザクションで outbox に書き込む
    await this.eventPublisher.publish(event);

    return {
      orderId: order.id,
      totalAmount: order.totalAmount.amount,
    };
  }
}

// === Read Side ===
// 読み取り専用のビュー（非正規化）
interface OrderSummaryView {
  orderId: string;
  customerName: string;       // Read側が持つ顧客情報から取得して結合済み
  itemCount: number;
  totalAmount: number;
  status: string;
  placedAt: Date;
}

// イベントハンドラでRead用ビューを更新
class OrderSummaryProjection {
  constructor(
    private readonly viewStore: OrderSummaryViewStore,
    private readonly customerLookup: CustomerLookup  // Read側が持つ顧客情報の参照
  ) {}

  async onOrderPlaced(event: OrderPlaced): Promise<void> {
    // 顧客名はイベントに含まれないので、Read側の顧客情報から取得する
    const customerName = await this.customerLookup.findNameById(event.customerId);

    // Read用のビューを非正規化して保存
    await this.viewStore.upsert({
      orderId: event.orderId,
      customerName,
      itemCount: event.items.length,
      totalAmount: event.totalAmount.amount,
      status: 'CONFIRMED',
      placedAt: event.occurredAt,
    });
  }
}
```

```typescript
// 冪等性の確保
// 前提: ドメインイベントは eventId（UUID）を持つ（上の OrderPlaced を参照）
class IdempotentEventHandler {
  constructor(
    private readonly db: Database,
    private readonly processedEvents: ProcessedEventStore,
    private readonly handler: EventHandler
  ) {}

  async handle(event: DomainEvent): Promise<void> {
    await this.db.transaction(async (tx) => {
      // processed_events.event_id に一意制約を付ける。
      // 同時に2件届いても INSERT に成功するのは1件だけ
      // tryInsert は INSERT ... ON CONFLICT (event_id) DO NOTHING を実行し、
      // 挿入した行数が1なら true を返す（重複でもエラーにならず、false が返る）
      const inserted = await this.processedEvents.tryInsert(tx, event.eventId);
      if (!inserted) return; // 処理済み
      await this.handler.handle(event, tx); // 業務処理も同じトランザクションで行う
    });
  }
}
```

- メール送信のようにDBの外に副作用がある処理は、同じトランザクションに入れられない。その場合は、送信先のAPIに冪等キーを渡すなどの別の対策が必要

## ディスカッションテーマ

1. **結果整合性の許容範囲**：ユーザーにとって「すぐに反映されなくてもよい」操作と「即座に反映されるべき」操作の境界はどこか？
2. **コレオグラフィ vs オーケストレーション**：それぞれの利点と適用場面は？自分のプロジェクトではどちらが適切か？
3. **イベント駆動の複雑さ**：イベント駆動を導入して失敗するケースは？避けるべきアンチパターンは？

## 課題

### 復習問題

1. コマンドとイベントの違いを説明し、それぞれの命名規則の例を3つずつ挙げてください
2. CQRSの3つの実装レベルと、それぞれが適切な場面を説明してください
3. Sagaパターンのコレオグラフィとオーケストレーションの違いを、メリット・デメリットと共に説明してください

### コード課題

ECサイトの「注文 → 在庫引当 → 決済 → 配送手配」のフローをイベント駆動で設計してください。

要件は次のとおり。

- 各ステップで失敗した場合の補償トランザクションを設計する
- 冪等性を考慮する
- TypeScriptでイベントの型定義とハンドラーの実装

成果物は次の4つ。

- イベントフロー図（テキストベース）
- イベントの型定義（TypeScript）
- 主要なイベントハンドラの実装
- 補償トランザクションの設計

### 発展課題（任意）

BullMQ（Redisベースのジョブキュー）を使って、注文確定後の非同期処理（メール送信、在庫更新）を実装してください。リトライと冪等性を含めてください。

## 参考資料

- 『データ指向アプリケーションデザイン』Martin Kleppmann
- 『マイクロサービスパターン』Chris Richardson（第4章（Saga）、第7章（クエリ／CQRS））
- [Event-Driven Architecture — Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)

## 次回との接続

第9回で、全9テーマの学習を終えた。最終回（第10回）では、9テーマで学んだ内容を振り返り、実際のシステムに対するアーキテクチャ設計レビューの演習を行う。
