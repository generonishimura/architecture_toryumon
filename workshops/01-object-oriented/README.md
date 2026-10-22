# 第1回：オブジェクト指向

> 変更に強いコードの設計原則を学ぶ

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | TypeScriptの基本文法（クラス、インターフェース、型） |
| 到達目標 | SOLID原則を理解し、責務の分離と依存の制御を意識した設計ができる |

## この回のキーメッセージ

> 「変更に強いコードは、責務が明確で依存が制御されている」

## 講義内容

### 1. オブジェクト指向の本質（15分）

- データと振る舞いをまとめる意味
- カプセル化 — 内部状態を隠蔽し、インターフェースで操作する
- ポリモーフィズム — 同じインターフェースで異なる振る舞いを実現する
- 「継承よりもコンポジション」— 継承は親クラスと強く結合し、組み合わせの数だけクラスが増える。部品を持たせて組み合わせるほうが変更しやすい

### 2. SOLID原則（50分）

#### S — 単一責任の原則 (Single Responsibility Principle)

- クラスが変更される理由は1つだけであるべき
- Martin は後に「1つのモジュールは、1つのアクター（変更を求めてくる人や部署）に対して責務を負う」と言い直している（『Clean Architecture』第7章）
- 「責務」の粒度をどう判断するか
- TypeScript例：レポート生成と出力フォーマットの分離

#### O — 開放閉鎖の原則 (Open/Closed Principle)

- 拡張に対して開いている、修正に対して閉じている
- Strategy パターンによる拡張ポイントの設計
- TypeScript例：割引計算ロジックの拡張

#### L — リスコフの置換原則 (Liskov Substitution Principle)

- サブタイプは基底型の契約を守らなければならない
- 振る舞いの互換性と事前条件・事後条件
- TypeScript例：不適切な継承と正しい設計の比較

#### I — インターフェース分離の原則 (Interface Segregation Principle)

- クライアントが使わないメソッドへの依存を強制しない
- 大きなインターフェースを小さく分割する
- TypeScript例：`interface Readable` と `interface Writable` の分離

#### D — 依存性逆転の原則 (Dependency Inversion Principle)

- 上位モジュールは下位モジュールに依存すべきでない
- 抽象（インターフェース）に依存する
- インターフェースは使う側（上位モジュール）が定義して所有する
- TypeScript例：リポジトリインターフェースによるDB依存の逆転

### 3. 実務での適用（15分）

- 過度な抽象化の罠 — YAGNI（You Ain't Gonna Need It）との折り合い
- テスタビリティと設計品質の関係
- 既存コードのリファクタリングアプローチ

## コード例の概要

講義中に使用する主なTypeScriptコード例。

```typescript
// Bad: 複数の責務が混在
class OrderService {
  calculateTotal(order: Order): number { /* ... */ }
  formatReceipt(order: Order): string { /* ... */ }
  sendEmail(to: string, body: string): void { /* ... */ }
}

// Good: 責務を分離
// 変更を求める人：経理・販売ルールの担当
class OrderCalculator {
  calculateTotal(order: Order): number { /* ... */ }
}

// 変更を求める人：画面・帳票のデザイン担当
class ReceiptFormatter {
  format(order: Order): string { /* ... */ }
}

// 変更を求める人：インフラ担当（メール基盤の変更）。infrastructure/ ディレクトリに置くクラスで、第6回で配置を扱う
class EmailSender {
  send(to: string, body: string): void { /* ... */ }
}
```

```typescript
// 開放閉鎖の原則
// Bad: 割引の種類が増えるたびに、この関数を書き換える必要がある
function calculateDiscountedPrice(price: number, type: 'none' | 'member' | 'campaign'): number {
  if (type === 'member') return price * 0.9;
  if (type === 'campaign') return price - 500;
  return price;
}

// Good: 割引の種類ごとにクラスを作る。新しい割引は、クラスを1つ追加するだけで済む
// （第4回で学ぶ Strategy パターンの予告）
interface DiscountPolicy {
  apply(price: number): number;
}

class MemberDiscount implements DiscountPolicy {
  apply(price: number): number { return price * 0.9; }
}

class CampaignDiscount implements DiscountPolicy {
  apply(price: number): number { return price - 500; }
}
```

```typescript
// リスコフの置換原則
// Bad: 子クラスが、親クラスのメソッドを「呼ぶと例外になるもの」に変えている
class List<T> {
  protected items: T[] = [];
  get(index: number): T | undefined { return this.items[index]; }
  add(item: T): void { this.items.push(item); }
}

class ReadOnlyList<T> extends List<T> {
  override add(_item: T): void {
    throw new Error('このリストには追加できません');
  }
}

// この関数は「List なら add() が使える」と期待している。
// ReadOnlyList を渡すと例外になり、呼び出し側の期待が壊れる
function addDefault(list: List<string>): void {
  list.add('default');
}
```

```typescript
// リスコフの置換原則
// Good: 「読める」と「書ける」を別のインターフェースに分ける
interface ReadableList<T> {
  get(index: number): T | undefined;
}

interface WritableList<T> extends ReadableList<T> {
  add(item: T): void;
}

// add() を使う関数は WritableList を受け取る。
// 読み取り専用のリストは ReadableList だけを実装するので、この関数には渡せない（コンパイル時にエラーになる）
function addDefault(list: WritableList<string>): void {
  list.add('default');
}
```

