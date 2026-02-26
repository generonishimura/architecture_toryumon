# 第3回：データモデリング

> ビジネスの構造を正しく反映したデータモデルを設計する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
| 前提知識 | 第2回（RDB）の内容、正規化の基礎知識 |
| 到達目標 | ER図の読み書きができ、概念/論理/物理モデルの段階的な設計プロセスを理解している |

## この回のキーメッセージ

> 「良いデータモデルは、ビジネスの構造を正しく反映している」

## 講義内容

### 1. データモデリングとは（10分）

- 現実世界の構造をデータとして表現する技法
- なぜ「いきなりテーブル設計」ではダメなのか
- データモデリングの3段階：概念 → 論理 → 物理

### 2. 概念データモデル（20分）

#### 概念モデルの目的

- ビジネスの主要なエンティティとその関係を洗い出す
- 技術的な制約を含まない、ビジネス視点のモデル
- ステークホルダーとの認識合わせに使う

#### ER図の基本要素

- エンティティ（実体）
- アトリビュート（属性）
- リレーションシップ（関連）
- カーディナリティ（多重度）— 1:1、1:N、M:N

#### 概念モデリングのコツ

- ビジネスの言葉でエンティティを命名する
- 「〜は〜を持つ」「〜は〜に所属する」で関係を表現
- 最初から完璧を目指さず、繰り返し洗練する

### 3. 論理データモデル（20分）

#### 概念モデルから論理モデルへ

- M:N関係の中間テーブルへの分解
- 属性の型の決定（まだ物理的な型ではなく論理的な型）
- 主キーの選定（ナチュラルキー vs サロゲートキー）

#### 論理モデルの設計ポイント

- 正規化の適用（第2回の知識を活用）
- NULL許容の判断基準
- 導出属性（計算で求められる値）をどう扱うか

### 4. 物理データモデル（20分）

#### 論理モデルから物理モデルへ

- データベース固有の型への変換
- インデックス設計（第2回の知識を活用）
- パーティショニングの検討
- 制約の定義（NOT NULL、UNIQUE、CHECK、外部キー）

#### 物理設計の実践的なポイント

- タイムスタンプ列の設計（created_at, updated_at）
- ソフトデリートの是非
- 履歴テーブルの設計パターン
- マイグレーション戦略

### 5. 実践：モデリングワークスルー（30分）

講義中に簡単なビジネスドメイン（例：図書管理システム）を題材に、3段階のモデリングを実演する。

- 要件ヒアリングのシミュレーション
- 概念モデルの作成
- 論理モデルへの変換
- 物理モデル（Prisma schema）の作成

## コード例の概要

```typescript
// 概念モデルをPrisma schemaに落とし込む例

// 図書管理システム
model Author {
  id        String   @id @default(uuid())
  name      String
  books     Book[]
  createdAt DateTime @default(now()) @map("created_at")

  @@map("authors")
}

model Book {
  id          String       @id @default(uuid())
  title       String
  isbn        String       @unique
  publishedAt DateTime     @map("published_at")
  authorId    String       @map("author_id")
  author      Author       @relation(fields: [authorId], references: [id])
  loans       BookLoan[]
  createdAt   DateTime     @default(now()) @map("created_at")

  @@index([authorId])
  @@index([isbn])
  @@map("books")
}

model Member {
  id        String     @id @default(uuid())
  name      String
  email     String     @unique
  loans     BookLoan[]
  createdAt DateTime   @default(now()) @map("created_at")

  @@map("members")
}

// M:N を中間テーブルで表現（貸出管理）
model BookLoan {
  id         String    @id @default(uuid())
  bookId     String    @map("book_id")
  memberId   String    @map("member_id")
  loanedAt   DateTime  @default(now()) @map("loaned_at")
  returnedAt DateTime? @map("returned_at")
  book       Book      @relation(fields: [bookId], references: [id])
  member     Member    @relation(fields: [memberId], references: [id])

  @@index([bookId])
  @@index([memberId])
  @@index([returnedAt])
  @@map("book_loans")
}
```

## ディスカッションテーマ

1. **ナチュラルキー vs サロゲートキー** — UUID / auto increment / ビジネスキー、それぞれのメリット・デメリットは？
2. **ソフトデリート** — `deleted_at` カラムを使うソフトデリートは良い設計か？どんな場面で適切か？
3. **モデルの進化** — ビジネス要件の変化に伴うデータモデルの変更をどう管理するか？

## 課題

### 復習問題

1. 概念モデル・論理モデル・物理モデルの違いを、それぞれの目的と対象読者の観点から説明してください
2. M:N関係を中間テーブルに分解する理由を説明してください
3. ソフトデリートを採用する場合に注意すべき点を3つ挙げてください

### コード課題

「タスク管理ツール」のデータモデルを設計してください。

要件：
- ユーザーがプロジェクトを作成し、プロジェクト内にタスクを作成できる
- タスクには担当者を割り当てられる（複数人も可）
- タスクにはコメントを投稿できる
- タスクにはラベル（タグ）を複数付けられる

成果物：
- 概念モデル（ER図 または テキストでの関連記述）
- Prisma schema形式の物理モデル
- インデックス設計とその根拠

### 発展課題（任意）

自分のプロジェクトの既存テーブル構造を概念モデルに逆変換し、現在のモデルの改善点を洗い出してください。

## 参考資料

- 『楽々ERDレッスン』羽生章洋
- 『達人に学ぶDB設計 徹底指南書』ミック
- [dbdiagram.io](https://dbdiagram.io/) — ER図作成ツール

## 次回との接続

データモデリングで学んだ「ビジネスの構造をモデルとして表現する」という考え方は、第4回（デザインパターン）でのオブジェクト設計、そして第5回（DDD）でのドメインモデリングの土台になります。
