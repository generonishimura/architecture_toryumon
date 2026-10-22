# 開発手法とアーキテクチャの関係

> 本ドキュメントは、アーキテクチャ登竜門の補足資料です。ソフトウェア開発手法とアーキテクチャ設計の関係をまとめています。ワークショップ本編では扱いきれませんが、アーキテクチャを実務で使うときに必要になる知識です。

---

## なぜ開発手法を知る必要があるか

アーキテクチャを学ぶと「最初から完璧な設計をしなければ」と考えがちである。しかし、実際のソフトウェア開発では以下の現実がある。

- 要件はプロジェクト中に変わる
- 最初にすべてを見通すことは不可能
- 過剰な事前設計は、使われない抽象化や不要な柔軟性を生む

開発手法を理解すると、「いつ・どの程度の設計をするか」を判断できるようになる。

---

## 1. 進化的アーキテクチャ（Evolutionary Architecture）

### 基本的な考え方

> アーキテクチャは、最初に一度で決めきるものではなく、時間をかけて少しずつ変えていくものである。

Neal FordとRebecca Parsonsが提唱する進化的アーキテクチャでは、次の3つを重視する。

- **段階的な変更（Incremental Change）**：小さな変更を頻繁に行い、フィードバックを得る
- **適応度関数（Fitness Functions）**：アーキテクチャの品質を自動的に検証する仕組み
- **適切な結合（Appropriate Coupling）**：変更の影響範囲を制御するための意図的な境界設計

### ワークショップとの関連

| ワークショップ回 | 関連する進化的アーキテクチャの考え方 |
|----------------|----------------------------------|
| 第4回 デザインパターン | パターンは最初から入れず、必要になったときに適用する |
| 第5回 DDD | 境界づけられたコンテキストは、サービス分割の将来の候補を示す |
| 第6回 クリーンアーキテクチャ | 依存性逆転により、インフラの差し替えが容易になる |
| 第7回 マイクロサービス | モノリスから始めて、システムの規模が大きくなったらサービスを切り出す |

### 実践のポイント

```
❌ ウォーターフォール的思考:
   「すべてのコンポーネントを事前に設計し、設計書通りに実装する」

✅ 進化的思考:
   「現在の要件に十分な設計から始め、学びに応じて設計を進化させる」
```

---

## 2. YAGNI原則（You Ain't Gonna Need It）

### 基本的な考え方

> 実際に必要になるまで、機能や抽象化を追加してはならない。

YAGNI原則はエクストリームプログラミング（XP）で提唱された原則で、次のようなことを戒める。

- 「将来使うかもしれない」抽象化の事前実装
- 現時点で必要のないデザインパターンの適用
- 使われることのない設定の柔軟性

### アーキテクチャ設計への適用

次のコードは過剰設計の例である。

```typescript
// ❌ 現在のシステムにはメール通知しかないのに、汎用通知基盤を作ってしまう
interface NotificationChannel {
  send(message: Message): Promise<void>;
}

class EmailChannel implements NotificationChannel { /* ... */ }
class SlackChannel implements NotificationChannel { /* ... */ }  // 使う予定なし
class SmsChannel implements NotificationChannel { /* ... */ }    // 使う予定なし
class PushChannel implements NotificationChannel { /* ... */ }   // 使う予定なし

class NotificationRouter {
  constructor(private channels: Map<string, NotificationChannel>) {}
  async route(message: Message, channelType: string): Promise<void> {
    const channel = this.channels.get(channelType);
    if (!channel) throw new Error(`Unknown channel: ${channelType}`);
    await channel.send(message);
  }
}
```

```typescript
// ✅ 現在の要件（メール通知のみ）に十分な設計
class EmailNotifier {
  async notify(to: string, subject: string, body: string): Promise<void> {
    // メール送信の実装
  }
}

// Slack通知が本当に必要になったとき、そのときにインターフェースを抽出する
```

### 判断の基準

次の問いで、YAGNIに反していないか確認できる。

1. 今のユーザーストーリーにこの抽象化は必要か？
2. この柔軟性を使う具体的なシナリオが、直近のロードマップにあるか？
3. この構造を追加すると、現在のコードは理解しにくくならないか？

---

## 3. ADR（Architecture Decision Records）

### 基本的な考え方

> 設計判断の「なぜ」を記録しておけば、後からチームに加わったメンバーが同じ議論を繰り返さずに済む。

### ADRの構成

