# 第6回：クリーンアーキテクチャ

> ビジネスロジックを外部の詳細から守ることが、変更容易性の鍵

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | 第1回（OOP、特に依存性逆転）、第4回（デザインパターン）、第5回（DDD） |
| 到達目標 | クリーンアーキテクチャの原則を理解し、レイヤー構造とその依存ルールに基づいた設計ができる |

## この回のキーメッセージ

> 「ビジネスロジックを外部の詳細から守ることが、変更容易性の鍵」

## 講義内容

### 1. なぜアーキテクチャが必要か（10分）

- ソフトウェアの変更コストは時間とともに増大する
- アーキテクチャの目的 = **変更容易性**の確保
- Robert C. Martin『Clean Architecture』の中心となる主張
- 関連するアーキテクチャパターン：Hexagonal / Onion / Clean

> **3つのアーキテクチャパターンの比較**: Hexagonal（ポート&アダプター）、Onion、Clean Architecture は表現が違いますが、核心は同じです。
>
> ```
> 共通の原則:
> 1. ビジネスロジックを中心に据える
> 2. 外部（DB、API、UI）は「詳細」であり、交換可能にする
> 3. 依存は常に内側（ビジネスロジック）に向かう
>
> Hexagonal:  「ポート（インターフェース）」と「アダプター（実装）」で表現
> Onion:      層を玉ねぎの皮のように表現。ドメイン → サービス → インフラ
> Clean:      同心円で4層に分けて表現。上記2つの統合・整理版
> ```
>
> 本ワークショップではClean Architecture の用語を使いますが、思想はどれも同じです。

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
- 内側のレイヤーから外側の機能を使う場合は、内側でインターフェース（ポート）を定義し、外側がそれを実装する

### 3. 各レイヤーの役割（30分）

#### Entities（ドメイン層）

- ビジネスルールをカプセル化したエンティティと値オブジェクト
- 第5回（DDD）で学んだドメインモデルをここに置く
- フレームワーク、DB、HTTPに一切依存しない

#### Use Cases（アプリケーション層）

- アプリケーション固有のビジネスルール
- ユースケースの入力（Input）と出力（Output）を明示的に定義
- ドメイン層のオブジェクトを操作してユースケースを実行
- 1ユースケース = 1クラスが基本

#### Interface Adapters（インターフェースアダプター層）

- ユースケースと外部（HTTPやDBなど）をつなぐ
- Controller：HTTPリクエストをユースケースの入力に変換
- Presenter：ユースケースの出力をHTTPレスポンスに変換
- Gateway：リポジトリインターフェースの実装

#### Frameworks & Drivers（フレームワーク・ドライバー層）

- Express / Prisma本体 / 外部APIのSDK・HTTPライブラリ等
- 最も「詳細」で最も「交換可能」なレイヤー

### 4. Node.js / TypeScriptでの実装（25分）

#### ディレクトリ構造

```
src/
├── main.ts           # コンポジションルート（Express アプリの起動、PrismaClient の生成）
├── generated/
│   └── prisma/       # Prisma Client の生成先（npx prisma generate で作られる）
├── domain/           # ドメイン層（Entities）
│   ├── entities/
│   ├── value-objects/
│   └── repositories/  # リポジトリインターフェース
├── application/      # アプリケーション層（Use Cases）
│   ├── use-cases/
│   └── dto/          # Input / Output
├── infrastructure/   # インフラストラクチャ（同心円の層との対応は下の表を参照）
│   ├── persistence/  # DB実装（Prisma等）
│   ├── external/     # 外部APIクライアント
│   └── config/
└── presentation/     # プレゼンテーション層
    ├── controllers/
    ├── routes/
    └── middleware/
```

#### 同心円のレイヤーとディレクトリの対応

| 同心円のレイヤー | 本教材のディレクトリ | 例 |
|------------------|----------------------|----|
| Entities | domain/ | Order, Money, OrderRepository（interface） |
| Use Cases | application/ | PlaceOrderUseCase, PlaceOrderInput / PlaceOrderOutput |
| Interface Adapters | presentation/controllers/・presentation/routes/（入力側）、infrastructure/persistence/（出力側＝Gateway） | OrderController, ルートの定義, PrismaOrderRepository |
| Frameworks & Drivers | src/main.ts、infrastructure/config/、Express・Prisma本体 | Expressアプリの起動とPrismaClientの生成（src/main.ts）、infrastructure/config/ |

