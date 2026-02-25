# 第9回：イベント駆動アーキテクチャ

> 非同期は強力だが複雑。結果整合性のトレードオフを理解する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第5回（DDD、ドメインイベント）、第7回（マイクロサービス） |
| 到達目標 | イベント駆動アーキテクチャの利点とトレードオフを理解し、CQRS・Sagaパターンを説明できる |

## この回のキーメッセージ

> 「非同期は強力だが複雑。結果整合性のトレードオフを理解する」

## 講義内容

### 1. イベント駆動アーキテクチャとは（15分）

- 「コマンド」と「イベント」の違い
  - コマンド：「〇〇をしてください」（命令）
  - イベント：「〇〇が起こりました」（事実の通知）
- イベント駆動が解決する問題
  - サービス間の疎結合
  - 非同期処理によるレスポンス改善
  - イベントによるシステム拡張性
- 第5回で学んだドメインイベントとの関係

### 2. メッセージングの基盤（20分）

#### メッセージブローカー

- メッセージキュー（RabbitMQ、Amazon SQS）
  - Point-to-Point モデル
  - 1つのメッセージは1つのコンシューマーが処理
- イベントストリーム（Apache Kafka）
  - Publish/Subscribe モデル
  - 1つのイベントを複数のコンシューマーが処理
  - イベントの永続化と再生（リプレイ）

#### メッセージングのパターン

- Pub/Sub（パブリッシュ・サブスクライブ）
- メッセージの順序保証
- At-least-once / At-most-once / Exactly-once デリバリー
- 冪等性（Idempotency）の重要性

### 3. CQRS（Command Query Responsibility Segregation）（25分）

#### CQRSの概念

- コマンド（書き込み）とクエリ（読み取り）の責務を分離
- なぜ分離するのか — 読み書きの要件は根本的に異なる
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
- 全てのシステムにCQRSは不要 — 複雑さとのトレードオフ

### 4. 結果整合性とSagaパターン（25分）

#### 結果整合性（Eventual Consistency）

- 強い整合性 vs 結果整合性
- 分散システムにおけるCAP定理
- 結果整合性を受け入れるための設計
- ユーザーへの見せ方の工夫

#### Sagaパターン

- 分散トランザクションの代替手法
- 複数サービスにまたがるビジネスプロセスの管理

#### コレオグラフィ vs オーケストレーション

```
コレオグラフィ（Choreography）:
  各サービスがイベントに反応して自律的に動く

  注文確定 → [注文サービス] → OrderPlaced イベント
                                     │
                    ┌────────────────┼─────────────────┐
                    ▼                ▼                   ▼
              [在庫サービス]   [決済サービス]       [通知サービス]
              在庫引当        決済処理             確認メール送信


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

#### 補償トランザクション

- Sagaの各ステップが失敗した場合のロールバック処理
- 「元に戻す」のではなく「打ち消す処理を行う」

### 5. Node.js でのイベント駆動実装（15分）

- EventEmitter を使ったプロセス内イベント
- Redis Pub/Sub を使ったサービス間イベント
- BullMQ を使ったジョブキュー

## コード例の概要

```typescript
// ドメインイベントの発行と購読（プロセス内）
import { EventEmitter } from 'events';

// イベントの型定義
interface OrderPlacedEvent {
  orderId: string;
  customerId: string;
  items: { productId: string; quantity: number }[];
  totalAmount: number;
  occurredAt: Date;
}

// イベントバス
class DomainEventBus {
  private emitter = new EventEmitter();

  publish(eventName: string, event: unknown): void {
    this.emitter.emit(eventName, event);
  }

  subscribe(eventName: string, handler: (event: unknown) => void): void {
    this.emitter.on(eventName, handler);
  }
}

// 購読側の例
const eventBus = new DomainEventBus();

// 在庫引当ハンドラ
eventBus.subscribe('OrderPlaced', async (event: OrderPlacedEvent) => {
  for (const item of event.items) {
    await inventoryService.reserve(item.productId, item.quantity);
  }
});

