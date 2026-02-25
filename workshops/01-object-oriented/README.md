# 第1回：オブジェクト指向

> 変更に強いコードの設計原則を学ぶ

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | TypeScriptの基本文法（クラス、インターフェース、型） |
| 到達目標 | SOLID原則を理解し、責務の分離と依存の制御を意識した設計ができる |

## この回のキーメッセージ

> 「変更に強いコードは、責務が明確で依存が制御されている」

## 講義内容

### 1. オブジェクト指向の本質（15分）

- データと振る舞いをまとめる意味
- カプセル化 — 内部状態を隠蔽し、インターフェースで操作する
- ポリモーフィズム — 同じインターフェースで異なる振る舞いを実現する
- 「継承よりもコンポジション」という現代的な考え方

### 2. SOLID原則（50分）

#### S — 単一責任の原則 (Single Responsibility Principle)

- クラスが変更される理由は1つだけであるべき
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
class OrderCalculator {
  calculateTotal(order: Order): number { /* ... */ }
}

class ReceiptFormatter {
  format(order: Order): string { /* ... */ }
}

class EmailSender {
  send(to: string, body: string): void { /* ... */ }
}
```

```typescript
// 依存性逆転の原則
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

### 発展課題（任意）

自分のプロジェクトのコードから、SOLID原則に違反していると思われる箇所を1つ見つけ、リファクタリング案を書いてください。Before/Afterのコードと、どの原則を適用したかを説明してください。

## 参考資料

- 『オブジェクト指向設計実践ガイド』Sandi Metz
- 『Clean Code』Robert C. Martin
- [SOLID Principles in TypeScript](https://blog.logrocket.com/solid-principles-typescript/) （Web記事）

## 次回との接続

第1回で学んだ**依存性逆転の原則**と**インターフェースによる抽象化**は、第2回以降のすべてのテーマで繰り返し登場します。特に第2回（RDB）では、データアクセスの抽象化としてリポジトリパターンの具体例を見ていきます。
