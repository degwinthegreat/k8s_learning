# ローカルkind環境でもGateway APIを採用する

本番GKEではGateway API + Google Cloud Load Balancerの統合を使う予定のため、ローカルのkind環境でもIngress-NGINXではなくGateway APIを採用する。実装にはkindとの相性が良いEnvoy Gatewayを使う。

ローカルではIngress-NGINXの方がセットアップが簡単だが、Ingress APIとGateway APIはリソースの書き方（`GatewayClass`・`Gateway`・`HTTPRoute`）が根本的に異なる。ローカルでGateway APIに慣れることで、本番GKEへの移行時に知識がそのまま活きる。

## Considered Options

- **Ingress-NGINX**: kindの公式ドキュメントに手順があり最もセットアップが簡単。ただしIngress APIの知識は本番GKE構成に転用できない。
- **Envoy Gateway（採用）**: kindとの相性が良く、Gateway API仕様のリソースをそのまま学べる。本番はGKE独自のGateway Controllerになるが、HTTPRoute等のマニフェストの書き方は共通。
