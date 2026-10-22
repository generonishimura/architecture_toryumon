# 第3回：データモデリング

> ビジネスの構造を正しく反映したデータモデルを設計する

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
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
- カーディナリティ（多重度）：1:1、1:N、M:N

#### 概念モデリングのコツ

- ビジネスの言葉でエンティティを命名する
- 「〜は〜を持つ」「〜は〜に所属する」で関係を表現
- 最初から完璧を目指さず、繰り返し見直して改善する

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
- 「同じ本が2冊あったら？」「共著の本は？」を考え、書誌（Book）と蔵書（BookCopy）の区別、著者と本のM:Nを見つける
- 論理モデルへの変換
- 物理モデル（Prisma schema）の作成

## コード例の概要

```prisma
// 概念モデルをPrisma schemaに落とし込む例

// 図書管理システム
model Author {
  id        String       @id @default(uuid())
  name      String
  books     BookAuthor[]
  createdAt DateTime     @default(now()) @map("created_at")

  @@map("authors")
}

// 書誌（タイトル）。ISBN が同じ本は1件だけ登録する
model Book {
  id          String       @id @default(uuid())
  title       String
  isbn        String       @unique
  publishedAt DateTime     @map("published_at")
  authors     BookAuthor[]
  copies      BookCopy[]
  createdAt   DateTime     @default(now()) @map("created_at")

  @@index([isbn])
  @@map("books")
}

// M:N を中間テーブルで表現（共著：1冊の本に複数の著者、1人の著者に複数の本）
model BookAuthor {
  bookId   String @map("book_id")
  authorId String @map("author_id")
  book     Book   @relation(fields: [bookId], references: [id])
  author   Author @relation(fields: [authorId], references: [id])

  @@id([bookId, authorId])
  @@index([authorId])
  @@map("book_authors")
}

// 蔵書（図書館が持っている物理的な1冊）。同じ本を2冊持っていれば2件になる
model BookCopy {
  id              String     @id @default(uuid())
  bookId          String     @map("book_id")
  accessionNumber String     @unique @map("accession_number") // 所蔵番号
  status          String     @default("AVAILABLE") // 状態（AVAILABLE / ON_LOAN / LOST など）
  book            Book       @relation(fields: [bookId], references: [id])
  loans           BookLoan[]
  createdAt       DateTime   @default(now()) @map("created_at")

  @@index([bookId])
  @@map("book_copies")
}

model Member {
  id        String     @id @default(uuid())
  name      String
  email     String     @unique
  loans     BookLoan[]
  createdAt DateTime   @default(now()) @map("created_at")

  @@map("members")
}

// 貸出は単純な中間テーブルではなく「出来事のエンティティ」
// 独自の id と貸出日・返却日を持つ。同じ会員が同じ蔵書を何度借りても、そのたびに1件ずつ記録される
model BookLoan {
  id         String    @id @default(uuid())
  copyId     String    @map("copy_id")
  memberId   String    @map("member_id")
  loanedAt   DateTime  @default(now()) @map("loaned_at")
  returnedAt DateTime? @map("returned_at")
  copy       BookCopy  @relation(fields: [copyId], references: [id])
  member     Member    @relation(fields: [memberId], references: [id])

  @@index([copyId])
  @@index([memberId])
  @@index([returnedAt])
  @@map("book_loans")
}
```

## ディスカッションテーマ

1. **ナチュラルキー vs サロゲートキー**：UUID / auto increment / ビジネスキー、それぞれのメリット・デメリットは？
2. **ソフトデリート**：`deleted_at` カラムを使うソフトデリートは良い設計か？どんな場面で適切か？
3. **モデルの進化**：ビジネス要件の変化に伴うデータモデルの変更をどう管理するか？

## 課題

### 復習問題

1. 概念モデル・論理モデル・物理モデルの違いを、それぞれの目的と対象読者の観点から説明してください
2. M:N関係を中間テーブルに分解する理由を説明してください
3. ソフトデリートを採用する場合に注意すべき点を3つ挙げてください

### コード課題

「タスク管理ツール」のデータモデルを設計してください。

要件は次のとおり。

- ユーザーがプロジェクトを作成し、プロジェクト内にタスクを作成できる
- タスクには担当者を割り当てられる（複数人も可）
- タスクにはコメントを投稿できる
- タスクにはラベル（タグ）を複数付けられる

成果物は次の3つ。

- 概念モデル（ER図、またはテキストでの関連記述）
- Prisma schema形式の物理モデル
- インデックス設計とその根拠

達成基準は次のとおり。

- **最低限（30分）**: ユーザー・プロジェクト・タスク・コメントの4つのエンティティと、その関係を概念モデルに書く。同じ4つをPrisma schemaの物理モデルにする
- **目標**: 担当者（タスクとユーザー）とラベル（タスクとラベル）のM:Nを、中間テーブルに分解している。インデックスごとに、どのクエリを速くするためのものかを書いている
- **ヒント**: 「1つのタスクに担当者は何人いるか」「1人のユーザーは何件のタスクを担当するか」のように、関係ごとに両方向から数を確認する

### 発展課題（任意）

自分のプロジェクトの既存テーブル構造を概念モデルに逆変換し、現在のモデルの改善点を洗い出してください。

## 参考資料

- 『楽々ERDレッスン』羽生章洋
- 『達人に学ぶDB設計 徹底指南書 第2版』ミック（翔泳社, 2024）
- [dbdiagram.io](https://dbdiagram.io/)：ER図作成ツール
- 用語の確認: [用語集](../../docs/glossary.md)

## 次回との接続

データモデリングで学んだ「ビジネスの構造をモデルとして表現する」という考え方は、第4回（デザインパターン）のオブジェクト設計と、第5回（DDD）のドメインモデリングでも前提になる。