ディレクトリ名と同心円のレイヤー名は、1対1には対応しない。infrastructure/の中では、persistence/がInterface Adapters、config/がFrameworks & Driversにあたる。external/には、外部APIのクライアントを置く。このうち、ユースケースが使うインターフェースを実装するクラス（例: 決済APIのクライアントの実装）は、Interface AdaptersのGatewayである。そのクラスが呼び出すSDKやHTTPライブラリ本体は、Frameworks & Driversに属する。presentation/のcontrollers/とroutes/は、Interface Adaptersの入力側にあたる。src/main.tsはFrameworks & Driversに含める。

本教材のコード例では、ユースケースがOutputをreturnし、ControllerがそれをHTTPレスポンスに変換している。原典では、ユースケースがOutput Port（インターフェース）を呼び、Presenterがそれを実装する。小規模なアプリケーションなら、returnする方式で十分である。

#### 依存性注入（DI）

- クリーンアーキテクチャの依存ルールを実現する仕組み
- NestJSのDIコンテナ
- 手動DIによるシンプルな構成

手動DIでは、アプリケーションのエントリーポイント（src/main.ts）で具体クラスを生成し、コンストラクタに渡して組み立てる。

```typescript
// src/main.ts — コンポジションルート（依存を組み立てる唯一の場所）
// Prisma 7 は .env を自動で読み込まない。アプリのエントリーポイントの先頭で読み込む
import 'dotenv/config';
import express from 'express';
import { PrismaPg } from '@prisma/adapter-pg';
// Prisma 7 では、schema.prisma の generator で指定した output（"../src/generated/prisma"）から import する
// このファイルは src/ の直下にあるので、相対パスは './generated/prisma/client' になる
import { PrismaClient } from './generated/prisma/client';
// （各レイヤーのクラスの import は省略）

// Prisma 7 では、PrismaClient に driver adapter を渡す
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
const prisma = new PrismaClient({ adapter });

const placeOrder = new PlaceOrderUseCase(
  new PrismaOrderRepository(prisma),
  new PrismaProductRepository(prisma),
  new InMemoryDomainEventPublisher(),
);
const orderController = new OrderController(placeOrder);

const app = express();
app.use(express.json());
// 簡略化のため、ルートの定義を main.ts に書いている。実際のプロジェクトでは presentation/routes/ に置く
app.post('/orders', (req, res, next) => {
  orderController.create(req, res).catch(next);
});
app.listen(3000);
```

外側の具体クラス（PrismaOrderRepositoryなど）をimportして生成するのは、main.tsだけである。domain/とapplication/は、インターフェースにだけ依存する。

- prisma.config.tsにもimport 'dotenv/config'を書くが、この記述で.envが読み込まれるのは、Prisma CLI（npx prisma migrateなど）を実行するときだけである。アプリをnpx tsx src/main.tsで実行するときは、main.tsの先頭のimport 'dotenv/config'が.envを読み込む
- ExpressをTypeScriptで使うときは、expressに加えて型定義の@types/expressも必要である（npm install expressとnpm install -D @types/express）

> **手動DIとDIコンテナ**：サービス数が少ない（10程度まで）なら、手動DIで十分である。サービス数が増えたら、NestJSのDIコンテナを検討する。どちらを選んでも、依存はコンストラクタで外から渡す。

### 5. よくある誤解と実践上の注意（10分）

- 「クリーンアーキテクチャ = ディレクトリ構造」ではない
- 小規模プロジェクトでの適用度合い：全レイヤーが必要とは限らない
- 「完璧なクリーンアーキテクチャ」を目指すより、「依存の方向を意識する」ことを優先する

#### 規模に応じた適用レベル

```
レベル1: 最小限（小規模プロジェクト）
  「ビジネスロジックにフレームワーク/DB依存のコードを書かない」だけ意識
  → 2層（ドメイン+アプリ / インフラ+プレゼン）で十分

レベル2: 標準（中規模プロジェクト）
  4層をきちんと分離し、リポジトリインターフェースを使う
  → 本講義で学ぶ構成

レベル3: 厳格（大規模・長寿命プロジェクト）
  モジュール単位でパッケージを分離し、ビルド時に依存方向を強制
  → 大規模チーム向け
```

#### クリーンアーキテクチャの各レイヤーのテスト戦略

| レイヤー | テスト種別 | 特徴 |
|---------|----------|------|
| ドメイン層 | ユニットテスト | 外部依存なし。純粋なロジックのテスト。最も高速 |
| アプリケーション層 | ユニットテスト（モック使用） | リポジトリをモックに差し替えてユースケースをテスト |
| infrastructure/persistence/（Gateway） | 統合テスト | 実際のDB（テスト用）を使った永続化のテスト |
| プレゼンテーション層 | E2Eテスト / APIテスト | HTTPリクエスト→レスポンスの全体フローをテスト |

