# k8s Learning — Cloud Run to GKE Migration

Cloud RunからGKEへの移行を見据えたKubernetes学習プロジェクト。2つのRailsアプリケーションを複数Namespaceで運用し、ArgoCD・Argo Rollouts・KEDAを使ったクラスター管理を手を動かしながら習得する。

## Language

### アプリケーション

**App-A**:
2つのRailsアプリケーションのうちの一方。独自のNamespaceとArgoCDのApplicationを持つ。
_Avoid_: アプリA、service-a、app_a

**App-B**:
2つのRailsアプリケーションのうちのもう一方。独自のNamespaceとArgoCDのApplicationを持つ。
_Avoid_: アプリB、service-b、app_b

### リポジトリ

**Application Repository**:
App-AまたはApp-BのRailsソースコードを管理するGitリポジトリ。アプリごとに1つ存在する（ポリレポ）。
_Avoid_: アプリリポジトリ、ソースリポジトリ

**GitOps Repository**:
クラスター全体のKubernetesマニフェストを一元管理するGitリポジトリ。ArgoCDがここを監視してクラスターに反映する。
_Avoid_: マニフェストリポジトリ、k8sリポジトリ、インフラリポジトリ

### トラフィック

**Internal Traffic**:
App-AとApp-Bがクラスター内でHTTPにより互いに呼び合う通信。KubernetesのCluster-internal DNSを使う（例: `http://app-b.app-b-ns.svc.cluster.local`）。
_Avoid_: 内部通信、サービス間通信、East-West traffic

**External Traffic**:
クラスター外部（ブラウザ・外部クライアント）からApp-AまたはApp-Bへの通信。Gateway APIのHTTPRouteを経由してルーティングされる。
_Avoid_: 外部通信、インバウンドトラフィック、North-South traffic

### GitOps・マニフェスト管理

**App of Apps**:
ArgoCDのルートApplicationが、App-AとApp-BそれぞれのApplicationを子として管理するパターン。GitOps Repositoryのルートに親Applicationのマニフェストを置く。
_Avoid_: ApplicationSet、親子Application

**Plain YAML**:
HelmやKustomizeを介さず生のKubernetesマニフェストをGitOps Repositoryに配置する管理スタイル。学習フェーズで採用し、環境差分が必要になった時点でKustomizeに移行する。
_Avoid_: raw manifest、素のYAML

### デプロイ

**Rollout**:
Argo Rolloutsが管理するデプロイメントオブジェクト。学習フェーズではBlue-Greenを使い、本番GKEではCanaryを使う想定。
_Avoid_: デプロイ、Deployment（KubernetesのDeploymentと混同するため）

### スケーリング

**ScaledObject**:
KEDAが管理するスケーリング設定オブジェクト。学習フェーズではCronトリガーを使い、後にHTTPトリガーも試す。
_Avoid_: スケーリング設定、HPA

## Example Dialogue

> **Dev**: App-AからApp-Bを呼ぶとき、URLはどう書けばいい？
>
> **Domain expert**: Internal Trafficだから、`http://app-b.app-b-ns.svc.cluster.local` を使って。Cloud Runのときみたいにインターネット経由じゃなくて、Cluster-internal DNSで直接届く。
>
> **Dev**: じゃあ外からApp-Aを叩くときは？
>
> **Domain expert**: それはExternal Traffic。HTTPRouteにホスト名とパスを定義してEnvoy Gatewayに流す。GitOps Repositoryのマニフェストに書いてArgoCDに反映させる流れ。
>
> **Dev**: RolloutとDeploymentって何が違う？
>
> **Domain expert**: RolloutはArgo Rolloutsのリソース。KubernetesのDeploymentをほぼ置き換える形で使う。今はBlue-Greenで入門して、本番GKEではCanaryに切り替える予定。
