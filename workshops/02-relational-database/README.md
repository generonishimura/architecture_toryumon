# 第2回：リレーショナルデータベース

> データの整合性を守る仕組みを理解し、適切に活用する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | SQLの基本（SELECT, INSERT, UPDATE, DELETE）、第1回の内容 |
| 到達目標 | 正規化・インデックス・トランザクションを理解し、適切なテーブル設計ができる |

## この回のキーメッセージ

> 「データの整合性を守る仕組みを理解し、適切に活用する」

## 講義内容

### 1. RDBの基本概念の再確認（5分）

- テーブル、行、列、主キー、外部キー
- RDBが保証するもの：データの整合性
- なぜRDBが今でも重要なのか

### 2. 正規化（30分）

#### 正規化の目的

- データの重複を排除し、更新時異常を防ぐ
- 第1正規形〜第3正規形の段階的な理解

#### 各正規形の解説と具体例

正規化は段階的に適用します。以下の非正規形のデータを使って、各段階を追っていきます。

```
【非正規形の注文データ】
注文ID | 顧客名   | 商品名          | 商品価格 | 数量
-------+---------+-----------------+----------+-----
1      | 田中太郎 | りんご, みかん   | 100, 80  | 3, 5
2      | 佐藤花子 | みかん           | 80       | 2
```

- **第1正規形 (1NF)**: 繰り返し項目の排除、アトミックな値

```
【1NF: 1つのセルに1つの値】
注文ID | 顧客名   | 商品名   | 商品価格 | 数量
-------+---------+---------+----------+-----
1      | 田中太郎 | りんご   | 100      | 3
1      | 田中太郎 | みかん   | 80       | 5
2      | 佐藤花子 | みかん   | 80       | 2
```

- **第2正規形 (2NF)**: 部分関数従属の排除

```
【2NF: 複合キーの一部だけに依存する属性を分離】
問題: 「顧客名」は「注文ID」だけで決まる（商品名に依存しない）
→ 注文テーブルと注文明細テーブルに分離

注文テーブル: 注文ID | 顧客名
注文明細:     注文ID | 商品名 | 商品価格 | 数量
```

- **第3正規形 (3NF)**: 推移的関数従属の排除

```
【3NF: 非キー属性間の依存を排除】
問題: 「商品価格」は「商品名」で決まる（注文IDに依存しない）
→ 商品テーブルを独立させる

注文テーブル: 注文ID | 顧客ID
顧客テーブル: 顧客ID | 顧客名
商品テーブル: 商品ID | 商品名 | 商品価格
注文明細:     注文ID | 商品ID | 数量
```

> **覚え方**: 1NF は「繰り返しの排除」、2NF は「キーの一部への依存の排除」、3NF は「キー以外同士の依存の排除」。段階を踏むことで、更新時の矛盾（更新時異常・削除時異常・挿入時異常）が解消されていきます。

#### 非正規化の判断

- パフォーマンスのための意図的な非正規化
- 読み取り性能と書き込み整合性のトレードオフ
- 「まず正規化し、必要に応じて非正規化する」

> **非正規化の具体例**: ECサイトの注文履歴画面で、注文一覧に「商品名」「価格」を表示するために毎回JOINするのはコストが高い。この場合、注文明細テーブルに「注文時点の商品名」「注文時点の価格」を保存する非正規化が実務では有効です。注文後に商品名や価格が変わっても、注文当時の情報を正しく表示できるという副次的なメリットもあります。
>
> 非正規化を選ぶ判断基準:
> 1. 読み取り頻度が書き込み頻度を大幅に上回る
> 2. JOINが複雑でクエリ性能に問題がある
> 3. データの「スナップショット」としての性質を持つ（注文時点の価格など）

### 3. インデックス（25分）

#### インデックスの仕組み

- B-Treeインデックスの概念的な理解
- インデックスが効くケースと効かないケース
- 複合インデックスとカラム順序の重要性

B-Treeインデックスは「本の索引」に例えるとわかりやすいです。索引があれば目的のページにすぐたどり着けますが、すべてのキーワードに索引を付けると索引自体が厚くなりすぎます。

