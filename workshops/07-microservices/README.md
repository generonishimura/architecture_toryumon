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

#### アーキテクチャの進化パス

```
モノリス → モジュラーモノリス → マイクロサービス

┌─────────────────┐    ┌─────────────────┐    ┌────┐ ┌────┐ ┌────┐
│                 │    │ ┌──┐ ┌──┐ ┌──┐ │    │    │ │    │ │    │
│  1つの大きな     │    │ │A │ │B │ │C │ │    │ A  │ │ B  │ │ C  │
│  アプリケーション │ →  │ └──┘ └──┘ └──┘ │ →  │    │ │    │ │    │
│                 │    │  モジュール境界   │    └────┘ └────┘ └────┘
└─────────────────┘    └─────────────────┘     独立したサービス
 デプロイ: 1つ          デプロイ: 1つ           デプロイ: 個別
 DB: 共有              DB: 共有(スキーマ分離)   DB: 個別
```

> **モジュラーモノリスの推奨**: Sam Newman は「まずモノリスから始めよ」と強調しています。最初からマイクロサービスに分割すると、ドメインの理解が浅い段階で誤った境界を引いてしまうリスクがあります。モジュラーモノリスで境界を検証してから分割する方が安全です。
>
> **マイクロサービスを選ぶべき明確なサイン**:
> - 異なるチームが同じコードベースで頻繁にコンフリクトする
> - 特定の機能だけを独立してスケーリングしたい（例: 検索は高負荷だが決済は低負荷）
> - 異なる技術スタック（例: ML部分はPython、APIはNode.js）が必要

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

```
REST vs gRPC の使い分け:

| 観点             | REST              | gRPC                    |
|-----------------|-------------------|-------------------------|
| プロトコル       | HTTP/1.1 (JSON)   | HTTP/2 (Protocol Buffers)|
| 性能             | 普通              | 高速（バイナリ転送）      |
| スキーマ         | OpenAPI（任意）    | .proto ファイル（必須）   |
| ブラウザ対応      | ✅ 直接呼べる      | ❌ プロキシが必要         |
| ストリーミング    | SSE等で部分対応    | ✅ 双方向ストリーミング    |
| 推奨場面         | 外部API、BFF       | サービス間の内部通信       |
```

#### 非同期通信

- メッセージキュー（RabbitMQ、Amazon SQS）
- イベントストリーミング（Apache Kafka）
- 同期 vs 非同期の選択基準
- （第9回のイベント駆動で詳細に学ぶ）

> **同期 vs 非同期の選択基準**:
> - **同期を選ぶ場面**: 呼び出し元が結果をすぐに必要とする（商品情報の取得、在庫の確認）
> - **非同期を選ぶ場面**: 結果が後で分かってもよい（メール送信、ログ記録、レポート生成）
> - **迷ったら**: まず同期で実装し、性能問題やサービス間結合が問題になったら非同期に移行

#### API設計のパターン

- BFF（Backend for Frontend）パターン
- API Gateway パターン
- API バージョニング戦略

### 4. データ管理（20分）

#### Database per Service

- 各サービスが自身のデータベースを持つ原則
- なぜ共有データベースはアンチパターンか
- データの一貫性をどう担保するか

```
【共有DBの問題】
                    ┌─────────────┐
  注文サービス ─────▶│             │◀───── 在庫サービス
  商品サービス ─────▶│   共有 DB    │◀───── ユーザーサービス
                    └─────────────┘
  問題:
  ・あるサービスのスキーマ変更が他サービスに波及する
  ・DBがボトルネックになりスケールできない
  ・サービスの独立デプロイが実質不可能に

【Database per Service】
  注文サービス ──▶ [注文DB]
  在庫サービス ──▶ [在庫DB]
  商品サービス ──▶ [商品DB]
  ユーザーサービス ──▶ [ユーザーDB]

  利点: 各サービスが独立してスキーマ変更・スケーリング可能
  課題: サービス間のデータ結合が複雑になる（→ API Composition / CQRS で解決）
```

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

> **可観測性の3本柱**: マイクロサービスでは「何が起きているか」を把握することが格段に難しくなります。以下の3つを整備することが必須です。
>
> | 柱 | 目的 | ツール例 |
> |-----|------|---------|
> | **ログ** | 各サービスの動作記録 | Elasticsearch + Kibana, Loki |
> | **メトリクス** | CPU、メモリ、レイテンシ等の数値指標 | Prometheus + Grafana |
> | **トレーシング** | リクエストがサービスを横断する経路の追跡 | Jaeger, Zipkin, OpenTelemetry |
>
> 特に分散トレーシングは重要です。1つのHTTPリクエストが5つのサービスを通過する場合、どこでレイテンシが発生しているかをトレースIDで追跡できます。

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

## 事前準備

### 参加者

#### 前提知識の確認

- [ ] 第5回のDDD（特に境界づけられたコンテキスト、コンテキストマップ）を復習済み
- [ ] 第6回のクリーンアーキテクチャ（レイヤー構造、依存ルール）を復習済み
- [ ] REST APIの基本（HTTPメソッド、ステータスコード、JSON）を理解している

#### 事前読み物（40分程度）

- [ ] 自分のプロジェクトのアーキテクチャが「モノリス」か「マイクロサービス」か、その理由を整理しておく
- [ ] 以下のキーワードについて概要を調べておく
  - モノリス vs マイクロサービス
  - API Gateway
  - コンウェイの法則
  - Circuit Breaker パターン
- [ ] 自分のチーム/組織構造と、担当しているサービスの構造の関係を考えておく

#### 事前ミニ課題（任意・20分程度）

以下のモノリスアプリケーションの機能一覧を見て、「もしマイクロサービスに分割するなら、どこで分割するか」を考えてメモしておいてください。分割の理由も合わせて考えてください。

> ECサイトの機能一覧：
> - ユーザー登録・ログイン・プロフィール管理
> - 商品の検索・閲覧・レビュー投稿
> - カート管理・注文確定
> - 在庫管理・入出庫
> - 決済処理（クレジットカード、コンビニ払い）
> - 配送手配・追跡
> - 管理画面（売上レポート、ユーザー管理）

### 運営側

- [ ] 第6回の課題提出状況の確認
- [ ] サービス構成図のテンプレート（テキストベース or Miro）の準備
- [ ] REST API設計の具体例（OpenAPI仕様等）の準備
- [ ] Circuit Breaker のデモ（意図的にサービスを停止させて動作確認）の準備
- [ ] モノリス → マイクロサービスの移行事例の資料準備

## 参考資料

- 『マイクロサービスアーキテクチャ』Sam Newman
- 『モノリスからマイクロサービスへ』Sam Newman
- 『マイクロサービスパターン』Chris Richardson
- [microservices.io](https://microservices.io/) — パターンカタログ

## 次回との接続

第7回でマイクロサービスの設計を学びました。第8回（オーケストレーション）では、これらのサービスをどうデプロイ・管理・スケーリングするかを学びます。コンテナ化、Kubernetes、CI/CDパイプラインを取り上げます。
