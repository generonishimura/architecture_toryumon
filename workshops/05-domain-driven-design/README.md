# 第5回：ドメイン駆動設計（DDD）

> ビジネスの言葉でコードを書くことが、最も強力な設計手法

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第1回（OOP）、第3回（データモデリング）、第4回（デザインパターン） |
| 到達目標 | DDDの戦略的設計・戦術的設計を理解し、ドメインモデルの構築ができる |

## この回のキーメッセージ

> 「ビジネスの言葉でコードを書くことが、最も強力な設計手法」

## 講義内容

### 1. DDDの概要と背景（15分）

- Eric Evans『Domain-Driven Design』（2003）の本質
- DDDは「技術的な手法」ではなく「ビジネスとの向き合い方」
- DDDが解決する問題 — ビジネスとコードの乖離
- DDD = 戦略的設計 + 戦術的設計

### 2. 戦略的設計（30分）

#### ユビキタス言語（Ubiquitous Language）

- チーム全員（開発者 + ドメインエキスパート）が共通で使う言葉
- コード上の名前もユビキタス言語に従う
- 例：「注文」「配送」「請求」— ビジネスの言葉がそのままコードに

#### 境界づけられたコンテキスト（Bounded Context）

- 同じ用語が異なる意味を持つ場合の境界の設定
- 例：「商品」は在庫管理と商品カタログで意味が異なる
- コンテキストマップ — コンテキスト間の関係の可視化
- 関係パターン：共有カーネル、顧客/供給者、腐敗防止層

#### コンテキストマップの実践

```
┌──────────────────┐     ┌──────────────────┐
│  商品カタログ      │     │  在庫管理         │
│  ──────────       │     │  ──────          │
│  商品: 名前、説明、│ ACL │  商品: SKU、数量、│
│  カテゴリ、価格    │◄───│  倉庫、入出庫     │
│                   │     │                  │
└──────────────────┘     └──────────────────┘
         │                         │
         │ 公開ホスト               │ 順応者
         ▼                         ▼
┌──────────────────┐     ┌──────────────────┐
│  注文管理         │     │  配送管理         │
│  ──────          │     │  ──────          │
│  注文: 明細、合計、│────▶│  配送: 宛先、     │
│  ステータス       │     │  追跡番号、状態   │
└──────────────────┘     └──────────────────┘
```

### 3. 戦術的設計（40分）

#### エンティティ（Entity）

- 一意の識別子を持ち、ライフサイクルを通じて同一性を保つ
- 属性が変わっても「同じもの」であると判断できる
- TypeScript例：`Order` エンティティ

#### 値オブジェクト（Value Object）

- 識別子を持たず、属性の値そのものが意味を持つ
- イミュータブル（不変）であること
- TypeScript例：`Money`、`EmailAddress`、`Address`

#### 集約（Aggregate）

- 関連するエンティティと値オブジェクトのまとまり
- 集約ルート（Aggregate Root）を通じてのみ操作する
- トランザクション整合性の境界
- 集約の設計指針 — 小さく保つ

#### ドメインサービス

- エンティティや値オブジェクトに属さないドメインロジック
- ステートレスな操作
- TypeScript例：送金処理、在庫引当

#### ドメインイベント

- ドメイン内で発生した重要な出来事を表現
- 「〇〇が起こった」という過去形で命名
- TypeScript例：`OrderPlaced`、`PaymentCompleted`

#### リポジトリ（再掲・深掘り）

- 第4回で学んだRepositoryパターンのDDD文脈での役割
- 集約ルート単位で永続化と取得を行う

## コード例の概要

```typescript
// 値オブジェクト
class Money {
  private constructor(
    readonly amount: number,
    readonly currency: string
  ) {
    if (amount < 0) throw new Error('金額は0以上である必要があります');
  }

  static of(amount: number, currency: string = 'JPY'): Money {
    return new Money(amount, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error('通貨が異なります');
    }
    return Money.of(this.amount + other.amount, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
}
```

```typescript
// エンティティ（集約ルート）
class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus = OrderStatus.DRAFT;

  constructor(
    readonly id: string,
    readonly customerId: string,
    private _shippingAddress: Address
  ) {}

  addItem(productId: string, quantity: number, unitPrice: Money): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new Error('確定済みの注文には商品を追加できません');
    }
    this._items.push(new OrderItem(productId, quantity, unitPrice));
  }

  get totalAmount(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.of(0)
    );
  }

  confirm(): OrderPlaced {
    if (this._items.length === 0) {
      throw new Error('商品が1つもない注文は確定できません');
    }
    this._status = OrderStatus.CONFIRMED;
    // ドメインイベントを返す
    return new OrderPlaced(this.id, this.customerId, this.totalAmount);
  }

  get items(): ReadonlyArray<OrderItem> {
    return [...this._items];
  }

  get status(): OrderStatus {
    return this._status;
  }
}
```

```typescript
// ドメインイベント
class OrderPlaced {
  readonly occurredAt: Date = new Date();

  constructor(
    readonly orderId: string,
    readonly customerId: string,
    readonly totalAmount: Money
  ) {}
}
```

## ディスカッションテーマ

1. **ユビキタス言語の難しさ** — 開発者とビジネスサイドで言葉が異なる場合、どう擦り合わせるか？
2. **集約の境界** — 集約を大きくしすぎた/小さくしすぎた場合、それぞれどんな問題が起きるか？
3. **DDDの導入判断** — すべてのプロジェクトにDDDを適用すべきか？適用すべきでない場面は？

## 課題

### 復習問題

1. エンティティと値オブジェクトの違いを、具体例を交えて説明してください
2. 集約ルートの役割と、なぜ集約ルートを通じてのみ操作すべきかを説明してください
3. 境界づけられたコンテキストが必要になる場面の例を1つ挙げてください

### コード課題

「ホテル予約システム」のドメインモデルをTypeScriptで設計してください。

要件：
- 宿泊客が客室を予約できる
- 客室には種類（シングル、ダブル、スイート）と料金がある
- 予約にはチェックイン日・チェックアウト日がある
- 同じ客室の同一日程に複数の予約はできない
- 予約をキャンセルできる（チェックイン前日まで）

成果物：
- 値オブジェクト、エンティティ、集約の識別
- TypeScriptのコード（ドメインモデルの実装）
- ビジネスルールがドメインモデル内に表現されていること

### 発展課題（任意）

自分のプロジェクトの一部の機能について、コンテキストマップを描いてください。境界づけられたコンテキストを識別し、コンテキスト間の関係パターン（共有カーネル、ACL等）を明示してください。

## 参考資料

- 『ドメイン駆動設計入門』成瀬允宣
- 『エリック・エヴァンスのドメイン駆動設計』Eric Evans
- 『実践ドメイン駆動設計』Vaughn Vernon

## 次回との接続

第5回で設計したドメインモデルを、アプリケーション全体のどこに配置し、外部（DB、API、UI）とどう接続するか。第6回（クリーンアーキテクチャ）では、ドメインモデルを中心に据えたアプリケーション構造を学びます。
