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

正規化は1段階ずつ適用します。次の非正規形のデータを使って、各段階でテーブルがどう変わるかを示します。

```
【非正規形の注文データ】
注文ID | 顧客ID | 顧客名   | 商品名          | 商品価格 | 数量
-------+--------+---------+-----------------+----------+-----
1      | C1     | 田中太郎 | りんご, みかん   | 100, 80  | 3, 5
2      | C2     | 佐藤花子 | みかん           | 80       | 2
```

- **第1正規形 (1NF)**: 繰り返し項目の排除、アトミックな値

```
【1NF: 1つのセルに1つの値】
注文ID | 顧客ID | 顧客名   | 商品名   | 商品価格 | 数量
-------+--------+---------+---------+----------+-----
1      | C1     | 田中太郎 | りんご   | 100      | 3
1      | C1     | 田中太郎 | みかん   | 80       | 5
2      | C2     | 佐藤花子 | みかん   | 80       | 2
```

この表の主キーは（注文ID, 商品名）です。

- **第2正規形 (2NF)**: 部分関数従属の排除

```
【2NF: 複合キーの一部だけに依存する属性を分離】
問題1: 「顧客ID」「顧客名」は「注文ID」だけで決まる（商品名に依存しない）
問題2: 「商品価格」は「商品名」だけで決まる（注文IDに依存しない）
→ 注文テーブル、商品テーブル、注文明細テーブルに分離

注文テーブル: 注文ID | 顧客ID | 顧客名
商品テーブル: 商品名 | 商品価格
注文明細:     注文ID | 商品名 | 数量
```

- **第3正規形 (3NF)**: 推移的関数従属の排除

```
【3NF: 主キー以外の列どうしの依存を分離】
問題: 注文テーブルでは「注文ID → 顧客ID → 顧客名」の順に決まる（推移的関数従属）
→ 顧客テーブルを独立させる

注文テーブル: 注文ID | 顧客ID
顧客テーブル: 顧客ID | 顧客名
商品テーブル: 商品名 | 商品価格
注文明細:     注文ID | 商品名 | 数量
```

実務では、商品名が変わることに備えて、商品テーブルに商品ID（サロゲートキー）を付け、注文明細は商品IDで参照します。これは正規化の手順とは別の、主キーの選び方の話です（第3回で扱います）。

> **覚え方**: 1NF は「繰り返しの排除」、2NF は「キーの一部への依存の排除」、3NF は「キー以外同士の依存の排除」。1段階ずつテーブルを分けると、データの矛盾（更新時異常・削除時異常・挿入時異常）が起きにくくなります。

#### 非正規化の判断

- パフォーマンスのための意図的な非正規化
- 読み取り性能と書き込み整合性のトレードオフ
- 「まず正規化し、必要に応じて非正規化する」

> **非正規化の具体例**: 注文一覧に顧客名を表示するたびに `orders` と `customers` をJOINすると時間がかかる場合に、`orders` に `customer_name` をコピーして持たせる方法です。顧客名が変わったら両方を更新する必要があるので、更新時異常のリスクを引き受けることになります。
>
> なお、注文明細に「注文時点の単価」を保存するのは非正規化ではありません。商品の現在価格とは別の事実（その注文で合意した価格）なので、正規化された設計でも注文明細の列になります（第5回の「注文された品目」の属性と同じ）。
>
> 非正規化を選ぶ判断基準は次の3つです。
> 1. 読み取り頻度が書き込み頻度を大幅に上回る
> 2. JOINが複雑で、`EXPLAIN ANALYZE` で確認するとクエリ性能に問題がある
> 3. コピーした値を更新し忘れないしくみ（同じトランザクションで更新する、トリガーなど）を用意できる

### 3. インデックス（25分）

#### インデックスの仕組み

- B-Treeインデックスの概念的な理解
- インデックスが効くケースと効かないケース
- 複合インデックスとカラム順序の重要性

B-Treeインデックスは、列の値を順番に並べて木構造で保存したデータです。インデックスがあると、DBはテーブルを先頭から全部読まずに、目的の行の位置を探せます。ただし、インデックスを増やすほど、書き込みのたびに更新するインデックスが増え、使う容量も増えます。

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

