---
title: Self-Hosting Renovate on GitHub ActionsでTerraform Providerを自動更新する
type: tech
topics: ["renovate", "githubactions", "terraform"]
emoji: 🛠
published: false
---

Terraformのコードを管理するリポジトリでは、Terraform本体に加えて利用するProviderのバージョンなどを継続的にアップデートしていく必要があります。こういった依存関係の更新情報を日々チェックして手でコードを修正していくのはとても大変です。この作業を自動化してくれるツールは幾つかありますが、本記事ではRenovateのCLIをGitHub Actions上で定期実行し更新するやり方についてご紹介したいと思います。

# Renovateとは

今回利用するRenovateは、プロジェクトの依存関係を解析し、依存するパッケージにアップデートがあればコードを修正しPRの作成まで自動化することが行うことができるツールです。

https://www.mend.io/free-developer-tools/renovate/
https://github.com/renovatebot/renovate

Renovateには[GitHub App](https://github.com/apps/renovate) や [CLIツール](https://www.npmjs.com/package/renovate) や [Docker Image](https://hub.docker.com/r/renovate/renovate/) など幾つかの利用方式があります。

GitHub Appでは導入から定期実行まで一通り自動でやってくれるのに対し、CLIツールやDocker Imageを利用する いわゆる **Self-Hosting 形式** の場合は、別のCIサービスなどと組み合わせて導入から定期実行の仕組みまで自分達で用意する必要があります。

気軽に利用するにはGitHub Appが便利なのですが、セキュリティの観点から利用できる機能が制限されており、一部の機能はSelf-Hosting形式じゃないと使えないという特徴があります。


# Self-Hosting Renovate on GitHub Actions
今回はCLIツールとしてのRenovateを利用し、GitHub Actions上で定期実行させることで更新を自動化する方法についてご紹介したいと思います。

最初はGitHub Appの利用を試みたのですが、Renovateが生成する `.terraform.lock.hcl` ファイルを使って手元で `terraform init` を実行すると毎回差分が発生する問題に遭遇してしまい、それを解決できそうな機能がSelf-Hosting形式でないと使えなかったのでやむなく断念しました。

## Renovate 設定
まず、Renovateの設定ファイルとして以下を用意します。

```json:.github/renovate.json
{
  "extends": [
    "config:base"
  ],
  "labels": ["dependencies"],
  "packageRules": [
    {
      "packagePatterns": ["*"],
      "excludePackageNames": ["aws"],
      "enabled": false
    },
    {
      "matchPackageNames": ["aws"],
      "postUpgradeTasks": {
        "commands": [
          "/bin/bash -c \"[[ -f {{{packageFileDir}}}/.terraform.lock.hcl ]] && cd {{{packageFileDir}}} && rm .terraform.lock.hcl && terraform get -no-color -update && terraform providers lock -platform=darwin_amd64 -platform=darwin_arm64 -platform=linux_amd64\""
        ],
        "fileFilters": ["**/.terraform.lock.hcl"]
      }
    }
  ]
}
```

### 各設定項目の意味

設定内容について簡単に解説したいと思います。
各項目の詳細については[公式ドキュメント](https://docs.renovatebot.com/configuration-options/)をご参照ください。

* `extends`
    * Renovateでは設定を共有する仕組みがあり、あらかじめ用意されている設定に加えてGitHubなどで公開されているものも利用することができます。
    * ここでは、あらかじめ用意されている `config:base` を利用しています。
* `onbarding`
    * オンボーディングPRを作成するかを指定
    * 今回は不要なので `false`
* `labels`
    * PRに付与するラベル
* `packageRules`
    * ここに各パッケージに対する設定を記載していきます
    * 1つめのルール
        * `aws` (terraform-aws-provider) パッケージだけを有効にしています
        * デフォルトで全てのパッケージが有効になっているのですが、ひとまず小さく始めたかったので `aws` パッケージだけを有効にしそれ以外を無効にしています。
        * `packagePatterns` で指定したパッケージから `excludePackageNames` で指定したパッケージを除いたものを無効化 (`enabled: false`)（ややこしい）
    * 2つめのルール
        * **`.terraform.lock.hcl` ファイルを生成し直してます** 
        * [PostUpgradeTask](https://docs.renovatebot.com/configuration-options/#postupgradetasks) という機能を利用 (※ Self-Hosting版でのみ利用可能)
            * Renovateが依存関係を更新し修正したコードをcommitする前に指定したコマンドを実行させることができる
        * renovateは `.terraform.lock.hcl` も更新してくれるのですが、手元でterraform initを実行すると差分が発生してしまう問題に遭遇し、コマンドを叩いてこちらが望む形式で生成し直すという力技で解決しています。
        * `PostUpgradeTask`の機能がSelf-Hosting形式でしか利用できないものだったため、GitHub Appの利用を断念しました。


## GitHub Actions Workflow
続いて、以下のGitHub Actions Workflowを用意します。
```yaml:.github/workflows/renovate.yaml
name: renovate
on:
  schedule:
    - cron: '0 0 * * 5'
  workflow_dispatch:

jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Setup terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.1.2
          terraform_wrapper: false

      - uses: actions/setup-node@v1
        with:
          node-version: "16"

      - name: Setup renovate
        run: npm install -g renovate@32.89.0

      - name: Run renovate
        env:
          RENOVATE_REPOSITORIES: <ユーザ名>/<リポジトリ名>
          RENOVATE_TOKEN: ${{ secrets.PAT }}
          RENOVATE_ALLOW_POST_UPGRADE_COMMAND_TEMPLATING: true
          RENOVATE_ALLOWED_POST_UPGRADE_COMMANDS: ".*"
        run: renovate
```

### 動作概要
シンプルなWorkflowですが、動作について簡単に解説します。

* 毎週金曜の9時に起動 (手動起動も可能)
* Terraformのセットアップ
* Nodeのセットアップ
* Renovateのインストール
* Renovateの実行
    * 環境変数で以下を設定
        * `RENOVATE_REPOSITORIES`: 管理するリポジトリを指定
        * `RENOVATE_TOKEN`: PRを作成するためのPersonal Access Tokenを指定
        * `RENOVATE_ALLOW_POST_UPGRADE_COMMAND_TEMPLATING`: 
            * PostUpgradeTaskで実行するコマンドでテンプレート変数を利用可能に
        * `RENOVATE_ALLOWED_POST_UPGRADE_COMMANDS`:
            * PostUpgradeTaskで実行できるコマンドを指定（正規表現で指定可能）


以上の設定で、毎週金曜の9時にRenovateが実行され更新があれば自動でPRが生成されるようになりました。

# その他のツール達
今回はRenovateというツールを利用しましたが、同様のことを実現するためのツールは幾つかあり、それぞれ簡単にご紹介したいと思います。

## Dependabot
こちらもRenovateと同様にプロジェクトの依存関係を解析し更新を自動化できるツールです。GitHubと統合されているので導入が簡単なのが特徴ですね。
Terraform Providerの更新もサポートしており、こちらの場合はlockファイルに差分が出る問題もなかったのですが、PRをグルーピングする機能がないことが惜しい点でした。
Terraformの場合はディレクトリ毎にProviderのバージョン指定があり、ディレクトリの数だけPRが作られてしまうことになるため、大量のPRを自動マージする仕組み/運用を別途検討する必要があり、一旦見送ってRenovateを利用することにしました。

https://github.com/dependabot

## tfupdate

こちらはTerraform専用のツールで、Terraformコードに記載されているProviderなどのバージョン指定をまとめて書き換えてくれるものになります。

以前はこちらのツールを利用しており、tfupdateをGitHub Actions上で実行しgitコマンドを叩いてPRを生成する仕組みを用意していました。

https://github.com/minamijoyo/tfupdate

# まとめ
今回はSelf-Hosting形式のRenovateをGitHub Actions上で定期実行して、Terraform Providerの更新を自動化する方法についてご紹介しました。

Renovateを使い始めてまだ日が浅く、設定項目などまだ理解しきれてないところが多いのですが、ひとまず動くところまでできたので記事にまとめた次第でした。

何かの参考になれば幸いです。
