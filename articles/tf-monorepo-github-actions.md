---
title: GitHub Actions + Terraformモノレポ環境で変更ファイルに応じてCIを実行する
type: tech
topics: ["githubactions", "terraform"]
emoji: 🐈
published: false
---


Terraform モノレポにおけるCIでは、PR作成/更新時などに変更したファイルに応じて対象のディレクトリでのみterraform planを実行させたいというニーズがあるかと思います。 本記事では、それを実現するための幾つかのやり方についてご紹介したいと思います。
  
# 前提
ここでは、以下のようなリポジトリ構成となっていることを想定します。

```
.
├── account-a
│   ├── account-top.tf
│   ├── service-bar
│   │   └── bar.tf
│   └── service-foo
│       └── foo.tf
├── account-b
│   ├── account-top.tf
│   └── service-piyo
│       └── piyo.tf
└── modules
    └── common
        ├── main.tf
        └── variables.tf
```

* 複数のAWSアカウントを1つのリポジトリで管理している
  * `account-a/`, `account-b/` で異なるAWSアカウントを管理
* 同一アカウント内でもサービスや何らかの用途毎にディレクトリが分けられている
* `modules/` でローカルモジュールを管理しており、幾つかのディレクトリから利用されている

# やりたいこと
* PR作成/更新時に、変更したファイルに依存するディレクトリでterraform planを実行する
* 変更したファイルがあるディレクトリだけでなく、ローカルモジュールを更新したらそのモジュールを利用しているディレクトリでもterraform planを実行させたい

# 実現方法
## 1. GitHub Actionsのパスフィルター機能を利用するやり方

GitHub Actionsでは、`on.<push|pull_request>.paths` にパス名を記述すると、指定したパスにマッチしたファイルの変更があった場合のみワークフローを起動させることができます。この機能を利用し、以下の様なワークフローを各ディレクトリ毎に用意することで、やりたいことが簡単に実現できます。

```yaml
name: plan (account-a/service-foo)
on:
  pull_request:
    paths:
      - 'account-a/service-foo/*'
      - 'modules/common/*'

env:
  workdir: account-a/service-foo

jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.ASSUME_ROLE_FOR_ACCOUNT_A }}
          aws-region: ap-northeast-1

      - name: Setup terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.1.2
          terraform_wrapper: false

      - name: Init
        run: terraform init
        working-directory: ${{ env.workdir }}

      - name: Plan
        run: terraform plan
        working-directory: ${{ env.workdir }}
```

### 良い点
この方法の良い点はシンプルで実装が楽だという点です。管理しているインフラの規模が小さくディレクトリ数も少なければ、この方法で良さそうです。

### 問題点

