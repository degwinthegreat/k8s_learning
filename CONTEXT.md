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
App-AとApp-Bがクラスター内でHTTPにより互いに呼び合う通信。KubernetesのCluster-internal DNSを使う（例: `http://app-b.app-b.svc.cluster.local`）。Namespace名はサービス名と同一（`-ns`サフィックスは付けない）。
_Avoid_: 内部通信、サービス間通信、East-West traffic

**External Traffic**:
クラスター外部（ブラウザ・外部クライアント）からApp-AまたはApp-Bへの通信。Gateway APIのHTTPRouteを経由してルーティングされる。
_Avoid_: 外部通信、インバウンドトラフィック、North-South traffic

### GitOps・マニフェスト管理

**App of Apps**:
ArgoCDのルートApplicationが、App-AとApp-BそれぞれのApplicationを子として管理するパターン。GitOps Repositoryのルートに親Applicationのマニフェストを置く。Golden Path完成後にApplicationSetへ移行する予定の暫定パターン。
_Avoid_: 親子Application

**ApplicationSet**:
generator(規約)からArgoCD Applicationを自動生成するArgoCDのリソース。App of Appsの後継として採用予定で、移行後は`manifests/`配下にディレクトリを置くだけでApplicationが生成される。Software Catalogとは役割が異なり(こちらはデプロイ、Catalogは台帳)、競合しない。
_Avoid_: AppSet、Application生成器

**Plain YAML**:
HelmやKustomizeを介さず生のKubernetesマニフェストをGitOps Repositoryに配置する管理スタイル。自作ワークロード(App-A・App-Bなど)のマニフェストにのみ適用され、Platform Componentには適用しない。学習フェーズで採用し、環境差分が必要になった時点でKustomizeに移行する。
_Avoid_: raw manifest、素のYAML

**Platform Component**:
アプリケーションではなくクラスターの機能を提供する、公式イメージをそのまま動かすサードパーティ製コンポーネント(Envoy Gateway・Argo Rollouts・KEDA)。公式HelmチャートをソースとするArgoCD ApplicationとしてApp of Appsに登録して導入する。自作イメージをビルドするBackstageは含まない(自作ワークロード扱い)。
_Avoid_: ミドルウェア、インフラコンポーネント、アドオン

### 開発者ポータル

**Backstage**:
開発者ポータル。Software CatalogとGolden Pathを提供する。クラスターへのデプロイ自体は行わず、Day 0(サービスの誕生)と可視化を担当する。デプロイ以降はArgoCDの責務。kindクラスター内のワークロードとして稼働させる。
_Avoid_: ポータル、IDP(曖昧なため)

**Golden Path**:
新サービスをクラスターに追加するための標準化された手順一式。Software Templateとして実装され、実行するとGitOps Repositoryに新サービスのマニフェスト一式(Rollout・Service・HTTPRoute・ScaledObject・Namespace・ArgoCD Application・catalog-info)がPull Requestとして提案される。マージするとApp of Apps経由で自動デプロイされ、Software Catalogにも自動発見で登録される。mainへの直接pushは行わない。
_Avoid_: ゴールデンパステンプレート、雛形(単体では曖昧)

**Software Template**:
Golden Pathの実装体。Backstageのscaffolderが実行する雛形定義。
_Avoid_: スキャフォールド、テンプレート(単体では曖昧)

**Software Catalog**:
クラスターで稼働するサービスの台帳。App-A・App-BをComponentとして登録し、Backstage上で一覧・参照する。
_Avoid_: サービス一覧、カタログ(単体では曖昧)

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
> **Domain expert**: Internal Trafficだから、`http://app-b.app-b.svc.cluster.local` を使って。Cloud Runのときみたいにインターネット経由じゃなくて、Cluster-internal DNSで直接届く。
>
> **Dev**: じゃあ外からApp-Aを叩くときは？
>
> **Domain expert**: それはExternal Traffic。HTTPRouteにホスト名とパスを定義してEnvoy Gatewayに流す。GitOps Repositoryのマニフェストに書いてArgoCDに反映させる流れ。
>
> **Dev**: RolloutとDeploymentって何が違う？
>
> **Domain expert**: RolloutはArgo Rolloutsのリソース。KubernetesのDeploymentをほぼ置き換える形で使う。今はBlue-Greenで入門して、本番GKEではCanaryに切り替える予定。