```
【複合インデックスの「左端一致」ルール】

インデックス: (user_id, status, created_at) の場合

✅ 効く:
  WHERE user_id = 'xxx'
  WHERE user_id = 'xxx' AND status = 'ACTIVE'
  WHERE user_id = 'xxx' AND status = 'ACTIVE' AND created_at > '2025-01-01'

❌ 効かない:
  WHERE status = 'ACTIVE'           ← 左端の user_id がない
  WHERE created_at > '2025-01-01'   ← 左端の user_id がない
  WHERE user_id = 'xxx' AND created_at > '2025-01-01'
    ← status を飛ばしているため、created_at 部分はインデックスが効かない
```

> **実務Tip**: 複合インデックスの列の順番は「カーディナリティが高い列（値の種類が多い列）を先にする」のが基本です。ただし、最も重要なのはアプリケーションの実際のクエリパターンに合わせることです。

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
- PostgreSQL / MySQLのデフォルト分離レベルの違い

| 分離レベル | ダーティリード | ノンリピータブルリード | ファントムリード | 性能 |
|-----------|:---:|:---:|:---:|:---:|
| READ UNCOMMITTED | 発生する | 発生する | 発生する | 最速 |
| READ COMMITTED | 防止 | 発生する | 発生する | 速い |
| REPEATABLE READ | 防止 | 防止 | 発生する※ | 普通 |
| SERIALIZABLE | 防止 | 防止 | 防止 | 遅い |

※ MySQLのInnoDBではREPEATABLE READでもファントムリードをほぼ防止（ギャップロック）

> **実務での使い分け**:
> - **PostgreSQL**: デフォルトは READ COMMITTED。多くのWebアプリではこれで十分
> - **MySQL (InnoDB)**: デフォルトは REPEATABLE READ。InnoDBのMVCCにより高性能
> - 金融系・在庫管理など厳密な整合性が必要な場合は、必要な箇所だけ SERIALIZABLE を検討

各問題の具体例:
- **ダーティリード**: 他のトランザクションが未コミットのデータを読んでしまう（ロールバックされるかもしれないデータ）
- **ノンリピータブルリード**: 同じSELECTを2回実行すると、間に他のUPDATEが入って結果が変わる
- **ファントムリード**: 同じ範囲SELECTを2回実行すると、間に他のINSERTが入って行数が変わる

#### デッドロック

- デッドロックが発生する条件
- 防止策と検出時の対処

```
【デッドロックの典型例】

トランザクション A              トランザクション B
─────────────────           ─────────────────
① 注文テーブルのRow 1をロック
                             ② 在庫テーブルのRow 5をロック
③ 在庫テーブルのRow 5を        ④ 注文テーブルのRow 1を
  ロックしようとする →待ち       ロックしようとする →待ち
          💀 デッドロック！ 💀
```

**防止策**:
1. **ロック順序を統一する** — 全トランザクションで「注文→在庫」の順にロックする
2. **トランザクションを短く保つ** — ロック保持時間を最小化する
3. **タイムアウトを設定する** — 一定時間待ってもロックが取得できなければ再試行
4. **楽観的ロック（Optimistic Locking）** — バージョン番号で衝突を検出する

### 5. TypeScript / Node.jsからのRDB操作（15分）

- Prisma / TypeORM / Drizzle ORM等のORMの概要
- Prisma 7の設定（生成したクライアントのimport、driver adapter、`prisma.config.ts`）
- コネクションプーリング
- マイグレーション管理の重要性

#### ORM比較: Prisma vs TypeORM

| 観点 | Prisma | TypeORM |
|------|--------|---------|
| アプローチ | スキーマファースト（`.prisma`ファイル） | コードファースト（デコレータ） |
| 型安全性 | 自動生成される型で完全な型安全 | デコレータベースで部分的 |
| マイグレーション | `prisma migrate` で自動生成 | CLI or 手動管理 |
| 生SQL | `prisma.$queryRaw` で実行可能 | `query()` で実行可能 |
| 学習コスト | 独自DSLの学習が必要 | ActiveRecord/DataMapperパターンの知識が活かせる |

