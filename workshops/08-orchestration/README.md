# 第8回：オーケストレーション

> インフラの自動化と標準化が、サービスのスケーラビリティを支える

## 基本情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 2〜2.5時間 |
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

#### Docker の基本概念

- コンテナとは — プロセスの隔離とパッケージング
- イメージとコンテナの関係
- Dockerfile の書き方とベストプラクティス
- マルチステージビルドによるイメージの軽量化

#### Node.js アプリケーションのコンテナ化

```dockerfile
# マルチステージビルドの例
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

#### Docker Compose

- 複数コンテナの定義と管理
- ローカル開発環境での活用
- サービス間ネットワークとボリューム管理

### 3. コンテナオーケストレーション（30分）

#### Kubernetes の概要

- Kubernetes が解決する問題
- 主要コンポーネントの理解
  - Pod、Deployment、Service、Ingress
  - ConfigMap、Secret
  - Namespace

#### Kubernetes の基本リソース

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
- Istio / Linkerd の概要
- サイドカープロキシパターン

#### サービスメッシュが提供する機能

- トラフィック管理（ルーティング、負荷分散）
- セキュリティ（mTLS、認証・認可）
- 可観測性（メトリクス、トレーシング、ログ）
- 耐障害性（リトライ、Circuit Breaker、タイムアウト）

### 5. CI/CD パイプライン（20分）

#### CI（継続的インテグレーション）

- ビルド → テスト → 静的解析の自動化
- GitHub Actions / GitLab CI の概要
- テスト戦略（ユニット、統合、E2E）

#### CD（継続的デリバリー / デプロイメント）

- デプロイ戦略
  - ローリングアップデート
  - Blue-Green デプロイメント
  - カナリアリリース
- 各戦略のトレードオフ

#### CI/CD パイプラインの例

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
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
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

1. **Kubernetes の学習コスト** — Kubernetes は全員が理解すべきか？プラットフォームチームに任せるべきか？
2. **デプロイ戦略の選択** — ローリング、Blue-Green、カナリアの中で、自分のプロジェクトに最適なのはどれか？
3. **サーバーレス vs コンテナ** — AWS Lambda 等のサーバーレスとコンテナオーケストレーションの使い分けは？

## 課題

### 復習問題

1. コンテナ化のメリットを3つ挙げ、それぞれ説明してください
2. Kubernetes の Pod、Deployment、Service の関係を説明してください
3. Blue-Green デプロイメントとカナリアリリースの違いを説明してください

### コード課題

第7回の課題で設計したECサイトのマイクロサービスのうち、1つのサービス（注文サービス推奨）について以下を作成してください。

成果物：
- Dockerfile（マルチステージビルド）
- docker-compose.yml（サービス + データベース）
- GitHub Actions のCI設定ファイル（lint + test + build）
- Kubernetes の Deployment / Service マニフェスト

### 発展課題（任意）

docker-compose を使って、注文サービスと在庫サービスの2つのマイクロサービスをローカルで起動し、API通信させてください。ヘルスチェックエンドポイントを含めてください。

## 参考資料

- 『Kubernetes完全ガイド 第2版』青山真也
- 『Docker/Kubernetes 開発・運用のためのセキュリティ実践ガイド』須田瑛大 他
- [Kubernetes Documentation](https://kubernetes.io/ja/docs/home/) — 公式ドキュメント（日本語）

## 次回との接続

第8回ではサービスのデプロイ・管理を学びました。第9回（イベント駆動）では、マイクロサービス間のもう一つの重要な通信パターンであるイベント駆動アーキテクチャを学びます。非同期メッセージング、CQRS、Sagaパターンを扱います。
