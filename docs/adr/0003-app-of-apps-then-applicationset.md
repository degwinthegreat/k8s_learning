# App of AppsでGolden Pathを完成させてからApplicationSetへ移行する

Golden PathのフェーズではSoftware TemplateがArgoCD Application YAMLを明示的に生成するApp of Apps構成を維持し、Golden Pathが一巡動作した後にgit directory generatorによるApplicationSetへ移行する。

最初からApplicationSetにすればテンプレートは痩せるが、(1) すでに動作確認済みのApp of Appsを学習途中で壊さない、(2) 両パターンを実地で比較してから乗り換える方が学習価値が高い、という理由で段階移行を選んだ。移行後はテンプレートからApplication生成ステップが消え、PRはマニフェストディレクトリの追加だけになる。

## Considered Options

- **最初からApplicationSet**: テンプレートが最初から薄い。ただしApp of Appsとの比較体験が得られない。
- **App of Apps維持**: Application定義が全てGitに明示的に残る。新サービスごとに定型YAMLが増え続ける。
- **段階移行(採用)**: テンプレートのApplication生成部分を作って後で消す二度手間を、両パターンの実地比較というリターンで払う。