> **実務Tip**: ORMは便利ですが、生成されるSQLを意識することが重要です。N+1問題（関連データを1件ずつ取得してしまう）はORMでよくある性能問題です。`include`/`join` を適切に使い、`EXPLAIN ANALYZE` でクエリプランを確認する習慣をつけましょう。

#### コネクションプーリング

```typescript
// Prisma はデフォルトでコネクションプーリングを行う
// 設定例（DATABASE_URL のパラメータで制御）
// postgresql://user:pass@host:5432/db?connection_limit=10&pool_timeout=10

// コネクションプーリングがないと:
// リクエストごとにDB接続を確立 → TCP接続のオーバーヘッド → 性能劣化
//
// コネクションプーリングがあると:
// 事前に接続を確立しておき、使い回す → 高速 → ただし接続数には上限がある
```

## コード例の概要

コード例はPrisma 7を前提にしています。Prisma 7では、次の4点が以前の版と異なります。

- `PrismaClient` は `@prisma/client` からではなく、`prisma generate` で生成したコードからimportする。生成先はschemaの `output` で指定する
- `PrismaClient` にはdriver adapterを渡す。PostgreSQLでは `@prisma/adapter-pg` の `PrismaPg` を使う
- 接続先のURLは、schemaの `datasource` ではなく `prisma.config.ts` に書く
- Prisma 7はES moduleとして配布されている。package.jsonに `"type": "module"` を設定し、tsconfigをESM用の設定にする（Prisma公式の移行ガイドを参照）

インストールするパッケージは次のとおりです。`dotenv` と `@prisma/adapter-pg` はアプリの実行時に使うので、`-D` を付けません。

```bash
npm install @prisma/client @prisma/adapter-pg dotenv
npm install -D prisma
```

この教材では、生成先を `src/generated/prisma` にします。importは、各ファイルの場所からの相対パスで書きます。たとえば `src/main.ts` からは `'./generated/prisma/client'` をimportします。

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

```typescript
// prisma.config.ts（プロジェクトのルートに置く）
// この import 'dotenv/config' は Prisma CLI（prisma migrate、prisma generate など）を実行するときに使われる。
// アプリの実行時には読み込まれないので、アプリのエントリーポイントでも import 'dotenv/config' を書く
import 'dotenv/config'; // .env の DATABASE_URL を読み込む
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  datasource: { url: env('DATABASE_URL') },
});
```

```typescript
// src/transfer.ts（Prisma を使ったトランザクション例。npx tsx src/transfer.ts で実行するエントリーポイント）
import 'dotenv/config'; // .env の DATABASE_URL を process.env に読み込む
import { PrismaClient } from './generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';

const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });
const prisma = new PrismaClient({ adapter });

// 送金処理
async function transfer(
  fromAccountId: string,
  toAccountId: string,
  amount: number
): Promise<void> {
  await prisma.$transaction(async (tx) => {
    // 残高チェックと減算を1つのUPDATE文で行う。同時実行でも残高はマイナスにならない
    const debited = await tx.account.updateMany({
      where: { id: fromAccountId, balance: { gte: amount } },
      data: { balance: { decrement: amount } },
    });
    if (debited.count === 0) throw new Error('残高不足または口座が存在しません');

    await tx.account.update({
      where: { id: toAccountId },
      data: { balance: { increment: amount } },
    });
  });
}
```

> **注意:** トランザクションが保証するのは原子性（全部成功するか、全部失敗するか）だけです。「残高を読んでから、判定して減らす」という書き方では、READ COMMITTEDのもとで同じ口座から同時に2件送金すると、両方の判定が通ります。その結果、残高がマイナスになります。対策は次のいずれかです。(1) 上の例のような条件付きUPDATE、(2) `SELECT ... FOR UPDATE` で行をロックしてから読む、(3) `isolationLevel: 'Serializable'` を指定し、失敗したら再試行する。あわせて、DBにも `CHECK (balance >= 0)` の制約を付けます。Prisma schemaにはCHECK制約を書けません。そのため、`prisma migrate dev --create-only` で作ったマイグレーションSQLに、CHECK制約を手で追加します。また、A→BとB→Aの送金が同時に走ると、2つのトランザクションが互いの行ロックを待ち、デッドロックが起きます。上の例はこの対策を入れていません。防ぐには、口座IDの小さい順に更新します。

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