// 通知ハンドラ
eventBus.subscribe('OrderPlaced', async (event: OrderPlacedEvent) => {
  await notificationService.sendOrderConfirmation(
    event.customerId,
    event.orderId
  );
});
```

```typescript
// CQRS: 書き込みモデルと読み取りモデルの分離

// === Write Side ===
class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly eventPublisher: EventPublisher
  ) {}

  async execute(input: PlaceOrderInput): Promise<string> {
    const order = new Order(/*...*/);
    // ビジネスロジックの実行
    order.addItems(input.items);
    order.confirm();

    // 正規化されたDBに保存
    await this.orderRepo.save(order);

    // イベントを発行（Read側の更新をトリガー）
    await this.eventPublisher.publish(
      new OrderPlacedEvent(order.id, order.customerId, order.totalAmount)
    );

    return order.id;
  }
}

// === Read Side ===
// 読み取り専用のビュー（非正規化）
interface OrderSummaryView {
  orderId: string;
  customerName: string;       // Userサービスから結合済み
  itemCount: number;
  totalAmount: number;
  status: string;
  placedAt: Date;
}

// イベントハンドラでRead用ビューを更新
class OrderSummaryProjection {
  constructor(private readonly viewStore: OrderSummaryViewStore) {}

  async onOrderPlaced(event: OrderPlacedEvent): Promise<void> {
    // Read用のビューを非正規化して保存
    await this.viewStore.upsert({
      orderId: event.orderId,
      customerName: event.customerName,
      itemCount: event.items.length,
      totalAmount: event.totalAmount,
      status: 'CONFIRMED',
      placedAt: event.occurredAt,
    });
  }
}
```

```typescript
// 冪等性の確保
class IdempotentEventHandler {
  constructor(
    private readonly processedEvents: ProcessedEventStore,
    private readonly handler: EventHandler
  ) {}

  async handle(event: DomainEvent): Promise<void> {
    // 既に処理済みなら何もしない
    if (await this.processedEvents.exists(event.eventId)) {
      return;
    }

    // イベント処理
    await this.handler.handle(event);

    // 処理済みとして記録
    await this.processedEvents.markAsProcessed(event.eventId);
  }
}
```

## ディスカッションテーマ

1. **結果整合性の許容範囲** — ユーザーにとって「すぐに反映されなくてもよい」操作と「即座に反映されるべき」操作の境界はどこか？
2. **コレオグラフィ vs オーケストレーション** — それぞれの利点と適用場面は？自分のプロジェクトではどちらが適切か？
3. **イベント駆動の複雑さ** — イベント駆動を導入して失敗するケースは？避けるべきアンチパターンは？

## 課題

### 復習問題

1. コマンドとイベントの違いを説明し、それぞれの命名規則の例を3つずつ挙げてください
2. CQRSの3つの実装レベルと、それぞれが適切な場面を説明してください
3. Sagaパターンのコレオグラフィとオーケストレーションの違いを、メリット・デメリットと共に説明してください

### コード課題

ECサイトの「注文 → 在庫引当 → 決済 → 配送手配」のフローをイベント駆動で設計してください。

要件：
- 各ステップで失敗した場合の補償トランザクションを設計する
- 冪等性を考慮する
- TypeScriptでイベントの型定義とハンドラーの実装

成果物：
- イベントフロー図（テキストベース）
- イベントの型定義（TypeScript）
- 主要なイベントハンドラの実装
- 補償トランザクションの設計

### 発展課題（任意）

BullMQ（Redis ベースのジョブキュー）を使って、注文確定後の非同期処理（メール送信、在庫更新）を実装してください。リトライと冪等性を含めてください。

## 参考資料

- 『データ指向アプリケーションデザイン』Martin Kleppmann
- 『マイクロサービスパターン』Chris Richardson（第4〜5章）
- [Event-Driven Architecture — Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)

## 次回との接続

第9回で、全9テーマの学習が完了しました。最終回（第10回）では、これまでの知識を総合的に振り返り、実際のシステムに対するアーキテクチャ設計レビューの演習を行います。
