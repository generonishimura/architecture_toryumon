# 開発手法とアーキテクチャの関係

> 本ドキュメントは、アーキテクチャ登竜門の補足資料として、ソフトウェア開発手法とアーキテクチャ設計の関係性をまとめたものです。ワークショップ本編では扱いきれないが、アーキテクチャを実務で活かすために不可欠な知識を提供します。

---

## なぜ開発手法を知る必要があるか

アーキテクチャを学ぶと「最初から完璧な設計をしなければ」と考えがちである。しかし、実際のソフトウェア開発では以下の現実がある。

- 要件はプロジェクト中に変わる
- 最初にすべてを見通すことは不可能
- 過剰な事前設計は、使われない抽象化や不要な柔軟性を生む

開発手法を理解することで、「いつ・どの程度の設計をするか」という **設計のタイミングと粒度の判断** ができるようになる。

---

## 1. 進化的アーキテクチャ（Evolutionary Architecture）

### 核心的な考え方

> アーキテクチャは最初に「決める」ものではなく、時間をかけて「育てる」ものである。

Neal Ford & Rebecca Parsons の提唱する進化的アーキテクチャでは、以下を重視する。

- **段階的な変更（Incremental Change）**: 小さな変更を頻繁に行い、フィードバックを得る
- **適応度関数（Fitness Functions）**: アーキテクチャの品質を自動的に検証する仕組み
- **適切な結合（Appropriate Coupling）**: 変更の影響範囲を制御するための意図的な境界設計

### ワークショップとの関連

| ワークショップ回 | 関連する進化的アーキテクチャの考え方 |
|----------------|----------------------------------|
| 第4回 デザインパターン | パターンは「最初から入れる」ではなく「必要になったときに適用する」 |
| 第5回 DDD | 境界づけられたコンテキストは、サービス分割の将来の候補を示す |
| 第6回 クリーンアーキテクチャ | 依存性逆転により、インフラの差し替えが容易になる |
| 第7回 マイクロサービス | モノリスから始めて、成長に応じてサービスを切り出す |

### 実践のポイント

```
❌ ウォーターフォール的思考:
   「すべてのコンポーネントを事前に設計し、設計書通りに実装する」

✅ 進化的思考:
   「現在の要件に十分な設計から始め、学びに応じて設計を進化させる」
```

---

## 2. YAGNI原則（You Ain't Gonna Need It）

### 核心的な考え方

> 実際に必要になるまで、機能や抽象化を追加してはならない。

YAGNI原則はエクストリームプログラミング（XP）から生まれた原則で、以下を戒める。

- 「将来使うかもしれない」抽象化の事前実装
- 現時点で必要のないデザインパターンの適用
- 使われることのない設定の柔軟性

### アーキテクチャ設計への適用

**過剰設計の例:**

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

以下の問いかけで、YAGNIに反していないか確認できる。

1. **今のユーザーストーリーにこの抽象化は必要か？**
2. **この柔軟性を使う具体的なシナリオが、直近のロードマップにあるか？**
3. **この構造を追加することで、現在のコードの理解しやすさは下がらないか？**

---

## 3. ADR（Architecture Decision Records）

### 核心的な考え方

> 設計判断の「なぜ」を記録することで、未来のチームメンバーが同じ議論を繰り返さなくて済む。

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
- **Auth0**: 機能は豊富だが、無料枠の制限が厳しい
- **自前実装（Passport.js）**: 柔軟だが、セキュリティの担保が困難

## 結果
Firebase Authenticationで認証基盤を構築し、3ヶ月後に
パフォーマンスとコストを再評価する。
```

### ワークショップとの関連

本ワークショップで学ぶ設計判断は、すべてADRの対象となり得る。

- 「なぜモノリスではなくマイクロサービスを選んだか」
- 「なぜCQRSを導入したか」
- 「なぜこの集約の境界にしたか」

ADRを書く習慣を持つことで、設計判断の「なぜ」を言語化する力が鍛えられる。これは本ワークショップの目標「設計判断の引き出しを増やす」に直結する。

---

## 4. 技術的負債の管理

### 核心的な考え方

> 技術的負債は「悪」ではなく、意図的に取るか無意識に生まれるかが問題である。

### 技術的負債の4象限（Martin Fowler）

|  | 意図的 | 無意識 |
|--|--------|--------|
| **慎重** | 「今はこの設計で出荷し、次のスプリントでリファクタリングする」 | 「レイヤー分離がもっと良い方法があったと後から気づいた」 |
| **無謀** | 「設計する時間がないからとりあえず動くコードを書く」 | 「そもそもレイヤー分離が何かを知らなかった」 |

### アーキテクチャ学習との関係

本ワークショップを受講することで、以下の変化が期待できる。

- **無意識の負債が減る**: 設計の選択肢を知ることで、知らずに生まれる負債が減少
- **意図的な判断ができる**: 「今は第3正規形で十分。パフォーマンス問題が出たら非正規化を検討する」のように、根拠を持って判断できる
- **負債の返済計画が立てられる**: リファクタリングの方向性（どのパターンに向かうか）が見える

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

クリーンアーキテクチャの最大の実務的メリットのひとつが **テスタビリティ** である。

```typescript
// ❌ インフラ層に直接依存したビジネスロジック → テストが困難
class OrderService {
  async createOrder(items: Item[]): Promise<Order> {
    const db = new PostgresClient();  // テスト時にDBが必要
    const order = new Order(items);
    await db.query('INSERT INTO orders ...', [order]);

    const mailer = new SendGridClient();  // テスト時にメール送信が発生
    await mailer.send(order.customerEmail, 'Order confirmed');

    return order;
  }
}