△ 一部だけ効く:
  WHERE user_id = 'xxx' AND created_at > '2025-01-01'
    ← user_id でインデックスを使って絞り込める。created_at はインデックスの範囲検索には使えず、絞り込んだ行への追加の条件として調べられる
```

> **実務Tip**: 複合インデックスの列の順番は、実際のクエリに合わせて決めます。等価条件（=）で絞る列を先に置き、範囲条件（>、BETWEEN）や ORDER BY に使う列を後に置きます。上の例で `created_at` が最後にあるのはこのためです。「カーディナリティが高い列を先に」はよく聞く説明ですが、正しくありません。等価条件の列どうしなら、どの順番でも絞り込みの効率はほぼ同じです。その列だけで検索するクエリがある列を先頭に置くと、1つのインデックスをより多くのクエリで使えます。

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

※ この表は、SQL標準が「起きてもよい」とする現象です。実際の動作はDBMSごとに違います。PostgreSQLでは、READ UNCOMMITTEDを指定してもREAD COMMITTEDとして動くので、ダーティリードは起きません。PostgreSQLのREPEATABLE READはスナップショットを読むので、ファントムリードも起きません（ただし書き込みスキューは起きえます。防ぐにはSERIALIZABLEを使います）。MySQL InnoDBのREPEATABLE READでも、通常のSELECTではファントムは見えません。`SELECT ... FOR UPDATE` などのロックを伴う読み取りでは、ギャップロックによって、範囲へのINSERTを防ぎます。

> **実務での使い分け**:
> - **PostgreSQL**: デフォルトは READ COMMITTED。多くのWebアプリではこれで十分
> - **MySQL (InnoDB)**: デフォルトは REPEATABLE READ。PostgreSQLと同じくMVCCを使い、読み取りと書き込みが互いを待たない
> - 金融系・在庫管理など厳密な整合性が必要な場合は、必要な箇所だけ SERIALIZABLE を検討

各問題の具体例は次のとおりです。

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

デッドロックの防止策は次の2つです。

1. **ロック順序を統一する**：全トランザクションで「注文→在庫」の順にロックする
2. **トランザクションを短く保つ**：ロックを持っている時間を短くする

それでもデッドロックが起きたときは、次のように対処します。

- DBはデッドロックを自動で検出し、片方のトランザクションをエラーにしてロールバックします（PostgreSQLはSQLSTATE 40P01、MySQLはエラー番号1213）
- アプリはこのエラーを受け取ったら、回数に上限を付けて、トランザクションを最初から再試行します

ロック待ちのタイムアウトは、ロックを待ち続けることを防ぐ設定で、デッドロックの防止策ではありません。楽観的ロック（バージョン番号で衝突を検出する方式）は、画面操作の間などにロックを持たないための方式です。DBの行ロックどうしのデッドロックを防ぐものではありません。

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

> **実務Tip**: ORMを使うときも、ORMが生成するSQLを確認しましょう。N+1問題（関連データを1件ずつ取得してしまう問題）は、ORMでよく起きる性能問題です。`include`/`join` を適切に使い、`EXPLAIN ANALYZE` でクエリプランを確認する習慣をつけましょう。

#### コネクションプーリング

Prisma 7でdriver adapterを使う場合、接続プールはドライバー（pg）の `pg.Pool` が管理します。プールの設定は `PrismaPg` に渡します。Prisma 6までの `DATABASE_URL` のパラメータ（`connection_limit`、`pool_timeout`）は使われません。

```typescript
// src/main.ts
import 'dotenv/config';
import { PrismaClient } from './generated/prisma/client';
import { PrismaPg } from '@prisma/adapter-pg';

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL,
  max: 10, // プールが持つ接続の上限（pg の既定値は 10）
  connectionTimeoutMillis: 5_000, // 接続の確立と、空き接続を待つ時間の上限（pg の既定値は 0 = 無制限）
});
const prisma = new PrismaClient({ adapter });
```

プールがないと、リクエストのたびにDBへの接続を確立するので、TCP接続と認証の時間が毎回かかります。プールを使うと、確立済みの接続を使い回せるので、この時間はかかりません。同時に使える接続の数は `max` で決まり、すべて使用中のときは、接続が空くまで待ちます。

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
