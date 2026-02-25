# 第6回：クリーンアーキテクチャ

> ビジネスロジックを外部の詳細から守ることが、変更容易性の鍵

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第1回（OOP、特に依存性逆転）、第4回（デザインパターン）、第5回（DDD） |
| 到達目標 | クリーンアーキテクチャの原則を理解し、レイヤー構造とその依存ルールに基づいた設計ができる |

## この回のキーメッセージ

> 「ビジネスロジックを外部の詳細から守ることが、変更容易性の鍵」

## 講義内容

### 1. なぜアーキテクチャが必要か（10分）

- ソフトウェアの変更コストは時間とともに増大する
- アーキテクチャの目的 = **変更容易性**の確保
- Robert C. Martin『Clean Architecture』の核心メッセージ
- 関連するアーキテクチャパターン：Hexagonal / Onion / Clean

### 2. クリーンアーキテクチャの全体像（20分）

#### 同心円モデル

```
┌─────────────────────────────────────────────┐
│           Frameworks & Drivers               │
│  ┌───────────────────────────────────────┐  │
│  │       Interface Adapters               │  │
│  │  ┌───────────────────────────────┐    │  │
│  │  │     Application Business Rules │    │  │
│  │  │  ┌───────────────────────┐    │    │  │
│  │  │  │  Enterprise Business  │    │    │  │
│  │  │  │       Rules           │    │    │  │
│  │  │  │    (Entities)         │    │    │  │
│  │  │  └───────────────────────┘    │    │  │
│  │  │     (Use Cases)               │    │  │
│  │  └───────────────────────────────┘    │  │
│  │   (Controllers, Gateways, Presenters) │  │
│  └───────────────────────────────────────┘  │
│    (Web, DB, External APIs, UI)              │
└─────────────────────────────────────────────┘

依存の方向: 外側 → 内側（内側は外側を知らない）
```

#### 依存ルール

- **内側のレイヤーは外側のレイヤーを知らない**
- 依存は常に内側に向かう
- 内側のレイヤーが外側の機能を使いたい場合は **インターフェース（ポート）** を定義し、外側が実装する

### 3. 各レイヤーの役割（30分）

#### Entities（ドメイン層）

- ビジネスルールをカプセル化したエンティティと値オブジェクト
- 第5回（DDD）で学んだドメインモデルがここに位置する
- フレームワーク、DB、HTTPに一切依存しない

#### Use Cases（アプリケーション層）

- アプリケーション固有のビジネスルール
- ユースケースの入力（Input）と出力（Output）を明示的に定義
- ドメイン層のオブジェクトを操作してユースケースを実行
- 1ユースケース = 1クラスが基本

#### Interface Adapters（インターフェースアダプター層）

- ユースケースと外部世界の橋渡し
- Controller：HTTPリクエストをユースケースの入力に変換
- Presenter：ユースケースの出力をHTTPレスポンスに変換
- Gateway：リポジトリインターフェースの実装

#### Frameworks & Drivers（インフラストラクチャ層）

- Express / Prisma / 外部API クライアント等
- 最も「詳細」で最も「交換可能」なレイヤー

### 4. Node.js / TypeScript での実装（25分）

#### ディレクトリ構造

```
src/
├── domain/           # ドメイン層（Entities）
│   ├── entities/
│   ├── value-objects/
│   └── repositories/  # リポジトリインターフェース
├── application/      # アプリケーション層（Use Cases）
│   ├── use-cases/
│   └── dto/          # Input / Output
├── infrastructure/   # インフラストラクチャ層
│   ├── persistence/  # DB実装（Prisma等）
│   ├── external/     # 外部APIクライアント
│   └── config/
└── presentation/     # プレゼンテーション層
    ├── controllers/
    ├── routes/
    └── middleware/
```

#### 依存性注入（DI）

- クリーンアーキテクチャの依存ルールを実現する仕組み
- NestJSのDIコンテナ
- 手動DIによるシンプルな構成

### 5. よくある誤解と実践上の注意（10分）

