# Golden PathはGitOps RepositoryへのPRとして生成する

Software Template(templates/workload/)の出力は、このリポジトリ(モノレポ)への Pull Request として生成する(`publish:github:pull-request`)。マージがデプロイとカタログ登録の唯一のトリガーであり、Scaffolderはクラスターにもmainブランチにも直接触れない。

「クラスターの状態は全てGitのmainが決める」というGitOps原則をScaffolderにも貫くための選択。テンプレートが直接mainにpushしたりクラスターにapplyしたりすると、レビューを迂回する第2の変更経路ができてしまう。カタログ登録もPRに含まれる `catalog/<name>.yaml` をワイルドカードlocationが検出する方式で、Scaffolderによる明示的な `catalog:register` を使わない(マージ前に登録される不整合を防ぐ)。

## Considered Options

- **サービスごとに新規リポジトリを作成**: 本番で最も一般的。ただし学習環境では1サービス=whoami数十行にリポジトリ・ArgoCD認証・discovery設定が毎回付いてきて、本質(Golden Pathの体験)から遠ざかる。
- **mainへ直接push + catalog:register**: 即時反映で体験は速いが、レビューを迂回する変更経路になり、GitOpsの学習目的と矛盾する。
- **モノレポへのPR(採用)**: マージ=デプロイ+カタログ登録の単一経路。個人アカウントでGitHub org discoveryが使えない制約も、モノレポのワイルドカードlocation(`catalog/*.yaml`)で回避できる。
