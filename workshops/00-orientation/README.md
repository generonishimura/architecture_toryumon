# 第0回：オリエンテーション

> 全体像を共有し、学習の準備を整える

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 1.5時間 |
| 前提知識 | TypeScript基本文法、Git基本操作 |
| 到達目標 | ワークショップの全体像を理解し、学習環境が整っている状態 |

## アジェンダ

### 1. イントロダクション（15分）

- ワークショップ「アーキテクチャ登竜門」の趣旨説明
- 「アーキテクチャ」とは何か — ソフトウェアにおける設計判断の集合
- 本ワークショップで扱う範囲と扱わない範囲

### 2. 参加者自己紹介（15分）

- 名前、チーム、主な担当領域
- 現在のアーキテクチャに関する悩み・学びたいこと
- （事前アンケート結果があれば共有）

### 3. 全体ロードマップの紹介（15分）

- 9つのテーマの概要と順序の意図
- フェーズ構成（基礎 → 設計パターン → 分散システム）
- 各テーマがどうつながるかの関連図

### 4. 学習の進め方（15分）

- 各回の構成（講義 + ディスカッション + Q&A）
- 課題の取り組み方とフィードバックの仕組み
- 質問・議論の推奨（「分からない」を歓迎する文化）

### 5. 環境構築（15分）

環境構築は事前準備で済ませておく。当日は動作確認だけを行う。

- Node.js / TypeScript の開発環境の動作確認（`node --version` で Node.js 24 が入っていることを確認する）
- 作業用ディレクトリで、`hello.ts` が `npx tsx hello.ts` で実行できることを確認する
- エディタ（VS Code推奨）の設定

### 6. ウォームアップ：設計について考える（15分）

以下の問いについて、グループで議論する。

- 「良いコード」と「良い設計」の違いは何か？
- 設計で一番困った経験は？
- なぜアーキテクチャを学ぶ必要があるのか？

## 事前準備

### 参加者

- [ ] Node.js 24 をインストールする（nvm なら `nvm install 24`）。本教材は Node.js 24 LTS を想定している。Node.js 24 のサポート期限は2028年4月
- [ ] TypeScript の実行環境の準備（下の手順を参照）
- [ ] VS Code のインストール
- [ ] Git の設定確認
- [ ] 事前アンケートへの回答
- [ ] （任意）Docker と PostgreSQL のインストール：第2回のデモを手元で試したい人向け

**TypeScript の実行環境の準備手順**

TypeScript はグローバルにインストールしない。プロジェクトごとに `typescript` と `tsx` と `@types/node`（Node.js の型定義）を入れる。こうすると、プロジェクトごとに使う TypeScript のバージョンを固定できる。

```bash
mkdir toryumon-workspace && cd toryumon-workspace
npm init -y
npm pkg set type=module   # ES module として扱う（第2回の Prisma 7 の前提とそろえる）
npm install -D typescript tsx @types/node
echo "const message: string = 'hello'; console.log(message);" > hello.ts
npx tsx hello.ts   # hello と表示されれば準備完了
```

- Windows の PowerShell では、`echo ... > hello.ts` で作ったファイルが UTF-16 で保存され、`npx tsx` で実行できないことがある。Windows では、`hello.ts` をエディタで作る（または `Set-Content -Encoding utf8` を使う）
- 型チェックをするときは、作業用ディレクトリに次の `tsconfig.json` を置き、`npx tsc --noEmit` を実行する。`npx tsc --init` が出力する設定は `"module": "nodenext"` などになっており、拡張子のない相対 import（`'./generated/prisma/client'` など）がエラーになるので、この教材では使わない

```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "esnext",
    "moduleResolution": "bundler",
    "types": ["node"],
    "strict": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["src", "*.ts"]
}
```

- Express を使う回では、`@types/express` も必要になる（`npm install -D @types/express`）
- `node hello.ts` では、教材のコード（`constructor(private readonly ...)` のようなパラメータプロパティなど）を実行できない。教材のコードは `npx tsx` で実行する

### 運営側

- [ ] 事前アンケートの配布・集計
- [ ] 開発環境セットアップ手順書の準備
- [ ] 全体ロードマップのスライド準備
- [ ] 参加者リストの確定

## 次回予告

次回（第1回）は **オブジェクト指向** です。以下のキーワードについて事前に考えておくと理解が深まります。

- カプセル化、継承、ポリモーフィズムの違い
- SOLID原則（聞いたことがあれば）
- 自分が書いたコードで「設計が良い/悪い」と感じた経験
