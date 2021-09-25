---
title: "AirFlowのDAG処理通知をTeamsに通知してみた"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["airflow", "microsoftteams", "bot"]
published: false
---

# Airflow での処理通知を Slack でなく Teams に送りたい

ドキュメントを見るに、Slack 通知に関してはライブラリが用意されていそうだが、 Teams はいつも通り冷遇されている。
カスタム関数を用意して、Teams に処理通知を飛ばせるようにしてみた。(要は任意の Webhook URL に飛ばせるようにしてみた)
https://www.astronomer.io/guides/error-notifications-in-airflow

## 処理完了時に通知を飛ばす dag.py

以下が完成したもの。
`custom_success_function()`は処理成功時にキックされ、`custom_failure_function()`は処理失敗時にキックされる。
カスタム関数内での成功/失敗時の同様処理を別途関数で切り出したかったのだが、処理を関数内に収める必要があり冗長になってしまった。
Operator 部分は適宜変更して流用してください。

```python
from datetime import datetime, timedelta
import json
import requests

from airflow import DAG
from airflow.utils.dates import days_ago

from airflow.operators.dummy_operator import DummyOperator

JOB_NAME = 'Airflow Job'
TEAMS_WEBHOOK_URL='https://xxx' # 処理通知飛ばし先に当たるWebhookURL
AIRFLOW_DOMAIN = 'http://airflow.xxx.com/' # localhost:8080でlog urlが出力されるので、Airflowをホストしているドメインを指定して変換する
# https://airflow.apache.org/docs/apache-airflow/stable/macros-ref
# https://www.astronomer.io/guides/error-notifications-in-airflow

def custom_failure_function(context):
    dag_run = context.get('dag_run')
    task_instances = dag_run.get_task_instances()
    # payload = get_teams_payload(context, success=False)

    success = False

    title = f'"{JOB_NAME}" has been completed successfully.' if success else f'"{JOB_NAME}" has been failed.'
    image = 'https://www.freeiconspng.com/uploads/success-icon-19.png' if success else 'https://www.freeiconspng.com/uploads/failure-icon-2.png'
    log_url = context.get("task_instance").log_url
    log_url_after_domain = log_url.replace('http://localhost:8080/','')

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "activitySubtitle": "Airflow job notification",
            "activityImage": image,
            "facts": [
                {'name': 'DAG', 'value': f'{context.get("task_instance").dag_id}'},
                {'name': 'Task', 'value': f'{context.get("task_instance").task_id}'},
                {'name': 'Execution Time', 'value': f'{context.get("execution_date").in_timezone("Asia/Tokyo").format("YYYY/MM/DD-hh:mm:ssZ")}'},
                {'name': 'Next Execution Time', 'value': f'{context.get("next_execution_date").in_timezone("Asia/Tokyo").format("YYYY/MM/DD-hh:mm:ssZ")}'},
            ],
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": "Airflow Log URL",
                "targets": [{
                    "os": "default",
                    "uri": f'{AIRFLOW_DOMAIN}{log_url_after_domain}'
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

def custom_success_function(context):
    dag_run = context.get('dag_run')
    task_instances = dag_run.get_task_instances()
    # payload = get_teams_payload(context, success=False)

    success = True

    title = f'"{JOB_NAME}" has been completed successfully.' if success else f'"{JOB_NAME}" has been failed.'
    image = 'https://www.freeiconspng.com/uploads/success-icon-19.png' if success else 'https://www.freeiconspng.com/uploads/failure-icon-2.png'
    log_url = context.get("task_instance").log_url
    log_url_after_domain = log_url.replace('http://localhost:8080/','')

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "activitySubtitle": "Airflow job notification",
            "activityImage": image,
            "facts": [
                {'name': 'DAG', 'value': f'{context.get("task_instance").dag_id}'},
                {'name': 'Task', 'value': f'{context.get("task_instance").task_id}'},
                {'name': 'Execution Time', 'value': f'{context.get("execution_date").in_timezone("Asia/Tokyo").format("YYYY/MM/DD-hh:mm:ssZ")}'},
                {'name': 'Next Execution Time', 'value': f'{context.get("next_execution_date").in_timezone("Asia/Tokyo").format("YYYY/MM/DD-hh:mm:ssZ")}'},

            ],
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": "Airflow Log URL",
                "targets": [{
                    "os": "default",
                    "uri": f'{AIRFLOW_DOMAIN}{log_url_after_domain}'
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

# custom_function内で完結する必要があるので関数での切り出しはできなさそう
# def get_teams_payload(context, success):
#     title = f'{JOB_NAME} has been completed successfully.' if success else f'{JOB_NAME} has been failed.'
#     image = 'https://www.freeiconspng.com/uploads/success-icon-19.png' if success else 'https://www.freeiconspng.com/uploads/failure-icon-2.png'
#     # payload =
#     return payload

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2021, 9, 1), # start_date + schedule_intervalが初回実行日付になるので注意
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(seconds=10),
    'on_failure_callback': custom_failure_function,
    'on_success_callback': custom_success_function,
}

with DAG(
    'test-job', # ジョブ名
    default_args=default_args,
    description='Notification test.(webhook for Teams)',
    schedule_interval='@monthly', # ジョブ実行間隔
) as dag:

    task1 = DummyOperator(
        task_id='task1',
    )
```

## 処理通知用アイコンの悩み

こういう時にイイ感じのアイコンの URL の取得先を知らず、なんだかあまりしっくりこないものしか選べない。イイ感じのアイコン URL リストを今後ストックしていきたい。

現状使っているもの：
成功アイコン
https://www.freeiconspng.com/uploads/success-icon-19.png

失敗アイコン
https://www.freeiconspng.com/uploads/failure-icon-2.png
