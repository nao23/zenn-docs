---
title: pytest-splitによるテスト分割実行
type: tech
topics: ["python", "pytest", "githubactions"]
emoji: 🐍
published: true
---

Pythonのテストフレームワークとして [pytest](https://docs.pytest.org/en/8.0.x/index.html) が使いやすく機能も豊富で気に入っています。

テストが増えるに従って、全てのテストを実行するのにかかる時間は長くなっていきます。
コードの変更に対してCIでテストの通過を頻繁にチェックするような状況においては、テスト実行時間の増加が開発速度の低下につながります。

テストがある程度の規模になると、この問題に対してテストを分割してそれぞれ並列に実行するという方法を取ることがあるかと思いますが、今回はそういった分割実行を行うための [pytest-split](https://github.com/jerry-git/pytest-split) という pytestプラグインが便利だったので簡単に使い方を紹介したいと思います。

# pytest-split

pytest-splitはpytestのプラグインで、テストを実行時間に基づいて分割し実行する機能を提供してくれます。
https://github.com/jerry-git/pytest-split

テストを分割して実行しようと思った時、まずディレクトリ単位で分割して実行するアプローチを取ることがありました。ただ、このアプローチではディレクトリ毎に実行時間の偏りがあると並列に実行しても一番遅い部分に律速されるため、全体としてあまり高速化されないという問題にぶつかることがありました。

pytest-splitを使うことでディレクトリ構造に依存せず実行時間に基づいた分割が可能になります。

使い方はとても簡単で、まず `--store-durations` というオプションをつけて全部のテストを実行し、テストの実行時間を記録したファイルを生成します。

```shell
$ pytest --store-durations
```

その後、`--splits` オプションでいくつのグループに分割するかと `--group` オプションで分割グループの何番目を実行するかを指定します。

```shell
$ pytest --splits 3 --group 1
```

この場合、テスト実行時間が同一になるように3つのグループに分割し、そのうちの1番目のグループのテストを実行します。

詳しい使い方については以下のドキュメントをご参照ください。
https://jerry-git.github.io/pytest-split/

# GitHub Actions Workflow

ここでは、GitHub Actions上で実行するためのワークフローの例を記載します。

前述の通り、pytest-splitではテスト実行時間を記録したファイルを利用するため、そのファイルの管理方法を検討する必要があります。最初はリポジトリの管理下に入れ手動で定期的に更新するというのでもよいですが、手動での管理は手間ですし忘れ去られる可能性があるため、今回はGitHub Actions上から実行時間ファイルを生成し、artifactsに保存して利用する構成を考えました。

```yaml
name: Test

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    if: github.ref != 'refs/heads/release'
    strategy:
      matrix:
        group: [1, 2, 3, 4]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: 3.11

      - name: Install dependencies
        run: |
          curl -sSL https://install.python-poetry.org | POETRY_VERSION=1.8.1 python3 -
          poetry config virtualenvs.create false && poetry install

      - name: Download duration file
        env:
          GITHUB_ARTIFACT_API_ENDPOINT: https://api.github.com/repos/{owner}/{repos}/actions/artifacts
        run: |
          LATEST_ARTIFACT=$(curl -s -H "Authorization: token ${{ github.token }}" ${GITHUB_ARTIFACT_API_ENDPOINT}?name=test_durations | jq '.artifacts[0]')
          echo $LATEST_ARTIFACT
          DOWNLOAD_URL=$(echo $LATEST_ARTIFACT| jq -r '.archive_download_url')
          curl -s -H "Authorization: token ${{ github.token }}" -L $DOWNLOAD_URL -o artifact.zip
          unzip artifact.zip

      - name: Test
        run: |
          poetry run pytest --splits 4 --group ${{ matrix.group }} --durations-path=.test_durations

  test-and-store-duration-file:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/release'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: 3.11

      - name: Install dependencies
        run: |
          curl -sSL https://install.python-poetry.org | POETRY_VERSION=1.8.1 python3 -
          poetry config virtualenvs.create false && poetry install

      - name: Test
        run: |
          poetry run pytest --store-durations --durations-path=.test_durations

      - name: Upload duration file
        uses: actions/upload-artifact@v4
        with:
          name: test_durations
          path: .test_durations
```

このワークフローは pushイベントをトリガーに起動し、以下を行います

- releaseブランチ以外の場合:
  - 4並列で実行
    - Checkout
    - Pythonのセットアップ
    - 依存パッケージのインストール
    - artifactsから最新の実行時間ファイルをダウンロード
    - ダウンロードした実行時間ファイルを利用して4つのグループにテストを分割し、そのうちの1つを実行

- releaseブランチの場合:
  - Checkout
  - Pythonのセットアップ
  - 依存パッケージのインストール
  - 全てのテストを実行し、実行時間ファイルを生成
  - 実行時間ファイルをartifactsにアップロード

# まとめ

Pythonのテストで分割実行を行うためのpytest-splitというツールについて紹介しました。
シンプルで使いやすく便利かなと思います。

実行時間ファイルの管理について色々やり方があると思いますが、今回はGitHub Actions artifact上に保存して利用する構成を紹介しました。

何かの参考になれば幸いです。
