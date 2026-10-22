# 第4回：デザインパターン

> 設計の「型」を知り、適切に適用する判断力を養う

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | 第1回（オブジェクト指向）の内容、特にSOLID原則 |
| 到達目標 | 実務で頻出するデザインパターンを理解し、問題に応じて適用判断ができる |

## この回のキーメッセージ

> 「パターンは道具。問題に合わせて選択し、過剰適用を避ける」

## 講義内容

### 1. デザインパターンとは（10分）

- GoF（Gang of Four）の23パターンとその背景
- パターンは「発明」ではなく「発見」：繰り返し現れる解決策に名前を付けたもの
- パターンを学ぶ意義：チームの共通語彙になる、設計判断の選択肢が増える
- アンチパターン：パターンの過剰適用（パターン病）

### 2. 生成に関するパターン（20分）

#### Factory Method / Abstract Factory

- オブジェクト生成のロジックを分離する
- 「どのクラスを生成するか」の判断をカプセル化
- TypeScript例：通知チャネル（Email / Slack / SMS）のファクトリ

#### Builder

- 複雑なオブジェクトを段階的に構築する
- メソッドチェインによる可読性の向上
- TypeScript例：クエリビルダー、設定オブジェクトの構築

### 3. 構造に関するパターン（20分）

#### Adapter

- 既存のインターフェースを期待されるインターフェースに変換する
- 外部ライブラリやレガシーコードとの接続に活用
- TypeScript例：外部決済APIのアダプター

#### Facade

- 複雑なサブシステムに対するシンプルなインターフェースを提供
- TypeScript例：注文処理のファサード（在庫確認 + 決済 + 配送手配）

### 4. 振る舞いに関するパターン（20分）

#### Strategy

- アルゴリズムをカプセル化し、実行時に切り替えられるようにする
- 第1回で学んだ開放閉鎖の原則の実践
- TypeScript例：価格計算戦略（通常 / 会員 / セール）

#### Observer

- オブジェクト間の1対多の依存関係を定義
- 状態変化を自動通知する仕組み
- TypeScript例：注文状態の変化に応じた通知
- EventEmitter / RxJSとの関連

### 5. GoF以外の実務パターン：Repository（15分）

- RepositoryはGoFの23パターンには含まれない。出典はMartin Fowler『Patterns of Enterprise Application Architecture』（2002）とEric Evans『Domain-Driven Design』（2003）
- データアクセスのロジックを抽象化する
- ドメインオブジェクトの永続化と取得のインターフェース
- 第1回の依存性逆転、第2回のRDB知識を統合
- TypeScript例：インメモリリポジトリとDBリポジトリの切り替え

### 6. パターンの適用判断（15分）

- 「問題があってパターンがある」：パターンありきで設計しない
- パターンを適用すべき兆候（コードの匂い）
- TypeScript / Node.jsエコシステムにおけるパターンの現れ方
  - ミドルウェアパターン（Express / Koa）
  - プラグインパターン（各種ライブラリ）
  - DIコンテナ（NestJS）

## コード例の概要

```typescript
// Strategy パターン
interface PricingStrategy {
  calculate(basePrice: number): number;
}

class RegularPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice;
  }
}

class MemberPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice * 0.9; // 10% off
  }
}

class SalePricing implements PricingStrategy {
  constructor(private readonly discountRate: number) {}
  calculate(basePrice: number): number {
    return basePrice * (1 - this.discountRate);
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  calculatePrice(basePrice: number): number {
    return this.strategy.calculate(basePrice);
  }
}
```

```typescript
// src/repositories/UserRepository.ts
// Repository パターン（GoF ではなく、PoEAA / DDD のパターン）
// Prisma 7 では、schema.prisma の generator で指定した output（"../src/generated/prisma"）から import する
// このファイルは src/repositories/ にあるので、src/generated/prisma/client への相対パスは '../generated/prisma/client' になる
import { PrismaClient } from '../generated/prisma/client';

interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// テスト用：インメモリ実装
class InMemoryUserRepository implements UserRepository {
  private users: Map<string, User> = new Map();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }
  // ...
}

// 本番用：Prisma実装
// PrismaClient は第2回と同じく driver adapter（PrismaPg）を渡して生成し、コンストラクタで受け取る
class PrismaUserRepository implements UserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } });
    return data ? UserMapper.toDomain(data) : null;
  }
  // ...
}
```

## ディスカッションテーマ

1. **パターンの過剰適用**：「パターンを使いすぎた」経験や、見かけたコードはあるか？
2. **フレームワークとパターン**：Express、NestJS、Next.js等で、どのデザインパターンが使われているか？
3. **パターンの命名効果**：チーム内で「このクラスはStrategyパターンだ」と言えることの価値は何か？

## 課題

### 復習問題

1. Factory MethodパターンとStrategyパターンの違いを説明してください
2. Adapterパターンが必要になる具体的な場面を2つ挙げてください
3. Repositoryパターンを使うことで得られるテスト上の利点を説明してください

### コード課題

ECサイトの「配送料金計算」をデザインパターンを用いて設計してください。

要件は次のとおり。

- 配送方法は「通常配送」「速達」「コンビニ受取」の3種類
- 地域（都道府県）によって料金が異なる
- 一定金額以上の注文で送料無料になる
- 将来的に新しい配送方法の追加が見込まれる

成果物は次の2つ。

- どのパターンを適用したかと、その理由
- TypeScriptのコード（インターフェースと主要クラスの実装）

達成基準は次のとおり。

- **最低限（30分）**: 配送方法ごとの料金計算をStrategyパターンで書く。`ShippingFeeStrategy` インターフェースと、3種類の配送方法の実装クラスを用意する
- **目標**: 新しい配送方法を追加するときに、既存のクラスを変更しなくてよい（OCP）。地域ごとの料金と送料無料の判定が、配送方法の各クラスに重複して書かれていない
- **ヒント**: 「配送方法ごとに変わる部分」と「どの配送方法でも同じ部分（送料無料の判定など）」を分けて考える

### 発展課題（任意）

NestJSのソースコードを読み、どのデザインパターンが使われているかを3つ以上特定し、それぞれの役割を説明してください。

## 参考資料

- 『Head First デザインパターン 第2版』Eric Freeman 他（オライリー・ジャパン, 2022）
- 『リファクタリング 既存のコードを安全に改善する（第2版）』Martin Fowler
- [Refactoring Guru — Design Patterns](https://refactoring.guru/design-patterns)
- 用語の確認: [用語集](../../docs/glossary.md)

## 次回との接続

第4回で学んだRepositoryパターンとFactoryパターンは、第5回（DDD）の戦術的設計で、構成要素としてそのまま使う。Strategyパターンも、ドメインサービスや料金計算の切り替えで使うことがある。第5回では、これらのパターンを組み合わせてドメインモデルを作る。
