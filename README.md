# アーキテクチャ登竜門

> ソフトウェアアーキテクチャの基礎から実践まで、段階的に学ぶワークショップシリーズ

## 概要

「アーキテクチャ登竜門」は、実務経験2〜5年目のエンジニアを対象に、ソフトウェアアーキテクチャの基礎から応用までを**全11回・約3ヶ月**で体系的に学ぶ座学中心のワークショップです。

技術スタックとして **TypeScript / Node.js** を使用し、概念だけでなく具体的なコード例を交えて理解を深めます。

## カリキュラム

| 回 | テーマ | 内容 |
|----|--------|------|
| 第0回 | [オリエンテーション](workshops/00-orientation/README.md) | 企画の全体像、学習ロードマップの共有 |
| 第1回 | [オブジェクト指向](workshops/01-object-oriented/README.md) | SOLID原則、カプセル化、ポリモーフィズム |
| 第2回 | [リレーショナルデータベース](workshops/02-relational-database/README.md) | 正規化、インデックス、トランザクション |
| 第3回 | [データモデリング](workshops/03-data-modeling/README.md) | ER図、概念・論理・物理モデル |
| 第4回 | [デザインパターン](workshops/04-design-patterns/README.md) | GoFパターン、実務で使えるパターン厳選 |
| 第5回 | [ドメイン駆動設計](workshops/05-domain-driven-design/README.md) | 戦略的設計・戦術的設計、ユビキタス言語 |
| 第6回 | [クリーンアーキテクチャ](workshops/06-clean-architecture/README.md) | 依存性逆転、レイヤー分離、境界の設計 |
| 第7回 | [マイクロサービス](workshops/07-microservices/README.md) | サービス分割、API設計、データ独立性 |
| 第8回 | [オーケストレーション](workshops/08-orchestration/README.md) | コンテナ管理、サービスメッシュ、CI/CD |
| 第9回 | [イベント駆動](workshops/09-event-driven/README.md) | メッセージング、CQRS、結果整合性 |
| 第10回 | [総合演習・振り返り](workshops/10-capstone/README.md) | 全体の総括、アーキテクチャ設計レビュー |

## ドキュメント

- [企画概要・コンセプト](docs/overview.md) - 目的、背景、対象者の詳細
- [全体スケジュール](docs/schedule.md) - 週次の詳細スケジュール
- [学習の流れ・トピック関連図](docs/progression.md) - トピック間の依存関係と学習順序
- [講師・ファシリテーター準備ガイド](docs/facilitator-prep-guide.md) - 各回の講師向け準備チェックリストと進行ガイド

## 対象者

- 実務経験 2〜5年目のソフトウェアエンジニア
- TypeScript / JavaScript の基本的な文法を理解している
- チーム開発の経験がある
- 設計やアーキテクチャについて体系的に学びたいと感じている

## 形式

- **座学中心** - 講義で概念を学び、コード例で具体化
- **週1回開催** - 各回 2〜2.5時間（講義90分 + 議論・Q&A 30〜60分）
- **課題あり** - 各回終了後に復習課題を出題（翌週までに取り組む）
- **技術スタック** - TypeScript / Node.js

## ディレクトリ構成

```
architecture_toryumon/
├── README.md                  # このファイル
├── docs/
│   ├── overview.md            # 企画概要・コンセプト
│   ├── schedule.md            # 全体スケジュール
│   ├── progression.md         # 学習の流れ・トピック関連図
│   └── facilitator-prep-guide.md # 講師・ファシリテーター準備ガイド
└── workshops/
    ├── 00-orientation/        # 第0回：オリエンテーション
    ├── 01-object-oriented/    # 第1回：オブジェクト指向
    ├── 02-relational-database/# 第2回：リレーショナルデータベース
    ├── 03-data-modeling/      # 第3回：データモデリング
    ├── 04-design-patterns/    # 第4回：デザインパターン
    ├── 05-domain-driven-design/# 第5回：ドメイン駆動設計
    ├── 06-clean-architecture/ # 第6回：クリーンアーキテクチャ
    ├── 07-microservices/      # 第7回：マイクロサービス
    ├── 08-orchestration/      # 第8回：オーケストレーション
    ├── 09-event-driven/       # 第9回：イベント駆動
    └── 10-capstone/           # 第10回：総合演習・振り返り
```