// ✅ 依存性逆転でインフラを分離 → 単体テストが容易
interface OrderRepository {
  save(order: Order): Promise<void>;
}
interface NotificationService {
  notifyOrderCreated(order: Order): Promise<void>;
}

class CreateOrderUseCase {
  constructor(
    private orderRepo: OrderRepository,
    private notifier: NotificationService,
  ) {}

  async execute(items: Item[]): Promise<Order> {
    const order = new Order(items);
    await this.orderRepo.save(order);
    await this.notifier.notifyOrderCreated(order);
    return order;
  }
}

// テスト: DBもメールも不要
test('注文を作成できる', async () => {
  const fakeRepo: OrderRepository = { save: async () => {} };
  const fakeNotifier: NotificationService = { notifyOrderCreated: async () => {} };
  const useCase = new CreateOrderUseCase(fakeRepo, fakeNotifier);

  const order = await useCase.execute([{ name: 'Book', price: 1500 }]);

  expect(order.items).toHaveLength(1);
});
```

### TDDの基本サイクル

```
Red    → テストを書く（まだ実装がないので失敗する）
Green  → テストが通る最小限の実装を書く
Refactor → コードを整理する（テストが通り続けることを確認）
```

TDDはアーキテクチャ設計を「テストが書きやすい構造に自然と導く」効果がある。テストが書きにくいと感じたら、それは設計改善のシグナルである。

---

## 6. アジャイル開発とアーキテクチャの関係

### よくある誤解

> 「アジャイルではアーキテクチャ設計をしない」

これは誤りである。アジャイルは **事前に完璧な設計をしない** のであって、**設計をしない** のではない。

### 正しい理解

| ウォーターフォール的アプローチ | アジャイル的アプローチ |
|-----------------------------|---------------------|
| 最初に完全な設計書を作成 | 最初は「十分な」設計で始める |
| 設計変更は「手戻り」として忌避 | 設計変更は「学び」として歓迎 |
| アーキテクトが設計し、開発者が実装 | チーム全体でアーキテクチャを育てる |
| 設計は最初のフェーズで完了 | 設計は開発を通して継続的に進化 |

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

既存システムのアーキテクチャを改善する際に使える実践的なパターン。

### ストラングラーフィグパターン（Strangler Fig Pattern）

モノリスからマイクロサービスへの移行で最もよく使われるパターン。

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

新しいアーキテクチャへの移行を段階的に行うための仕組み。

```typescript
// フィーチャーフラグで新旧の実装を切り替え
async function processOrder(order: Order): Promise<void> {
  if (featureFlags.isEnabled('use-event-driven-order')) {
    // 新しいイベント駆動の実装
    await eventBus.publish(new OrderCreatedEvent(order));
  } else {
    // 従来の同期的な実装
    await orderProcessor.process(order);
    await notificationService.send(order);
    await inventoryService.reserve(order);
  }
}
```

---

## 推薦図書

本ドキュメントの内容をさらに深く学びたい方向け。

| 書籍 | 著者 | 関連トピック |
|------|------|------------|
| *Building Evolutionary Architectures* | Neal Ford, Rebecca Parsons | 進化的アーキテクチャ |
| *Clean Agile* | Robert C. Martin | アジャイルとソフトウェア設計 |
| *Refactoring* | Martin Fowler | 段階的な設計改善 |
| *Working Effectively with Legacy Code* | Michael Feathers | 既存コードへの設計適用 |
| *Accelerate* | Nicole Forsgren et al. | 開発生産性とアーキテクチャの関係 |
