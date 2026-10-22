# アーキテクチャ登竜門

> ソフトウェアアーキテクチャの基礎から実践まで、段階的に学ぶワークショップシリーズ

## 概要

「アーキテクチャ登竜門」は、実務経験2〜5年目のエンジニアを対象にした、座学中心のワークショップです。ソフトウェアアーキテクチャの基礎から応用までを、**全11回・約3ヶ月**で体系的に学びます。

技術スタックはTypeScript / Node.jsです。講義では、概念の説明にTypeScriptの具体的なコード例を添えます。

## カリキュラム

| 回 | テーマ | 内容 |
|----|--------|------|
| 第0回 | [オリエンテーション](workshops/00-orientation/README.md) | 企画の全体像、学習ロードマップの共有 |
| 第1回 | [オブジェクト指向](workshops/01-object-oriented/README.md) | SOLID原則、カプセル化、ポリモーフィズム |
| 第2回 | [リレーショナルデータベース](workshops/02-relational-database/README.md) | 正規化、インデックス、トランザクション |
| 第3回 | [データモデリング](workshops/03-data-modeling/README.md) | ER図、概念・論理・物理モデル |
| 第4回 | [デザインパターン](workshops/04-design-patterns/README.md) | GoFパターン（Strategy、Observer、Factory等）から実務で使うものを厳選、GoF以外の実務パターン（Repository） |
| 第5回 | [ドメイン駆動設計](workshops/05-domain-driven-design/README.md) | 戦略的設計・戦術的設計、ユビキタス言語 |
| 第6回 | [クリーンアーキテクチャ](workshops/06-clean-architecture/README.md) | 依存性逆転、レイヤー分離、境界の設計 |
| 第7回 | [マイクロサービス](workshops/07-microservices/README.md) | サービス分割、API設計、データ独立性 |
| 第8回 | [オーケストレーション](workshops/08-orchestration/README.md) | コンテナ管理、サービスメッシュ、CI/CD |
| 第9回 | [イベント駆動](workshops/09-event-driven/README.md) | メッセージング、CQRS、結果整合性 |
| 第10回 | [総合演習・振り返り](workshops/10-capstone/README.md) | 全体の総括、アーキテクチャ設計レビュー |

## ドキュメント

### 参加者向け

- [概念理解ガイド](docs/concept-guide.md)：全概念について「なぜ必要か」と概念同士のつながりを通しで解説
- [用語集](docs/glossary.md)：各回の重要用語の一言定義
- [概念の関連図](docs/visual-concept-map.md)：概念同士の関係を図で確認
- [企画概要・コンセプト](docs/overview.md)：目的、背景、対象者の詳細
- [全体スケジュール](docs/schedule.md)：週次の詳細スケジュール
- [学習の流れ・トピック関連図](docs/progression.md)：トピック間の依存関係と学習順序
- [設計力セルフアセスメント](docs/assessment-template.md)：開始時と終了時に同じ項目で自己評価するためのテンプレート
- [開発手法とアーキテクチャの関係](docs/development-methodology-guide.md)：本編で扱わない開発手法とアーキテクチャの関係をまとめた補足資料

### 講師向け

- 各回の講師ノート（`workshops/XX-xxx/instructor-notes.md`）：時間配分・想定質問・つまずきポイント
- [講師・ファシリテーター準備ガイド](docs/facilitator-prep-guide.md)：各回の講師向け準備チェックリストと進行ガイド
- [講師のための自己学習ガイド](docs/instructor-learning-guide.md)：各回で深めるべきポイントと推奨リーディング
- [ディスカッションファシリテーション集](docs/discussion-facilitation-kit.md)：各回のディスカッションシナリオと進行台本
- [回をまたぐ「つなぎ台本」](docs/session-bridge-scripts.md)：前回の振り返りと次回への橋渡しフレーズ集
- [講師の理解度セルフチェックシート](docs/instructor-self-check.md)：教えられるレベルに達しているかの確認リスト
- [「よくある誤解」ハンドブック](docs/common-misconceptions.md)：各テーマの典型的な誤解と正しい理解
- [参考書籍クイックリファレンス](docs/instructor-reading-list.md)：講師向け優先読書ガイド
- [改善提案まとめ](docs/improvement-proposals.md)：今後の進め方についての改善点と追加トピックの提案

## 対象者

- 実務経験2〜5年目のソフトウェアエンジニア
- TypeScript / JavaScriptの基本的な文法を理解している
- チーム開発の経験がある
- 設計やアーキテクチャについて体系的に学びたいと感じている

## 形式

- 座学中心：講義で概念を学び、コード例で具体的に確認する
- 週1回開催：各回約2.5時間（講義約100分 + 休憩10分 + 議論・Q&A 30分 + 振り返り・予告10分）。第0回（オリエンテーション）のみ約90分
- 課題あり：各回の終了後に復習課題を出す。参加者は翌週までに取り組む
- 技術スタック：TypeScript / Node.js（Node.js 24 LTSを想定）

## ディレクトリ構成

```
architecture_toryumon/
├── README.md                  # このファイル
├── docs/
│   ├── overview.md            # 企画概要・コンセプト
│   ├── schedule.md            # 全体スケジュール
│   ├── progression.md         # 学習の流れ・トピック関連図
│   ├── concept-guide.md       # 概念理解ガイド
│   ├── glossary.md            # 用語集
│   ├── visual-concept-map.md  # 概念の関連図
│   ├── assessment-template.md # 設計力セルフアセスメント
│   ├── development-methodology-guide.md # 開発手法とアーキテクチャの関係
│   ├── facilitator-prep-guide.md # 講師・ファシリテーター準備ガイド
│   ├── instructor-learning-guide.md # 講師のための自己学習ガイド
│   ├── discussion-facilitation-kit.md # ディスカッションファシリテーション集
│   ├── session-bridge-scripts.md # 回をまたぐ「つなぎ台本」
│   ├── instructor-self-check.md # 講師の理解度セルフチェックシート
│   ├── common-misconceptions.md # 「よくある誤解」ハンドブック
│   ├── instructor-reading-list.md # 参考書籍クイックリファレンス
│   └── improvement-proposals.md # 改善提案まとめ
└── workshops/
    ├── 00-orientation/        # 第0回：オリエンテーション
    │   ├── README.md          # 参加者向け資料
    │   └── instructor-notes.md # 講師ノート
    ├── 01-object-oriented/    # 第1回：オブジェクト指向（以降の各回も README.md と instructor-notes.md を持つ）
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
