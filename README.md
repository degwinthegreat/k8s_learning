# k8s_learning

Cloud RunからGKEへの移行を見据えたKubernetes学習プロジェクト。用語は [CONTEXT.md](CONTEXT.md)、設計判断は [docs/adr/](docs/adr/) を参照。

## クラスターのブートストラップ

クラスターはこのリポジトリ(GitOps Repository)から完全に復元できる。Gitに置けないのは以下の手動手順だけ。

### 1. kindクラスターを作成

```sh
kind create cluster --name k8s-learning --config cluster/kind-config.yaml
```

### 2. Backstageのイメージをビルドしてロード

Backstageは自作イメージのためレジストリに置かず、ローカルでビルドしてkindに直接ロードする。

```sh
cd backstage
yarn install --immutable
yarn tsc
yarn build:backend
docker build -t backstage:0.1.0 -f packages/backend/Dockerfile .
kind load docker-image backstage:0.1.0 --name k8s-learning
cd ..
```

### 3. ArgoCDをインストール

```sh
kubectl create namespace argocd
# ApplicationSetのCRDがclient-side applyのアノテーション上限を超えるためserver-sideで適用する
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.2/manifests/install.yaml
kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
```

### 4. root Applicationを適用

```sh
kubectl apply -f manifests/argocd/root-app.yaml
```

以降はApp of Apps経由で全Application(App-A・App-B・Platform Component・Gateway設定・Backstage)が自動で同期される。CRDの導入順や下記の手動Secretの関係で最初の数分はSyncFailedが出るが、リトライで収束する。

### 5. 手動Secretを作成

Backstageが参照する `backstage-secrets` はGitに置けないため手動で作成する。
ArgoCDトークンは、root Application同期後(argocd-cmに `accounts.backstage` が反映された後)に発行する。

```sh
# argocd CLI(brew install argocd)のcoreモードでbackstageアカウントのトークンを発行する
kubectl config set-context --current --namespace=argocd
ARGOCD_TOKEN=$(argocd account generate-token --account backstage --core)
kubectl config set-context --current --namespace=default

kubectl -n backstage create secret generic backstage-secrets \
  --from-literal=POSTGRES_PASSWORD=$(openssl rand -hex 16) \
  --from-literal=BACKEND_SECRET=$(openssl rand -base64 24) \
  --from-literal=ARGOCD_AUTH_TOKEN=$ARGOCD_TOKEN \
  --from-literal=GITHUB_TOKEN=$(gh auth token)
```

Secret作成前はBackstageとPostgreSQLのPodが `CreateContainerConfigError` で待機するが、作成後に自動で起動する。

### 6. 動作確認

```sh
kubectl get applications -n argocd        # 全てSynced/Healthyになるまで待つ
curl http://app-a.localhost/              # Envoy Gateway経由でApp-Aに到達
curl http://app-b.localhost/              # 同App-B
curl http://backstage.localhost/          # Backstage (ブラウザで開くとSoftware Catalog)
```

### ArgoCD UIへのアクセス

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080 に admin / 上記パスワードでログイン
```

## 手動管理のSecret

| Secret | Namespace | キー | 用途 |
|---|---|---|---|
| backstage-secrets | backstage | POSTGRES_PASSWORD | BackstageのPostgreSQL接続 |
| backstage-secrets | backstage | BACKEND_SECRET | Backstageバックエンドのサービス間認証鍵 |
| backstage-secrets | backstage | ARGOCD_AUTH_TOKEN | Roadie ArgoCDプラグインの読み取りアクセス(backstageアカウント・role:readonly) |
| backstage-secrets | backstage | GITHUB_TOKEN | ScaffolderのPR作成とカタログ取得(`repo`スコープのPAT。`gh auth token`で流用可) |

## Golden Path (Software Template)

Backstageの「Create」から **学習用ワークロード (Golden Path)** テンプレートを実行すると、
Rollout・Service・HTTPRoute・ScaledObject・ArgoCD Application・カタログ登録の一式が
このリポジトリへのPull Requestとして生成される([templates/workload/](templates/workload/))。

マージ後は人手の操作は不要:

1. root Application(App of Apps)が `manifests/argocd/apps/` の新しいApplicationを検出してデプロイ
2. カタログのワイルドカードlocation(`catalog/*.yaml`)が新しいComponentを自動登録
3. `http://<name>.localhost/` で到達確認できる

Backstageの設定([app-config.production.yaml](backstage/app-config.production.yaml))は
イメージに焼き込まれるため、設定変更時は手順2の再ビルドとロード、
`kubectl -n backstage rollout restart deployment backstage` が必要。