```markdown
# ADR-001: 認証基盤にFirebase Authenticationを採用する

## ステータス
承認済み（2024-01-15）

## コンテキスト
ユーザー認証機能を実装する必要がある。チームにはセキュリティの専門家がおらず、
認証基盤を自前で構築するリスクが高い。

## 決定
Firebase Authenticationを採用する。

## 理由
- 自前実装のセキュリティリスクを回避できる
- ソーシャルログイン（Google, GitHub）に標準対応
- チームの学習コストが低い（ドキュメントが充実）

## 検討した代替案
- **Auth0**：機能は豊富だが、無料枠の制限が厳しい
- **自前実装（Passport.js）**：柔軟だが、セキュリティの担保が困難

## 結果
Firebase Authenticationで認証基盤を構築し、3ヶ月後に
パフォーマンスとコストを再評価する。
```

### ワークショップとの関連

本ワークショップで学ぶ設計判断は、すべてADRの対象となり得る。

- 「なぜモノリスではなくマイクロサービスを選んだか」
- 「なぜCQRSを導入したか」
- 「なぜこの集約の境界にしたか」

ADRを書き続けると、設計判断の理由を文章で説明できるようになる。これは本ワークショップの目標「設計判断の引き出しを増やす」につながる。

---

## 4. 技術的負債の管理

### 基本的な考え方

> 技術的負債は、それ自体が悪いものではない。問うべきなのは、その負債を意図して抱えたのか、気づかないうちに生まれたのかである。

### 技術的負債の4象限（Martin Fowler）

|  | 意図的 | 無意識 |
|--|--------|--------|
| **慎重** | 「今はこの設計で出荷し、次のスプリントでリファクタリングする」 | 「レイヤー分離にはもっと良い方法があったと後から気づいた」 |
| **無謀** | 「設計する時間がないからとりあえず動くコードを書く」 | 「そもそもレイヤー分離が何かを知らなかった」 |

### アーキテクチャ学習との関係

本ワークショップを受講すると、次のような変化が期待できる。

- **無意識の負債が減る**：設計の選択肢を知っていれば、知らないうちに負債を作ることが少なくなる
- **意図的な判断ができる**：「今は第3正規形で十分。パフォーマンス問題が出たら非正規化を検討する」のように、根拠を持って判断できる
- **負債の返済計画が立てられる**：リファクタリングでどのパターンを目指すかを決められる

---

## 5. テスト戦略とアーキテクチャ

### テストピラミッド

```
         /  E2E   \        ← 少数・遅い・高コスト
        / テスト    \          ブラウザ操作、API全体の検証
       /____________\
      /  統合テスト   \      ← 中程度
     /   （Integration）\      DB接続、外部API連携の検証
    /____________________\
   /    単体テスト         \  ← 多数・速い・低コスト
  /    （Unit Test）        \    ビジネスロジックの検証
 /__________________________\
```

### クリーンアーキテクチャとテスタビリティ

クリーンアーキテクチャを実務で使う大きな利点の一つは、テスタビリティ（テストのしやすさ）である。

```typescript
// ❌ ビジネスロジックが DB クライアントやメッセージブローカーのクライアント（Frameworks & Drivers）を直接 new している → テストが困難
class OrderService {
  async createOrder(items: Item[]): Promise<Order> {
    const db = new PostgresClient();  // テスト時にDBが必要
    const order = new Order(items);
    await db.query('INSERT INTO orders ...', [order]);

    const producer = new KafkaProducer();  // テスト時にメッセージブローカーが必要
    await producer.send('OrderPlaced', { orderId: order.id });

    return order;
  }
}

// ✅ 依存性逆転で、DB とイベントの送信を Use Cases の外に出す → 単体テストが容易
// （確認メールは第9回のとおり、決済完了（PaymentCompleted）の後に別のハンドラが送る）
interface OrderRepository {
  save(order: Order): Promise<void>;
}
type OrderPlaced = { type: 'OrderPlaced'; orderId: string };
interface DomainEventPublisher {
  publish(event: OrderPlaced): Promise<void>;
}

class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private events: DomainEventPublisher,
  ) {}

  async execute(items: Item[]): Promise<Order> {
    const order = new Order(items);
    await this.orderRepo.save(order);
    await this.events.publish({ type: 'OrderPlaced', orderId: order.id });
    return order;
  }
}

// テスト: DBもメッセージブローカーも不要
test('注文を作成すると OrderPlaced が発行される', async () => {
  const fakeRepo: OrderRepository = { save: async () => {} };
  const published: OrderPlaced[] = [];
  const fakeEvents: DomainEventPublisher = { publish: async (e) => { published.push(e); } };
  const useCase = new CreateOrderUseCase(fakeRepo, fakeEvents);

  const order = await useCase.execute([{ name: 'Book', price: 1500 }]);

  expect(order.items).toHaveLength(1);
  expect(published).toEqual([{ type: 'OrderPlaced', orderId: order.id }]);
});
```