インフラの規模が大きくなりディレクトリ数が増えていくと、それに伴いワークフローの数が増加するため、管理し辛くなっていくという問題があります。ジョブの中身はほとんど同じになるので、以下の様に[Composite Run Step Action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)を使って共通化したり、[Reusable Workflow](https://docs.github.com/ja/actions/using-workflows/reusing-workflows) を活用することで各ワークフロー自体の記述量は抑えられますが、「ワークフローの起動条件（パス一覧）」と「実行ディレクトリ」と「利用するSecret名」の組み合わせだけが異なるワークフローファイルが大量に生まれることになります。

```yaml
# Composite Run Step Actionを利用したワークフロー例
name: plan (account-a/service-foo)
on:
  pull_request:
    paths:
      - 'account-a/service-foo/*'
      - 'modules/common/*'

env:
  workdir: account-a/service-foo

jobs:
  plan:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Plan
        uses: ./.github/actions/plan
        with:
          role-to-assume: ${{ secrets.ASSUME_ROLE_FOR_ACCOUNT_A }}
          workdir: ${{ env.workdir }}
```

```yaml
# .github/actions/plan/action.yml
name: 'Plan'
description: 'exec terraform plan'
inputs:
  role-to-assume:
    required: true
    description: assume role arn
  workdir:
    description: working directory
    required: true

runs:
  using: "composite"
  steps:
    - name: configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        role-to-assume: ${{ inputs.role-to-assume }}
        aws-region: ap-northeast-1

    - name: Setup terraform
      uses: hashicorp/setup-terraform@v1
      with:
        terraform_version: 1.1.2
        terraform_wrapper: false

    - name: Init
      shell: bash
      run: terraform init
      working-directory: ${{ inputs.workdir }}

    - name: Plan
      shell: bash
      run: terraform plan
      working-directory: ${{ inputs.workdir }}
```

## 2. ジョブの中でパスフィルタリングを行い、動的に実行ディレクトリを決定するやり方

1つ目の方法はGitHub Actionsが提供するパスフィルターの機能を利用したものでしたが、ディレクトリ数の増加に伴いワークフロー数が増加してしまう問題がありました。それに対し、以下の様にパスフィルタリングをジョブの中で実行（つまり自前で実装）し、実行ディレクトリを動的に決定、[build matrix](https://docs.github.com/ja/actions/using-jobs/using-a-build-matrix-for-your-jobs) として渡して利用させることで、単一のワークフローで同じことを実現することができます。

```yaml
name: plan
on:
  pull_request:

jobs:
  determine-workdir:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
      contents: read
    outputs:
      workdirs: ${{ steps.filter.outputs.workdirs }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: .github/path-filter.yml

      - name: filter
        id: filter
        run: |
          WORKDIRS=$(echo '${{ toJSON(steps.changes.outputs) }}' | jq '. | to_entries[] | select(.value == "true") | .key') 
          echo "::set-output name=workdirs::$(echo $WORKDIRS | jq -sc '.')"
  
  plan:
    needs: determine-workdir
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
    if: needs.determine-workdir.outputs.workdirs != '[]'
    strategy:
      matrix:
        workdir: ${{ fromJSON(needs.determine-workdir.outputs.workdirs) }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Set environment variables for each env
        run: |
          ENV=$(echo "${{ matrix.workdir }}" | cut -d '/' -f1)
          yq "with_entries(select(.key == \"$ENV\")) | .$ENV" .github/secret-mapping.yml -o props | tr -d " " >> $GITHUB_ENV

      - name: configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets[env.SECRET_NAME_FOR_ASSUME_ROLE] }}
          aws-region: ap-northeast-1

      - name: Setup terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.1.2
          terraform_wrapper: false

      - name: Init
        run: terraform init
        working-directory: ${{ matrix.workdir }}

      - name: Plan
        run: terraform plan
        working-directory: ${{ matrix.workdir }}
```

```yaml
# .github/path-filter.yml
account-a:
  - 'account-a/*'

account-a/service-foo:
  - 'account-a/service-foo/*'
  - 'modules/common/*'

account-a/service-bar:
  - 'account-a/service-var/*'

account-b:
  - 'account-b/*'

account-b/service-foo:
  - 'account-a/service-foo/*'
  - 'modules/common/*'

account-b/service-bar:
  - 'account-a/service-var/*'
```

```yaml
# .github/secret-mapping.yml
account-a:
  SECRET_NAME_FOR_ASSUME_ROLE: ASSUME_ROLE_FOR_ACCOUNT_A

account-b:
  SECRET_NAME_FOR_ASSUME_ROLE: ASSUME_ROLE_FOR_ACCOUNT_B
```


### 解説
ワークフローの内容が若干難しくなるので簡単に解説を行います。

* **determine-workdir** と **plan** という2つのジョブで構成されます
  * determine-workdir: terraform planを実行するディレクトリの一覧を生成するジョブ
  * plan: 実際にterraform planを実行するジョブ

* determine-workdir ジョブの流れ
  * リポジトリのcheckout
  * [dorny/paths-filter](https://github.com/dorny/paths-filter) を利用し、パスフィルタリングを実行する
    * ジョブレベルでパスフィルタリングの機能を提供してくれるアクション
    * inputとしてフィルターパターンをYAML形式で与える　
      * インラインで渡すこともできるが、別ファイルに切り出してファイル名を指定する形も可能（今回の例は後者を採用）
    * フィルターパターンの各キー毎に対応するoutputが出力され、マッチしたキーは `true` が設定される
    * 実行ディレクトリパスをキーとして記載したフィルターパターンを指定
  * 前段のステップのoutputから、バリューが `true` となっているキーの一覧を取得し、ジョブのoutputとして出力
    * 変更したファイルに対応する実行ディレクトリパスの一覧が生成される

* plan ジョブの流れ
  * determine-workdir ジョブのoutputをmatrixに指定し、ディレクトリ毎に並列実行
  * リポジトリのcheckout
  * 実行ディレクトリに対応するSecret名を取得
    * AWSアカウント毎に利用するSecretを切り替える必要がある
    * `.github/secret-mapping.yml` に、AWSアカウントと対応するディレクトリパスとSecret名のマッピングを記載
  * AWSクレデンシャルを取得
  * Terraformのインストール
  * terraform initを実行
  * terraform planを実行

### 良い点
この方法の良い点はワークフローが単一になることです。[dorny/paths-filter](https://github.com/dorny/paths-filter) では、フィルターパターンを別ファイルに切り出すことができるため、ディレクトリが増えた場合でもワークフローを修正することなく、フィルターパターンファイルの修正のみで対応することが可能となります。

### 問題点
1つ目の方法に比べて、ワークフローの内容が複雑になるという問題があります。ディレクトリを追加した際の対応手順などをコメントやREADMEなどに記載しておく必要がありそうです。

## まとめ
GitHub ActionsでTerraformモノレポのCIを実施する際に、変更したファイルに応じて対象のディレクトリでterraform planを実行させる方法について紹介しました。どちらの方法も一長一短あるので、組織や管理するインフラの規模に応じてベストな方法を選ぶのが良いかと思います。何かの参考になれば幸いです。