> **テストピラミッドとの関係**: ドメイン層のテストが最も多く・高速に、E2Eテストが最も少なく・低速に。クリーンアーキテクチャはこのテストピラミッドを自然に実現します。

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
    // 簡略化のため直接 publish している。本番では保存と同じトランザクションで outbox に書き込む（第9回で扱う）
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
// === プレゼンテーション層（presentation/controllers/。同心円では Interface Adapters の入力側） ===
// src/presentation/controllers/OrderController.ts
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

// === Interface Adapters 層の Gateway（infrastructure/persistence/ に配置） ===
// src/infrastructure/persistence/PrismaOrderRepository.ts
// リポジトリ実装（アダプター）
// PrismaClient は Prisma 7 の生成先（src/generated/prisma/client）から import する
// このファイルは src/infrastructure/persistence/ にあるので、相対パスは '../../generated/prisma/client' になる
import { PrismaClient } from '../../generated/prisma/client';

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

1. **レイヤーの粒度**：4つのレイヤーすべてが常に必要か？簡略化する場合、どこを省くか？
2. **DIの手動 vs フレームワーク**：NestJSのDIコンテナを使うべきか、手動DIで十分か？
3. **テスト戦略**：クリーンアーキテクチャにおいて、各レイヤーのテストはどう書くべきか？

## 課題

### 復習問題

1. クリーンアーキテクチャの「依存ルール」を自分の言葉で説明してください
2. ユースケース層がドメイン層のリポジトリインターフェースを使う仕組みを、依存性逆転の原則の観点から説明してください
3. 「フレームワークは詳細である」とはどういう意味か説明してください

### コード課題

第5回の課題で設計した「ホテル予約システム」のドメインモデルに、クリーンアーキテクチャのレイヤー構造を適用してください。

成果物として、次の4つを提出してください。

- ディレクトリ構造の設計
- 「部屋を予約する」ユースケースクラスの実装
- リポジトリインターフェースの定義とインメモリ実装
- Controllerの実装（Express）

### 発展課題（任意）

自分のプロジェクトのコードを、クリーンアーキテクチャの4レイヤーに分類してみてください。依存ルールに違反している箇所を特定し、改善案を提示してください。

## 事前準備

### 参加者

#### 前提知識の確認

- [ ] 第1回の依存性逆転の原則（DIP）を復習済み — 「抽象に依存する」意味が説明できる
- [ ] 第4回のデザインパターン（Adapter、Repository）を復習済み
- [ ] 第5回のDDD（エンティティ、値オブジェクト、集約、ドメインサービス）を復習済み

#### 事前読み物（30分程度）

- [ ] 自分のプロジェクトのディレクトリ構造を確認し、「ビジネスロジック」と「フレームワーク/DB依存のコード」がどこに置かれているか整理しておく
- [ ] 以下のキーワードについて概要を調べておく
  - クリーンアーキテクチャの同心円モデル
  - ヘキサゴナルアーキテクチャ（ポートとアダプター）
  - ユースケース（アプリケーションサービス）

#### 事前ミニ課題（任意・15分程度）

以下のコードで、どの部分が「ビジネスロジック」でどの部分が「外部の詳細（フレームワーク・DB）」か、線引きしてみてください。

```typescript
app.post('/orders', async (req, res) => {
  const { customerId, items } = req.body;
  const customer = await prisma.customer.findUnique({ where: { id: customerId } });
  if (!customer) return res.status(404).json({ error: 'Customer not found' });

  let total = 0;
  for (const item of items) {
    const product = await prisma.product.findUnique({ where: { id: item.productId } });
    if (product.stock < item.quantity) {
      return res.status(400).json({ error: `${product.name} の在庫が不足しています` });
    }
    total += product.price * item.quantity;
  }

  const order = await prisma.order.create({
    data: { customerId, totalAmount: total, items: { create: items } },
  });
  res.status(201).json({ orderId: order.id });
});
```

### 運営側

- [ ] 第5回の課題提出状況の確認（ドメインモデル設計の理解度）
- [ ] 同心円モデルの図解スライドの準備
- [ ] ディレクトリ構造の例をプロジェクトテンプレートとして準備
- [ ] ユースケースクラスのライブコーディングデモの準備
- [ ] 「クリーンアーキテクチャの過剰適用」の失敗事例の資料準備

## 参考資料

- 『Clean Architecture』Robert C. Martin
- 『ソフトウェアアーキテクチャの基礎』Mark Richards, Neal Ford
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)：Uncle Bob's Blog

## 次回との接続

第6回までは「1つのアプリケーション内」の設計を学びました。第7回（マイクロサービス）からは、複数のアプリケーション（サービス）に分割する際の設計判断を学びます。DDDの境界づけられたコンテキストがサービス分割の単位になります。