### TDDの基本サイクル

```
Red    → テストを書く（まだ実装がないので失敗する）
Green  → テストが通る最小限の実装を書く
Refactor → コードを整理する（テストが通り続けることを確認）
```

TDDではテストを先に書くので、設計はテストを書きやすい構造になりやすい。テストが書きにくいと感じたときは、設計に改善の余地がある。

---

## 6. アジャイル開発とアーキテクチャの関係

### よくある誤解

> 「アジャイルではアーキテクチャ設計をしない」

これは誤りである。アジャイルでも設計はする。事前に完璧な設計を作り込まず、開発を進めながら必要な分だけ設計する。

### 正しい理解

| ウォーターフォール的アプローチ | アジャイル的アプローチ |
|-----------------------------|---------------------|
| 最初に完全な設計書を作成 | 最初は「十分な」設計で始める |
| 設計変更は「手戻り」として忌避 | 設計変更は「学び」として歓迎 |
| アーキテクトが設計し、開発者が実装 | チーム全体でアーキテクチャを決め、改善する |
| 設計は最初のフェーズで完了 | 設計は開発を通して継続的に見直す |

### スプリント内でのアーキテクチャ活動

```
スプリント計画時:
  → 今回のユーザーストーリーに必要な設計判断を議論
  → 必要であればADRを作成

スプリント中:
  → 実装しながら設計の妥当性を検証
  → 想定と違った場合は設計を調整

スプリントレビュー:
  → 技術的な学びを共有
  → 次のスプリントで必要な設計改善を特定

レトロスペクティブ:
  → アーキテクチャ上の課題を振り返り
  → 技術的負債の棚卸し
```

---

## 7. 段階的移行パターン

既存システムのアーキテクチャを少しずつ改善するときに使えるパターンを2つ挙げる。

### ストラングラーフィグパターン（Strangler Fig Pattern）

モノリスからマイクロサービスへ移行するときに、最もよく使われるパターンである。

```
[Phase 1] モノリスの前にルーターを配置
  Client → Router → Monolith

[Phase 2] 新機能を新サービスで実装、既存機能は徐々に移行
  Client → Router → Monolith（既存機能の一部）
                  → New Service A（新機能）
                  → New Service B（移行済み機能）

[Phase 3] モノリスの機能がすべて移行されたら除去
  Client → Router → Service A
                  → Service B
                  → Service C
```

### フィーチャーフラグ（Feature Flags）

新しいアーキテクチャへの移行を段階的に進めるための仕組みである。次の例では、確定済みの注文を処理するときに、フラグの値で新旧どちらの実装を使うかを切り替える。

```typescript
// フィーチャーフラグで新旧の実装を切り替える（order は確定済み = CONFIRMED）
async function processOrder(order: Order): Promise<void> {
  if (featureFlags.isEnabled('use-event-driven-order')) {
    // 新しいイベント駆動の実装（第9回の Saga と同じ流れ）
    // OrderPlaced を発行すると、各サービスが在庫引当、決済の順に処理し、PaymentCompleted の後に確認メールを送る
    // 第9回の OrderPlaced は在庫引当に使う items を持つので、注文の明細から items を作って渡す
    await eventBus.publish(new OrderPlaced(
      order.id,
      order.customerId,
      order.totalAmount,
      order.items.map((item) => ({ productId: item.productId, quantity: item.quantity })),
    ));
  } else {
    // 従来の同期的な実装
    await inventoryService.reserve(order);   // 在庫引当
    await paymentService.charge(order);      // 決済
    await notificationService.send(order);   // 確認メール（決済の後）
  }
}
```

---

## 推薦図書

本ドキュメントの内容をさらに詳しく学びたい人向けの書籍を挙げる。

| 書籍 | 著者 | 関連トピック |
|------|------|------------|
| *Building Evolutionary Architectures* 第2版（2022） | Neal Ford, Rebecca Parsons, Patrick Kua, Pramod Sadalage | 進化的アーキテクチャ |
| *Clean Agile* | Robert C. Martin | アジャイルとソフトウェア設計 |
| *Refactoring* 第2版（2018） | Martin Fowler | 段階的な設計改善 |
| *Working Effectively with Legacy Code* | Michael Feathers | 既存コードへの設計適用 |
| *Accelerate* | Nicole Forsgren et al. | 開発生産性とアーキテクチャの関係 |
