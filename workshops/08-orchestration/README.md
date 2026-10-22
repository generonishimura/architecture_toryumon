# 第8回：オーケストレーション

> インフラの自動化と標準化が、サービスのスケーラビリティを支える

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2.5時間（150分） |
| 前提知識 | 第7回（マイクロサービス）の内容、Dockerの基礎知識（あれば望ましい） |
| 到達目標 | コンテナ化・オーケストレーション・CI/CDの概念を理解し、サービス運用の全体像を把握できる |

## この回のキーメッセージ

> 「インフラの自動化と標準化が、サービスのスケーラビリティを支える」

## 講義内容

### 1. なぜオーケストレーションが必要か（10分）

- マイクロサービスの運用課題の振り返り
- サービス数が増えると何が大変になるか
  - デプロイ、スケーリング、監視、ネットワーク管理
- 手動運用の限界と自動化の必要性

### 2. コンテナ技術（25分）

#### Dockerの基本概念

- コンテナとは：プロセスの隔離とパッケージング
- イメージとコンテナの関係
- Dockerfileの書き方とベストプラクティス
- マルチステージビルドによるイメージの軽量化

#### Node.jsアプリケーションのコンテナ化

```dockerfile
# マルチステージビルドの例
FROM node:24-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
# prisma.config.ts は DATABASE_URL を読むので、生成の時だけダミーの値を渡す（接続はしない）
RUN DATABASE_URL="postgresql://build:build@localhost:5432/build" npx prisma generate
RUN npm run build
RUN npm prune --omit=dev   # 実行に不要な devDependencies を削除

FROM node:24-alpine AS runner
ENV NODE_ENV=production
WORKDIR /app
COPY --from=builder /app/package.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

このDockerfileは、次の前提で書いている。

- `npm run build` はesbuildで `src/main.ts` を1つのファイルにまとめる。package.jsonのscriptsには `"build": "esbuild src/main.ts --bundle --platform=node --format=esm --packages=external --outfile=dist/main.js"` と書く（`npm install -D esbuild`）
- `tsc` ではビルドしない。教材のコードはES moduleで、相対importに拡張子を付けていない（`'./generated/prisma/client'` など）。`tsc` はこのimportを書き換えずに出力するので、`node dist/main.js` の起動時にNode.jsがファイルを見つけられない。esbuildは相対importのファイルを1つにまとめるので、この問題が起きない。型チェックは別に `npx tsc --noEmit` で行う
- `--packages=external` を付けているので、npmパッケージ（`@prisma/adapter-pg`、`dotenv`、`express` など）はまとめずに `node_modules` から読み込む。そのため、これらは `-D` を付けずにdependenciesに入れる
- Prismaの生成先は `src/generated/prisma` である（第2回を参照）。生成先はGitに含めないのが普通なので、Dockerfileの中で `npx prisma generate` を実行してからビルドする

`COPY . .` でホストのファイルがイメージに入らないように、`.dockerignore` を置く。

```
# .dockerignore の例
node_modules
dist
.git
.env*
```

- `.env` をイメージに含めない。秘密情報は、実行時に環境変数やKubernetesのSecretで渡す
- 本教材はNode.js 24 LTSを想定する（2028年4月までサポート）

#### Docker Compose

- 複数コンテナの定義と管理
- ローカル開発環境での活用
- サービス間ネットワークとボリューム管理

### 3. コンテナオーケストレーション（30分）

#### Kubernetesの概要

- Kubernetesで解決できる問題
- 主要コンポーネントの理解
  - Pod、Deployment、Service、Ingress / Gateway API
  - ConfigMap、Secret
  - Namespace

> 注: Ingressは、外部からのHTTPアクセスをServiceに振り分ける従来の仕組みである。Ingress APIに新機能は追加されなくなった。代表的な実装のingress-nginxは、2026年3月に保守を終了した。新しく構築するなら、後継のGateway API（Gateway / HTTPRouteリソース）を検討する。

> 注: この回の「オーケストレーション」は、コンテナの配置・起動・スケーリングを自動で管理することを指す。第9回のSagaパターンにも「オーケストレーション」という方式が出てくる。こちらは、1つのサービスが業務処理の実行順序を指示する方式で、別の概念である。

#### Kubernetesの基本リソース

```yaml
# Deployment の例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.2.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: order-db-secret
              key: url
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
```

#### スケーリング戦略

- 水平スケーリング（Horizontal Pod Autoscaler）
- 垂直スケーリング
- スケーリングの判断基準（CPU、メモリ、カスタムメトリクス）

### 4. サービスメッシュ（15分）

#### サービスメッシュとは

- サービス間通信のインフラ層を抽象化
- Istio / Linkerdの概要
- サイドカープロキシパターン

#### サービスメッシュが提供する機能

- トラフィック管理（ルーティング、負荷分散）
- セキュリティ（mTLS、認証・認可）
- 可観測性（メトリクス、トレーシング、ログ）
- 耐障害性（リトライ、Circuit Breaker、タイムアウト）

### 5. CI/CDパイプライン（20分）

#### CI（継続的インテグレーション）

- ビルド → テスト → 静的解析の自動化
- GitHub Actions / GitLab CIの概要
- テスト戦略（ユニット、統合、E2E）

#### CD（継続的デリバリー / デプロイメント）

- デプロイ戦略
  - ローリングアップデート
  - Blue-Greenデプロイメント
  - カナリアリリース
- 各戦略のトレードオフ

#### CI/CDパイプラインの例

```yaml
# GitHub Actions の例
name: CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: '24'
      - run: npm ci
      - run: npm run lint
      - run: npm test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Build and push Docker image
        run: |
          docker build -t myregistry/order-service:${{ github.sha }} .
          docker push myregistry/order-service:${{ github.sha }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/order-service \
            order-service=myregistry/order-service:${{ github.sha }}
```

## ディスカッションテーマ

1. **Kubernetesの学習コスト**：Kubernetesは全員が理解すべきか？プラットフォームチームに任せるべきか？
2. **デプロイ戦略の選択**：ローリング、Blue-Green、カナリアの中で、自分のプロジェクトに最適なのはどれか？
3. **サーバーレス vs コンテナ**：AWS Lambda等のサーバーレスとコンテナオーケストレーションの使い分けは？

## 課題

### 復習問題

1. コンテナ化のメリットを3つ挙げ、それぞれ説明してください
2. KubernetesのPod、Deployment、Serviceの関係を説明してください
3. Blue-Greenデプロイメントとカナリアリリースの違いを説明してください

### コード課題

第7回の課題で設計したECサイトのマイクロサービスのうち、1つのサービス（注文サービス推奨）について以下を作成してください。

成果物は次の4つ。

- Dockerfile（マルチステージビルド）
- docker-compose.yml（サービス + データベース）
- GitHub ActionsのCI設定ファイル（lint + test + build）
- KubernetesのDeployment / Serviceマニフェスト

### 発展課題（任意）

docker-composeを使って、注文サービスと在庫サービスの2つのマイクロサービスをローカルで起動し、API通信させてください。ヘルスチェックエンドポイントを含めてください。

## 参考資料

- 『Kubernetes完全ガイド 第2版』青山真也
- 『Docker/Kubernetes 開発・運用のためのセキュリティ実践ガイド』須田瑛大 他
- [Kubernetes Documentation](https://kubernetes.io/ja/docs/home/)：公式ドキュメント（日本語）

## 次回との接続

第8回では、サービスのデプロイと管理を学んだ。第9回（イベント駆動）では、マイクロサービス間のもう一つの通信パターンであるイベント駆動アーキテクチャを学ぶ。非同期メッセージング、CQRS、Sagaパターンを扱う。