- 「クリーンアーキテクチャ = ディレクトリ構造」ではない
- 小規模プロジェクトでの適用度合い — 全レイヤーが必要とは限らない
- 「完璧なクリーンアーキテクチャ」より「依存の方向を意識する」ことが重要

## コード例の概要

```typescript
// === ドメイン層 ===
// リポジトリインターフェース（ポート）
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// === アプリケーション層 ===
// ユースケースの入力
interface PlaceOrderInput {
  customerId: string;
  items: { productId: string; quantity: number }[];
  shippingAddress: { prefecture: string; city: string; line: string };
}

// ユースケースの出力
interface PlaceOrderOutput {
  orderId: string;
  totalAmount: number;
}

// ユースケース
class PlaceOrderUseCase {
  constructor(
    private readonly orderRepo: OrderRepository,
    private readonly productRepo: ProductRepository,
    private readonly eventPublisher: DomainEventPublisher
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    // 1. ドメインオブジェクトの生成
    const order = new Order(generateId(), input.customerId,
      Address.from(input.shippingAddress));

    // 2. 商品情報の取得と注文明細の追加
    for (const item of input.items) {
      const product = await this.productRepo.findById(item.productId);
      if (!product) throw new Error(`商品が見つかりません: ${item.productId}`);
      order.addItem(product.id, item.quantity, product.price);
    }

    // 3. 注文確定（ドメインイベント発行）
    const event = order.confirm();

    // 4. 永続化
    await this.orderRepo.save(order);
    await this.eventPublisher.publish(event);

    // 5. 出力
    return {
      orderId: order.id,
      totalAmount: order.totalAmount.amount,
    };
  }
}
```

```typescript
// === プレゼンテーション層 ===
// Express Controller
class OrderController {
  constructor(private readonly placeOrder: PlaceOrderUseCase) {}

  async create(req: Request, res: Response): Promise<void> {
    const input: PlaceOrderInput = {
      customerId: req.body.customerId,
      items: req.body.items,
      shippingAddress: req.body.shippingAddress,
    };

    const output = await this.placeOrder.execute(input);
    res.status(201).json(output);
  }
}

// === インフラストラクチャ層 ===
// リポジトリ実装（アダプター）
class PrismaOrderRepository implements OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });
    return data ? OrderMapper.toDomain(data) : null;
  }

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id },
      create: OrderMapper.toPersistence(order),
      update: OrderMapper.toPersistence(order),
    });
  }
}
```

## ディスカッションテーマ

1. **レイヤーの粒度** — 4つのレイヤーすべてが常に必要か？簡略化する場合、どこを省くか？
2. **DI の手動 vs フレームワーク** — NestJSのDIコンテナを使うべきか、手動DIで十分か？
3. **テスト戦略** — クリーンアーキテクチャにおいて、各レイヤーのテストはどう書くべきか？

## 課題

### 復習問題

1. クリーンアーキテクチャの「依存ルール」を自分の言葉で説明してください
2. ユースケース層がドメイン層のリポジトリインターフェースを使う仕組みを、依存性逆転の原則の観点から説明してください
3. 「フレームワークは詳細である」とはどういう意味か説明してください

### コード課題

第5回の課題で設計した「ホテル予約システム」のドメインモデルに、クリーンアーキテクチャのレイヤー構造を適用してください。

成果物：
- ディレクトリ構造の設計
- 「部屋を予約する」ユースケースクラスの実装
- リポジトリインターフェースの定義とインメモリ実装
- Controllerの実装（Express）

### 発展課題（任意）

自分のプロジェクトのコードを、クリーンアーキテクチャの4レイヤーに分類してみてください。依存ルールに違反している箇所を特定し、改善案を提示してください。

## 参考資料

- 『Clean Architecture』Robert C. Martin
- 『ソフトウェアアーキテクチャの基礎』Mark Richards, Neal Ford
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) — Uncle Bob's Blog

## 次回との接続

第6回までは「1つのアプリケーション内」の設計を学びました。第7回（マイクロサービス）からは、複数のアプリケーション（サービス）に分割する際の設計判断を学びます。DDDの境界づけられたコンテキストがサービス分割の単位になります。