```typescript
// インターフェース分離の原則
// Bad: 読み取りしかしないクラスにも、write() の実装を強制している
interface KeyValueStore {
  read(key: string): Promise<string | null>;
  write(key: string, value: string): Promise<void>;
}

// Good: 小さなインターフェースに分ける
interface Readable {
  read(key: string): Promise<string | null>;
}

interface Writable {
  write(key: string, value: string): Promise<void>;
}

// 読み取り専用のキャッシュは Readable だけを実装する
class ReadOnlyCache implements Readable {
  constructor(private readonly entries: Map<string, string>) {}

  async read(key: string): Promise<string | null> {
    return this.entries.get(key) ?? null;
  }
}
```

```typescript
// 依存性逆転の原則
// Before: ビジネスロジックが具体的なDB実装に依存している
// 依存の矢印：OrderService → PrismaOrderRepository
class OrderService {
  private readonly orderRepo = new PrismaOrderRepository();

  async completeOrder(orderId: string): Promise<void> { /* ... */ }
}
```

```typescript
// 依存性逆転の原則
// After: OrderRepository インターフェースは、ビジネスロジック側（ドメイン層）が定義して持つ。
// PrismaOrderRepository がそれを実装するので、依存の矢印は「実装クラス（infrastructure/）→ ドメイン」に逆転する
interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}

// ビジネスロジックは抽象に依存
class OrderService {
  constructor(private readonly orderRepo: OrderRepository) {}

  async completeOrder(orderId: string): Promise<void> {
    const order = await this.orderRepo.findById(orderId);
    if (!order) throw new Error('Order not found');
    order.complete();
    await this.orderRepo.save(order);
  }
}

// インフラ側の実装クラスが、ドメイン層のインターフェースを実装する
class PrismaOrderRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> { /* ... */ }
  async save(order: Order): Promise<void> { /* ... */ }
}
```

```typescript
// 継承よりもコンポジション
// 継承：「通知手段 × 優先度」の組み合わせごとにサブクラスが増える
class EmailNotifier { /* ... */ }
class SlackNotifier { /* ... */ }
class UrgentEmailNotifier extends EmailNotifier { /* ... */ }
class UrgentSlackNotifier extends SlackNotifier { /* ... */ }
// 通知手段を1つ追加すると、通常版と緊急版の2クラスが増える

// コンポジション：部品（通知手段と優先度）を外から渡して組み合わせる。クラスは増えない
interface Channel {
  send(to: string, body: string): Promise<void>;
}

type Priority = 'normal' | 'urgent';

class NotificationService {
  constructor(
    private readonly channel: Channel,
    private readonly priority: Priority,
  ) {}
}
```

## ディスカッションテーマ

以下のテーマから1つ選んでグループで議論する。

1. **SOLID原則のトレードオフ** — SOLID原則を厳格に適用しすぎると、どんな問題が起きるか？
2. **実務のコード改善** — 自分のプロジェクトで「単一責任の原則に違反している」と思うコードはあるか？
3. **テスタビリティ** — テストしづらいコードの特徴は何か？オブジェクト指向の原則とどう関係するか？

## 課題

### 復習問題

1. SOLID原則の各原則を、自分の言葉で1文ずつ説明してください
2. 依存性逆転の原則が守られていないコード例を1つ挙げ、改善案を示してください
3. 「継承よりもコンポジション」が推奨される理由を説明してください

### コード課題

以下の要件を持つ「通知システム」をTypeScriptで設計してください。

- メール、Slack、SMS の3つの通知手段に対応する
- 通知の送信先と本文を受け取って送信する
- 将来的に新しい通知手段（LINE等）を追加可能にする
- SOLID原則を意識した設計にする

#### 達成基準

- **最低限（30分）**: Notifier インターフェースと、1つの通知手段の実装
- **目標**: 3手段を実装し、新しい手段を追加するときに既存クラスを変更しない（OCP）。送信処理が具体クラスではなく Notifier に依存している（DIP）
- **ヒント**: まず OCP と DIP を意識する

### 発展課題（任意）

自分のプロジェクトのコードから、SOLID原則に違反していると思われる箇所を1つ見つけ、リファクタリング案を書いてください。Before/Afterのコードと、どの原則を適用したかを説明してください。

## 参考資料

- 『オブジェクト指向設計実践ガイド』Sandi Metz
- 『Clean Code』Robert C. Martin
- [SOLID Principles in TypeScript](https://blog.logrocket.com/solid-principles-typescript/) （Web記事）
- 用語の確認: [用語集](../../docs/glossary.md)

## 次回との接続

第1回で学んだ**依存性逆転の原則**と**インターフェースによる抽象化**は、第2回以降のすべてのテーマで繰り返し登場します。次の第2回（RDB）では、ここで抽象化した『データの保存先』そのもの、つまりデータベースの設計原則を学びます。データアクセスを抽象化する Repository パターンは、第4回（デザインパターン）で詳しく扱います。
