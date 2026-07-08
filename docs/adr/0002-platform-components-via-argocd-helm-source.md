# Platform ComponentはArgoCDのHelmソースで導入する

Envoy Gateway・Argo Rollouts・KEDAなどのPlatform Componentを導入するにあたり、公式HelmチャートをソースとするArgoCD ApplicationとしてApp of Appsに登録する方式を採用する。リポジトリにはApplication定義だけを置き、レンダリングはArgoCDに任せる。

「Plain YAML」方針はあくまで自作ワークロード(App-A/App-Bなど)のマニフェストに適用されるもので、サードパーティ製コンポーネントには適用しない。手動インストール(ブートストラップ層扱い)はクラスター再作成のたびに手順書が必要になり「全てをGitから」の学習テーマから外れるため、またレンダリング済みYAMLのコミットは数千行のファイル管理とバージョン更新の苦痛を伴うため、いずれも退けた。

## Considered Options

- **ArgoCDのHelmソース(採用)**: GitOps一貫性を保ちつつHelmを直接扱わない。リポジトリに置くのはApplication定義のみ。
- **手動ブートストラップ**: ArgoCD本体と同じ扱い。シンプルだが再現性がGitに残らない。
- **レンダリング済みYAMLをコミット**: Plain YAML方針の純粋な適用だが、管理コストが高すぎる。
