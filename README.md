# k8s_learning

Cloud RunからGKEへの移行を見据えたKubernetes学習プロジェクト。用語は [CONTEXT.md](CONTEXT.md)、設計判断は [docs/adr/](docs/adr/) を参照。

## クラスターのブートストラップ

クラスターはこのリポジトリ(GitOps Repository)から完全に復元できる。Gitに置けないのは以下の手動手順だけ。

### 1. kindクラスターを作成

```sh
kind create cluster --name k8s-learning --config cluster/kind-config.yaml
```

### 2. ArgoCDをインストール

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.2/manifests/install.yaml
kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s
```

### 3. root Applicationを適用

```sh
kubectl apply -f manifests/argocd/root-app.yaml
```

以降はApp of Apps経由で全Application(App-A・App-B・Platform Component・Gateway設定)が自動で同期される。CRDの導入順の関係で最初の数分はSyncFailedが出るが、リトライで収束する。

### 4. 動作確認

```sh
kubectl get applications -n argocd        # 全てSynced/Healthyになるまで待つ
curl http://app-a.localhost/              # Envoy Gateway経由でApp-Aに到達
curl http://app-b.localhost/              # 同App-B
```

### ArgoCD UIへのアクセス

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
# https://localhost:8080 に admin / 上記パスワードでログイン
```

## 手動管理のSecret

現時点ではなし。Backstage導入時にGitHub PATなどが加わる予定(追加時にここへ手順を記載する)。
