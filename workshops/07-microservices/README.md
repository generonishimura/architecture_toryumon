# 第7回：マイクロサービス

> 分割はコストを伴う。モノリスから始めて、必要に応じて分割する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第5回（DDD、特に境界づけられたコンテキスト）、第6回（クリーンアーキテクチャ） |
| 到達目標 | マイクロサービスの利点・課題・トレードオフを理解し、サービス分割の判断基準を持てる |

## この回のキーメッセージ

> 「分割はコストを伴う。モノリスから始めて、必要に応じて分割する」

## 講義内容

### 1. モノリスからマイクロサービスへ（15分）

- モノリスアーキテクチャの特徴と課題
- マイクロサービスとは何か — 独立してデプロイ可能な小さなサービスの集合
- マイクロサービスが登場した背景（組織のスケーリング、デプロイ頻度の向上）
- **モジュラーモノリス** — マイクロサービスの前段階としての選択肢

### 2. サービス分割の原則（25分）

#### DDDの境界づけられたコンテキスト = サービス境界

- 第5回で学んだコンテキストマップをサービス境界に適用
- 1つの境界づけられたコンテキスト ≒ 1つのマイクロサービス
- コンウェイの法則 — 組織構造がアーキテクチャに反映される

#### 分割の判断基準

- 独立した開発・デプロイの必要性
- スケーリング要件の違い
- 技術スタックの独立性
- チームの自律性

#### 分割しすぎの危険

- ネットワーク通信のオーバーヘッド
- 分散トランザクションの複雑さ
- 運用・監視コストの増大
- 「ナノサービス」の罠

### 3. サービス間通信（25分）

#### 同期通信

- REST API
  - RESTful設計の基本原則
  - HTTPメソッドとステータスコードの適切な使用
- gRPC
  - Protocol Buffers によるスキーマ定義
  - RESTとの使い分け

#### 非同期通信

- メッセージキュー（RabbitMQ、Amazon SQS）
- イベントストリーミング（Apache Kafka）
- 同期 vs 非同期の選択基準
- （第9回のイベント駆動で詳細に学ぶ）

#### API設計のパターン

- BFF（Backend for Frontend）パターン
- API Gateway パターン
- API バージョニング戦略

### 4. データ管理（20分）

#### Database per Service

- 各サービスが自身のデータベースを持つ原則
- なぜ共有データベースはアンチパターンか
- データの一貫性をどう担保するか

#### データの結合

- API Composition パターン — 複数サービスのデータを結合
- CQRS（Command Query Responsibility Segregation）の概要
- （第9回で詳細に学ぶ）

### 5. 実践的な考慮事項（15分）

#### サービス間の障害耐性

- Circuit Breaker パターン
- タイムアウトとリトライ戦略
- Bulkhead パターン

#### 可観測性（Observability）

- 分散トレーシング
- 集約ログ
- メトリクス監視

#### Node.js でのマイクロサービス構築

- Express / Fastify による軽量サービス
- NestJS Microservices モジュール
- gRPC + Node.js

## コード例の概要

```typescript
// サービス間のREST通信（クライアント側）
// 注文サービスが商品サービスに問い合わせる

interface ProductServiceClient {
  getProduct(productId: string): Promise<ProductDto | null>;
}

class HttpProductServiceClient implements ProductServiceClient {
  constructor(private readonly baseUrl: string) {}

  async getProduct(productId: string): Promise<ProductDto | null> {
    const response = await fetch(
      `${this.baseUrl}/api/products/${productId}`
    );
    if (response.status === 404) return null;
    if (!response.ok) throw new Error(`Product service error: ${response.status}`);
    return response.json();
  }
}
```

```typescript
// Circuit Breaker パターンの概念
class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime: number | null = null;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';

  constructor(
    private readonly threshold: number = 5,
    private readonly resetTimeout: number = 30000
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - (this.lastFailureTime ?? 0) > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit is OPEN — request blocked');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

```typescript
// API Gateway 的なBFFの例
// フロントエンドが必要なデータを1リクエストで取得

class OrderDetailBff {
  constructor(
    private readonly orderService: OrderServiceClient,
    private readonly productService: ProductServiceClient,
    private readonly userService: UserServiceClient
  ) {}

  async getOrderDetail(orderId: string) {
    const order = await this.orderService.getOrder(orderId);
    if (!order) throw new Error('Order not found');

    // API Composition: 複数サービスのデータを並行取得
    const [customer, products] = await Promise.all([
      this.userService.getUser(order.customerId),
      Promise.all(
        order.items.map(item =>
          this.productService.getProduct(item.productId)
        )
      ),
    ]);

    return {
      orderId: order.id,
      status: order.status,
      customer: { name: customer?.name, email: customer?.email },
      items: order.items.map((item, i) => ({
        productName: products[i]?.name,
        quantity: item.quantity,
        unitPrice: item.unitPrice,
      })),
      totalAmount: order.totalAmount,
    };
  }
}
```

## ディスカッションテーマ

1. **モノリス vs マイクロサービス** — 自分のプロジェクトはマイクロサービスにすべきか？その判断理由は？
2. **サービス分割の粒度** — 「ちょうどよい大きさ」のサービスとは？具体的な判断基準を議論
3. **コンウェイの法則** — チーム構造とアーキテクチャの関係は、自分の組織ではどう影響しているか？

## 課題

### 復習問題

1. マイクロサービスの3つの利点と3つの課題を挙げてください
2. 「Database per Service」の原則がなぜ重要か、共有DBの問題点と合わせて説明してください
3. 同期通信（REST）と非同期通信（メッセージキュー）の使い分けの基準を説明してください

### コード課題

ECサイトをマイクロサービスとして設計してください。

要件：
- 商品カタログ、注文管理、在庫管理、ユーザー管理 の4つのサービスに分割
- 各サービスのAPI設計（エンドポイント一覧）
- サービス間の通信方式の決定と理由

成果物：
- サービス構成図（テキストベース）
- 各サービスのAPI仕様（REST）
- サービス間通信のシーケンス図（テキストベース）：「注文を確定する」フロー

### 発展課題（任意）

2つのマイクロサービス（注文サービスと在庫サービス）をNode.js / Expressで実装し、REST APIで通信させてください。Circuit Breakerパターンを適用してください。

## 参考資料

- 『マイクロサービスアーキテクチャ』Sam Newman
- 『モノリスからマイクロサービスへ』Sam Newman
- 『マイクロサービスパターン』Chris Richardson
- [microservices.io](https://microservices.io/) — パターンカタログ

## 次回との接続

第7回でマイクロサービスの設計を学びました。第8回（オーケストレーション）では、これらのサービスをどうデプロイ・管理・スケーリングするかを学びます。コンテナ化、Kubernetes、CI/CDパイプラインを取り上げます。
