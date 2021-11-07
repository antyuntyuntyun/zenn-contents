---
title: "LambdaでAWSコストをTeamsに通知"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Lambda", "Bot", "AWS", "Microsoft Teams"]
published: true
---

## はじめに

Lambda で AWS コストを通知させるようにしたので備忘録。
Slack 通知は情報が溢れているけど Teams 通知している人をネットワークの大海で見かけませんでした。
aws cli 使える環境じゃないという制約があったのもあり、sam は使っていません。

### Teams で incoming-webhook の用意

公式ドキュメント通りにやります。WebhookURL を控えるのを忘れないようにしましょう
https://docs.microsoft.com/ja-jp/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook

### Lambda にアタッチする IAM Role の作成

マネコンから lambda のページに行き、とりあえず関数を作成します。
関数の名前は`aws_billing`等の任意の名前で作成します。
デフォルト実行ロールが作成されるので、作成されたロールがコストを取得できるように, iam のページに行ってアタッチするポリシーを変更します。

まず、以下二つ AWS 管理のポリシーをアタッチします。

- AWSAccountUsageReportAccess
- AWSBillingReadOnlyAccess

次にポリシーを作成してアタッチします。
作成するポリシーは`Cost Explorer Service`の ReadOnly 権限を与えてください。
`CostExplorerRead`等の任意の名前で保存してアタッチします。

### Lambda コードの記述(python)

クラスメソッドさんの Slack 通知用記事のコードをベースに書きました。
https://dev.classmethod.jp/articles/notify-slack-aws-billing/

Incoming-Webhook url は環境変数化して秘匿したかったのですが、時間がなかったので断念しました。
時間があるときにやります。
`TEAMS_WEBHOOK_URL`に前章で作成した、Teams の Incoming-Webhook URL を入力します。

```python
import os
import boto3
import json
import requests
from datetime import datetime, timedelta, date

# TODO: 環境変数化
# SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']
TEAMS_WEBHOOK_URL = "ここにWebhookURLを入力"


def lambda_handler(event, context) -> None:
    client = boto3.client('ce', region_name='ap-northeast-1')

    # 合計とサービス毎の請求額を取得する
    total_billing = get_total_billing(client)
    service_billings = get_service_billings(client)

    # Slack用のメッセージを作成して投げる
    (title, detail) = get_message(total_billing, service_billings)
    print(f'title: {title}')
    print(f'detail: {detail}')
    # post_slack(title, detail)
    post_teams(title, detail, service_billings)


def get_total_billing(client) -> dict:
    (start_date, end_date) = get_total_cost_date_range()

    # https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ce.html#CostExplorer.Client.get_cost_and_usage
    response = client.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='MONTHLY',
        Metrics=[
            'AmortizedCost'
        ]
    )
    return {
        'start': response['ResultsByTime'][0]['TimePeriod']['Start'],
        'end': response['ResultsByTime'][0]['TimePeriod']['End'],
        'billing': response['ResultsByTime'][0]['Total']['AmortizedCost']['Amount'],
    }


def get_service_billings(client) -> list:
    (start_date, end_date) = get_total_cost_date_range()

    # https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/ce.html#CostExplorer.Client.get_cost_and_usage
    response = client.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='MONTHLY',
        Metrics=[
            'AmortizedCost'
        ],
        GroupBy=[
            {
                'Type': 'DIMENSION',
                'Key': 'SERVICE'
            }
        ]
    )

    billings = []

    for item in response['ResultsByTime'][0]['Groups']:
        billings.append({
            'service_name': item['Keys'][0],
            'billing': item['Metrics']['AmortizedCost']['Amount']
        })
    return billings


def get_message(total_billing: dict, service_billings: list) -> (str, str):
    start = datetime.strptime(total_billing['start'], '%Y-%m-%d').strftime('%m/%d')

    # Endの日付は結果に含まないため、表示上は前日にしておく
    end_today = datetime.strptime(total_billing['end'], '%Y-%m-%d')
    end_yesterday = (end_today - timedelta(days=1)).strftime('%m/%d')

    total = round(float(total_billing['billing']), 2)

    title = f'{start}～{end_yesterday}の請求額は、{total:.2f} USDです。'

    details = []
    for item in service_billings:
        service_name = item['service_name']
        billing = round(float(item['billing']), 2)

        if billing == 0.0:
            # 請求無し（0.0 USD）の場合は、内訳を表示しない
            continue
        details.append(f'　・{service_name}: {billing:.2f} USD')

    return title, '\n'.join(details)


def post_slack(title: str, detail: str) -> None:
    # https://api.slack.com/incoming-webhooks
    # https://api.slack.com/docs/message-formatting
    # https://api.slack.com/docs/messages/builder
    payload = {
        'attachments': [
            {
                'color': '#36a64f',
                'pretext': title,
                'text': detail
            }
        ]
    }

    # http://requests-docs-ja.readthedocs.io/en/latest/user/quickstart/
    try:
        response = requests.post(SLACK_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)

def post_teams(title: str, detail: str, service_billings: list) -> None:
    # https://docs.microsoft.com/ja-jp/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL
    # payload = {
    #     'title': title,
    #     'text': detail
    # }

    facts = []
    for item in service_billings:
        service_name = item['service_name']
        billing = round(float(item['billing']), 2)

        if billing == 0.0:
            # 請求無し（0.0 USD）の場合は、内訳を表示しない
            continue
        dict_tmp = {'name': f'{billing:.2f} USD', 'value':service_name, 'billing':billing}
        facts.append(dict_tmp)

    facts_sorted_by_billing = sorted(facts, key=lambda x:x['billing'], reverse=True)

    # ソート用に保持していたbilling要素を削除
    for item in facts_sorted_by_billing:
        del item['billing']

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "activitySubtitle": "サービス別利用金額(金額降順)",
            "activityImage": "https://img.icons8.com/color/50/000000/amazon-web-services.png",
            "facts": facts_sorted_by_billing,
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": "Cost Management Console",
                "targets": [{
                    "os": "default",
                    "uri": "https://console.aws.amazon.com/cost-management/home?region=ap-northeast-1#/dashboard"
                }]
            }]
        }]
    }

    try:
        response = requests.post(TEAMS_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)

def get_total_cost_date_range() -> (str, str):
    start_date = get_begin_of_month()
    end_date = get_today()

    # get_cost_and_usage()のstartとendに同じ日付は指定不可のため、
    # 「今日が1日」なら、「先月1日から今月1日（今日）」までの範囲にする
    if start_date == end_date:
        end_of_month = datetime.strptime(start_date, '%Y-%m-%d') + timedelta(days=-1)
        begin_of_month = end_of_month.replace(day=1)
        return begin_of_month.date().isoformat(), end_date
    return start_date, end_date


def get_begin_of_month() -> str:
    return date.today().replace(day=1).isoformat()


def get_prev_day(prev: int) -> str:
    return (date.today() - timedelta(days=prev)).isoformat()


def get_today() -> str:
    return date.today().isoformat()
```