1. **正規化 vs 非正規化**：どのような場面で非正規化を選択するか？その判断基準は？
2. **ORMの利点と限界**：ORMを使うことで見えなくなるRDBの知識は何か？
3. **トランザクション設計**：自分のプロジェクトで、トランザクションが適切に使われていない箇所はあるか？

## 課題

### 復習問題

1. 第1正規形〜第3正規形をそれぞれ1文で説明してください
2. 複合インデックス `(user_id, status, created_at)` がある場合、どのようなクエリパターンでインデックスが効くか、3つ例を挙げてください
3. ACID特性の各要素を、ECサイトの注文処理を例に説明してください

### コード課題

ECサイトの「注文」に関するテーブル設計を行ってください。

- ユーザー、商品、注文、注文明細の4テーブルを設計
- 第3正規形を満たすこと
- 必要なインデックスを設計し、その理由を説明
- Prisma schema形式で記述

#### 達成基準

- **最低限（30分）**: 4テーブルをPrisma schemaで書き、主キーとリレーション（外部キー）を定義する
- **目標**: 第3正規形を満たしている。よく使う検索（例：あるユーザーの注文を新しい順に表示する）に合わせて `@@index` を定義し、その理由を書いている
- **ヒント**: まず、ユーザーと注文の関係（1対多）と、注文と商品の関係（注文明細で分解する多対多）を決める

### 発展課題（任意）

自分のプロジェクトのデータベースから、以下を分析してください。
- 正規化が不十分なテーブルはあるか
- 不足しているインデックスはあるか
- `EXPLAIN ANALYZE` でスロークエリを特定し、改善案を提示

## 事前準備

### 参加者

#### 前提知識の確認

- [ ] SQLの基本構文を書ける（`SELECT`、`INSERT`、`UPDATE`、`DELETE`、`JOIN`）
- [ ] 主キー（PRIMARY KEY）と外部キー（FOREIGN KEY）の違いが分かる
- [ ] 第1回の内容（特に依存性逆転の原則、リポジトリインターフェースの考え方）を復習済み

#### 事前読み物（30分程度）

- [ ] 自分のプロジェクトで使っているデータベースのテーブルを1つ選び、そのカラム構成を確認しておく
- [ ] 以下のキーワードについて、知っている範囲で意味を調べておく
  - 正規化（第1〜第3正規形）
  - インデックス
  - トランザクション、ACID特性

#### 事前ミニ課題（任意・15分程度）

以下のテーブル設計を見て、「どんな問題が起きそうか」を考えてメモしておいてください。

```sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_name VARCHAR(100),
  customer_email VARCHAR(100),
  customer_address VARCHAR(200),
  product_name VARCHAR(100),
  product_price INT,
  quantity INT,
  order_date DATE
);
```

ヒント: 同じ顧客が複数回注文した場合、同じ商品が複数の注文に含まれる場合を考えてみてください。

### 運営側

- [ ] 第1回の課題提出状況と理解度の確認
- [ ] 正規化の段階的な例題の準備（非正規形 → 第3正規形への変換を視覚的に示すスライド）
- [ ] `EXPLAIN ANALYZE` のデモ用クエリとデータの準備
- [ ] トランザクション分離レベルのデモ環境の準備（PostgreSQL or MySQL）
- [ ] Prisma のコード例が動作する環境の確認

## 参考資料

- 『達人に学ぶDB設計 徹底指南書（第2版）』ミック
- 『SQLアンチパターン（第2版）』Bill Karwin
- [Use The Index, Luke](https://use-the-index-luke.com/ja)：インデックスの解説サイト
- 用語の確認: [用語集](../../docs/glossary.md)

## 次回との接続

第3回（データモデリング）では、第2回で学んだテーブル設計の知識を使います。ビジネス要件からER図を作り、概念モデル → 論理モデル → 物理モデルの順に具体化する手順を学びます。
