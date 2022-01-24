---
title: "GitHubActions上でのcopilotによるCD環境を整理してみる"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cicd", "copilot", "githubactions"]
published: true
---

# はじめに

AWS copilot は便利。個人的には Docker でのスケジュールジョブ開発の AWS デプロイは copilt 一択。
ローカル端末だけでなく GitHub Actions 上でも動いて欲しくなったので、整備して動かしてみた。
便利な copilot はこちら
https://aws.amazon.com/jp/blogs/news/introducing-aws-copilot/

## 前提条件

- {}で囲んでいるところは自身の環境に沿って適宜読み替える
- copilot の ScheduledJobs が対象
  - 他のコンテナアプリケーションは動作未確認
- copilot deploy が既にローカル端末等で実行されて環境が作成されており、デプロイしたい環境用の IAM ロールが作成されている
  - copilot のアプリケーションおよび環境ごとに作成される管理ロール
  - {app}-{env}-EnvManagerRole

## IAM ID プロバイダー の用意

今回は`copilot-deploy`という名前のロールを作成。
以下の記事を参照にしながら、IAM ID プロバイダーとして作成。
マネコンからつくるとハマるところがあるので、以下をちゃんと読んでから作る。
https://dev.classmethod.jp/articles/create-iam-id-provider-for-github-actions-with-management-console/

### 必要なポリシーの付与

#### copilot 管理ロールの AssumeRole ポリシー

前提条件で書いた通り、既に`copilot deploy`が実行されて環境作成が成功していると、いくつか IAM ロールが作成されている。
copilot が作成した IAM ロールの中で、`{app}-{env}-EnvManagerRole`という名前の環境管理用ロールが必要になる。
前章で作成したロールに上記環境管理用ロールを AssumeRole できるようなポリシーを作成する必要がある。
以下で作成する。名前は `AssumeRole-{app}-{env}-EnvManagerRole`とでもしておく。

```
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::{account_id}:role/{app}-{env}-EnvManagerRole"
    }
}
```

#### その他必要なポリシーの付与

必要最低限にしたいものの、調べてもわからなかったので以下で設定。

- AmazonEC2ContainerRegistryFullAccess
- AmazonS3FullAccess
- CloudWatchFullAccess
- AmazonSSMFullAccess
- AWSCloudFormationFullAccess

#### GitHub レポジトリの secrets として設定

GitHubActions を設定したいレポジトリに web で行き,Settings から Repository secrets に登録
今回は `COPILOT_DEPLOY_IAM_ROLE_ARN` という名前で登録。

## GitHub Actions ワークフローの用意

develop への push がトリガーのワークフローとして以下で作成。
（ここについての{}は前提条件で描いていることは気にせずそのまま書けば良い）

```
name: copilot-job-deploy

on:
  push:
    branches:
      - develop
  workflow_dispatch:

permissions:
  id-token: write
  contents: read
jobs:
  aws-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.COPILOT_DEPLOY_IAM_ROLE_ARN }}
          aws-region: ap-northeast-1
      - name: check identity 👷‍♂️
        run: aws sts get-caller-identity
      - name: Setup AWS Copilot👨🏻‍💻
        run: |
            mkdir -p $GITHUB_WORKSPACE/bin
            # download copilot
            curl -Lo copilot-linux https://github.com/aws/copilot-cli/releases/download/v1.13.0/copilot-linux && \
            # make copilot bin executable
            chmod +x copilot-linux && \
            # move to path
            mv copilot-linux $GITHUB_WORKSPACE/bin/copilot && \
            # add to PATH
            echo "$GITHUB_WORKSPACE/bin" >> $GITHUB_PATH
      - name: deploy 🐥
        run: copilot job deploy --env prod
```

### さいごに

CloudTrail をうまく使って最小権限ポリシー生成できるようになっているらしいのでやってみたい。ただ、CloudTrail そんなに触ったことなく証跡設定とやらがわからなくてサクッと出来なかった。落ち着いたらやってみよう。
https://dev.classmethod.jp/articles/iam-access-analyzer-easier-implement-least-privilege-permissions-generating-iam-policies-access-activity/