#### Lambda コードの変更点

クラメソさんとの記事のコードとの差異としては、`post_teams()`を追加してる部分になります。
サービス別の料金を降順表示するようにして投稿させています。
Slack のメッセージも色々カスタマイズできて面白いですが、Teams もある程度カスタマイズできます。
コネクタメッセージ仕様の公式ドキュメントは以下です。
https://docs.microsoft.com/ja-jp/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL

```python
def post_teams(title: str, detail: str, service_billings: list) -> None:
    # https://docs.microsoft.com/ja-jp/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL
    # payload = {
    #     'title': title,
    #     'text': detail
    # }

    facts = []
    for item in service_billings:
        service_name = item['service_name']
        billing = round(float(item['billing']), 2)

        if billing == 0.0:
            # 請求無し（0.0 USD）の場合は、内訳を表示しない
            continue
        dict_tmp = {'name': f'{billing:.2f} USD', 'value':service_name, 'billing':billing}
        facts.append(dict_tmp)

    facts_sorted_by_billing = sorted(facts, key=lambda x:x['billing'], reverse=True)

    # ソート用に保持していたbilling要素を削除
    for item in facts_sorted_by_billing:
        del item['billing']

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "activitySubtitle": "サービス別利用金額(金額降順)",
            "activityImage": "https://img.icons8.com/color/50/000000/amazon-web-services.png",
            "facts": facts_sorted_by_billing,
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": "Cost Management Console",
                "targets": [{
                    "os": "default",
                    "uri": "https://console.aws.amazon.com/cost-management/home?region=ap-northeast-1#/dashboard"
                }]
            }]
        }]
    }

    try:
        response = requests.post(TEAMS_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)
```

#### Teams 投稿時のアイコン

今回のコードでは投稿されるメッセージボックスの中に aws アイコンを含めるようにしているのですが、背景が透明なアイコンしか見つからなかったので、Teamas のダークモードで見るとなんだか微妙な感じになります。。。
背景白塗りの aws アイコンの url わかる方いらっしゃいましたら、コメントで教えていただけると非常に助かります。

### 定期実行させるためのトリガ設定

トリガーを追加から`EventBridge(Cloud Watch Events)`を選択して、cron で書きます。
今回は平日毎朝 10 時に通知させるために、`cron(0 1 ? * MON-FRI *)`としました。(UTC なのでマイナス 9 時間の値)

### できたもの

![](https://storage.googleapis.com/zenn-user-upload/69a8f3af75f1ac81208d9372.png)

## 終わりに

仕事に疲れたときに bot 作るのは良い息抜きになると思うので Teams を利用している皆さん是非お試しください。
