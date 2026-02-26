# 第2回：リレーショナルデータベース

> データの整合性を守る仕組みを理解し、適切に活用する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | SQLの基本（SELECT, INSERT, UPDATE, DELETE）、第1回の内容 |
| 到達目標 | 正規化・インデックス・トランザクションを理解し、適切なテーブル設計ができる |

## この回のキーメッセージ

> 「データの整合性を守る仕組みを理解し、適切に活用する」

## 講義内容

### 1. RDBの基本概念の再確認（10分）

- テーブル、行、列、主キー、外部キー
- RDBが守ってくれるもの — データ整合性の保証
- なぜRDBが今でも重要なのか

### 2. 正規化（30分）

#### 正規化の目的

- データの重複を排除し、更新時異常を防ぐ
- 第1正規形〜第3正規形の段階的な理解

#### 各正規形の解説

- **第1正規形 (1NF)**: 繰り返し項目の排除、アトミックな値
- **第2正規形 (2NF)**: 部分関数従属の排除
- **第3正規形 (3NF)**: 推移的関数従属の排除

#### 非正規化の判断

- パフォーマンスのための意図的な非正規化
- 読み取り性能と書き込み整合性のトレードオフ
- 「まず正規化し、必要に応じて非正規化する」

### 3. インデックス（25分）

#### インデックスの仕組み

- B-Treeインデックスの概念的な理解
- インデックスが効くケースと効かないケース
- 複合インデックスとカラム順序の重要性

#### インデックス設計の指針

- WHERE句、JOIN句、ORDER BY句のパターンに基づくインデックス設計
- カーディナリティとインデックスの効果
- インデックスのコスト（書き込み性能、ストレージ）

### 4. トランザクション（25分）

#### ACID特性

- **Atomicity（原子性）**: すべて成功するか、すべて失敗するか
- **Consistency（一貫性）**: 制約を常に満たす
- **Isolation（分離性）**: 同時実行トランザクションの影響を制御
- **Durability（永続性）**: コミットされたデータは失われない

#### トランザクション分離レベル

- READ UNCOMMITTED / READ COMMITTED / REPEATABLE READ / SERIALIZABLE
- 各レベルで防げる問題（ダーティリード、ノンリピータブルリード、ファントムリード）
- PostgreSQL / MySQL のデフォルト分離レベルの違い

#### デッドロック

- デッドロックが発生する条件
- 防止策と検出時の対処

### 5. TypeScript / Node.js からのRDB操作（15分）

- Prisma / TypeORM 等のORMの概要
- コネクションプーリング
- マイグレーション管理の重要性

## コード例の概要

```typescript
// Prisma を使ったトランザクション例
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// 送金処理（トランザクションで原子性を保証）
async function transfer(
  fromAccountId: string,
  toAccountId: string,
  amount: number
): Promise<void> {
  await prisma.$transaction(async (tx) => {
    const from = await tx.account.findUniqueOrThrow({
      where: { id: fromAccountId },
    });

    if (from.balance < amount) {
      throw new Error('残高不足');
    }

    await tx.account.update({
      where: { id: fromAccountId },
      data: { balance: { decrement: amount } },
    });

    await tx.account.update({
      where: { id: toAccountId },
      data: { balance: { increment: amount } },
    });
  });
}
```

```typescript
// インデックスの効果を確認するクエリ例
// 効率的: インデックスが効くケース
const orders = await prisma.order.findMany({
  where: {
    userId: 'user-123',        // user_id にインデックスあり
    status: 'COMPLETED',       // 複合インデックス (user_id, status)
  },
  orderBy: { createdAt: 'desc' },
});

// 非効率: インデックスが効かないケース
const orders = await prisma.order.findMany({
  where: {
    note: { contains: '特急' }, // LIKE '%特急%' は全文検索が必要
  },
});
```

## ディスカッションテーマ

1. **正規化 vs 非正規化** — どのような場面で非正規化を選択するか？その判断基準は？
2. **ORM の利点と限界** — ORMを使うことで見えなくなるRDBの知識は何か？
3. **トランザクション設計** — 自分のプロジェクトで、トランザクションが適切に使われていない箇所はあるか？

## 課題

### 復習問題

1. 第1正規形〜第3正規形をそれぞれ1文で説明してください
2. 複合インデックス `(user_id, status, created_at)` がある場合、どのようなクエリパターンでインデックスが効くか、3つ例を挙げてください
3. ACID特性の各要素を、ECサイトの注文処理を例に説明してください

### コード課題

ECサイトの「注文」に関するテーブル設計を行ってください。

- ユーザー、商品、注文、注文明細 の4テーブルを設計
- 第3正規形を満たすこと
- 必要なインデックスを設計し、その理由を説明
- Prisma schema形式で記述

### 発展課題（任意）

自分のプロジェクトのデータベースから、以下を分析してください。
- 正規化が不十分なテーブルはあるか
- 不足しているインデックスはあるか
- `EXPLAIN ANALYZE` でスロークエリを特定し、改善案を提示

## 参考資料

- 『達人に学ぶDB設計 徹底指南書』ミック
- 『SQLアンチパターン』Bill Karwin
- [Use The Index, Luke](https://use-the-index-luke.com/ja) — インデックスの解説サイト

## 次回との接続

第2回で学んだテーブル設計の知識を土台に、第3回（データモデリング）では、ビジネス要件からER図を起こし、概念モデル → 論理モデル → 物理モデルへと落とし込むプロセスを学びます。
