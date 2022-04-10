---
title: 'CodeCommitの通知をLambdaでTeamsに飛ばしてみた'
emoji: '🔖'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['Lambda', 'CodeCommit', 'Bot', 'AWS', 'Microsoft Teams']
published: true
---

## はじめに

以下の記事に触発され Lambda で python で CodeCommit に通知を飛ばすように実装しました。zip でうｐが面倒なので、python3.7 で実装しました。
通知のための設定等細かな点にについては以下の記事内容をご確認ください。
https://qiita.com/aocm/items/8eb22939791691c19cde

## まずは Lambda で event の中身の確認

`lambda_handler()`に渡される event の中身は object で渡されるもので、どのような形で来るかわかりません。
CodeCommit の PR 関連トリガーで event にどういう形の情報が飛ばされるかをまず確認しました。

### Sns の event 確認コード

Sns から送信される event を以下のコードで確認しました。
`event['Records'][0]['Sns']['Message']`は unicode 文字列として認識されてしまい、そのままだと dict としてうまく読み込めないので、`json.loads()`を挟んでいます。
CloudWatchLogs で複数行に改行されたログで出力にならず 1 レコードで出力されるように logger をちゃんと設定します。
こちらを lambda で deploy したうえで、通知内容を確認したい処理を codecommit で実施し、 CloudWathLogs を確認することで、通知される event 内容を確認できます。

```python
import json
import os
import re
import requests
import sys
from logging import getLogger, INFO ,StreamHandler

logger = getLogger(__name__)
logger.setLevel(INFO)

def lambda_handler(event, context) -> None:
    sns_dict = event['Records'][0]['Sns']
    sns_dict['Message'] = json.loads(sns_dict['Message'])
    sns_dict_str = json.dumps(sns_dict, ensure_ascii=False)
    logger.info('sns message:')
    logger.info(sns_dict_str)
```

全操作パターン調査するのは大変なので、3 つのみ調査します。
(調査面倒なので仕様書求ム)

#### event 確認:PR 作成

```python
{
    "Type": "Notification",
    "MessageId": "aaaaa-aaaaa-aaaaa-aaaa", #mask
    "TopicArn": "arn:aws:sns:ap-northeast-1:aaaaa:codcommit-notification", # mask
    "Subject": null,
    "Message": {
        "account": "aaaaa", #mask
        "detailType": "CodeCommit Pull Request State Change",
        "region": "ap-northeast-1",
        "source": "aws.codecommit",
        "time": "2021-10-31T16:20:26Z",
        "notificationRuleArn": "arn:aws:codestar-notifications:ap-northeast-1:aaaaa:notificationrule/aaaaa", # mask
        "detail": {
            "sourceReference": "refs/heads/feature/notification_test2",
            "lastModifiedDate": "Sun Oct 31 16:20:18 UTC 2021",
            "author": "arn:aws:iam::aaaaa:user/iam_user_name", #mask
            "isMerged": "False",
            "pullRequestStatus": "Open",
            "notificationBody": "A pull request event occurred in the following AWS CodeCommit repository: test-repository. User: aarn:aws:iam::aaaaa:user/iam_user_name. Event: Created. Pull request name: 12. Additional information: A pull request was created with the following ID: 12. The title of the pull request is: notification test 2. For more information, go to the AWS CodeCommit console {repogitory_link}",
            "description": "notification desictiption",
            "destinationReference": "refs/heads/master",
            "callerUserArn": "arn:aws:iam::aaaaa:user/iam_user_name", #mask
            "creationDate": "Sun Oct 31 16:20:18 UTC 2021",
            "pullRequestId": "12",
            "title": "notification test 2",
            "revisionId": "aaaaaa", # mask
            "repositoryNames": [
                "test-repogitory"
            ],
            "destinationCommit": "423f021893fa3233cf956f287ece47d0987be441",
            "event": "pullRequestCreated",
            "sourceCommit": "bcda8ac3245cf63388d61df6a16769f32b2de590"
        },
        "resources": [
            "arn:aws:codecommit:ap-northeast-1:aaaaa:test-repogitory" #mask
        ],
        "additionalAttributes": {
            "numberOfFilesAdded": "0",
            "numberOfFilesDeleted": "1",
            "numberOfFilesModified": "0",
            "changedFiles": [
                {
                    "changeType": "D",
                    "filePath": "test2.txt"
                }
            ]
        }
    },
    "Timestamp": "2021-10-31T16:20:28.873Z",
    "SignatureVersion": "1",
    "Signature": "aaaaa", # mask
    "SigningCertUrl": "{pem_link}", #mask
    "UnsubscribeUrl": "{link}", #mask
    "MessageAttributes": {}
}
```

#### event 確認:PR マージ

```python
{
    "Type": "Notification",
    "MessageId": "aaaaa-aaaaa-aaaaa-aaaa", #mask
    "TopicArn": "arn:aws:sns:ap-northeast-1:aaaaa:codcommit-notification", #mask
    "Subject": null,
    "Message": {
        "account": "aaaaa", #mask
        "detailType": "CodeCommit Pull Request State Change",
        "region": "ap-northeast-1",
        "source": "aws.codecommit",
        "time": "2021-10-31T16:03:49Z",
        "notificationRuleArn": "arn:aws:codestar-notifications:ap-northeast-1:aaaaa:notificationrule/aaaaa", #mask
        "detail": {
            "sourceReference": "refs/heads/feature/notification_test",
            "lastModifiedDate": "Sun Oct 31 16:03:39 UTC 2021",
            "author": "arn:aws:iam::aaaaa:user/iam_user_name",#mask
            "isMerged": "True",
            "pullRequestStatus": "Closed",
            "notificationBody": "A pull request event occurred in the following AWS CodeCommit repository: test-repogitory. User: arn:aws:iam::aaaa:iam_user_name. Event: Updated. Pull request name: 11. Additional information: The pull request merge status has been updated. The status is merged. For more information, go to the AWS CodeCommit console {repogitory_link}",#mask
            "description": "# notification実装テスト用PRの説明セクション\ntest\n## test\n- test\n  - test",
            "destinationReference": "refs/heads/master",
            "callerUserArn": "arn:aws:iam::aaaaa:user/iam_user_name", #mask
            "creationDate": "Sun Oct 31 14:46:05 UTC 2021",
            "pullRequestId": "11",
            "title": "notification実装テスト用PR",
            "revisionId": "aaaaa", # mask
            "repositoryNames": [
                "test-repogitory"
            ],
            "destinationCommit": "1da430df4696193ce4883bd029d7fa650606a078",
            "event": "pullRequestMergeStatusUpdated",
            "mergeOption": "SQUASH_MERGE",
            "sourceCommit": "ae1dd8f163e45e150d6074f78357d1ce6c030ab8"
        },
        "resources": [
            "arn:aws:codecommit:ap-northeast-1:aaaaa:test-repogitory" #mask
        ],
        "additionalAttributes": {}
    },
    "Timestamp": "2021-10-31T16:03:51.808Z",
    "SignatureVersion": "1",
    "Signature": "aaaaa", # mask
    "SigningCertUrl": "{pem_link}", #mask
    "UnsubscribeUrl": "{link}", #mask
    "MessageAttributes": {}
}
```

#### event 確認:PR コメント追加

```python
{
    "Type": "Notification",
    "MessageId": "aaaaa", # mask
    "TopicArn": "arn:aws:sns:ap-northeast-1:aaaaa:codcommit-notification", # mask
    "Subject": null,
    "Message": {
        "account": "909287433301",
        "detailType": "CodeCommit Comment on Pull Request",
        "region": "ap-northeast-1",
        "source": "aws.codecommit",
        "time": "2021-10-31T15:20:36Z",
        "notificationRuleArn": "arn:aws:codestar-notifications:ap-northeast-1:aaaaa:notificationrule/aaaaa", #mask
        "detail": {
            "beforeCommitId": "1da430df4696193ce4883bd029d7fa650606a078",
            "notificationBody": "A pull request event occurred in the following AWS CodeCommit repository: test-repository. The user: arn:aws:iam::aaaaa:user/iam_user_name made a comment or replied to a comment. The comment was made on the following Pull Request: 11. For more information, go to the AWS CodeCommit console {repogitory_link}", # mask
            "repositoryId": "aaaaa", # mask
            "commentId": "aaaaa", # mask
            "afterCommitId": "c98211927b2d78bc7bad803b08221a0e0329ce20",
            "callerUserArn": "arn:aws:iam::aaaaa:user/iam_user_name", #mask
            "event": "commentOnPullRequestCreated",
            "pullRequestId": "11",
            "repositoryName": "test-repository"
        },
        "resources": [
             "arn:aws:codecommit:ap-northeast-1:aaaaa:test-repogitory" # mask
        ],
        "additionalAttributes": {
            "commentedLine": null,
            "resourceArn":  "arn:aws:codecommit:ap-northeast-1:aaaaa:test-repogitory", # mask
            "comments": [
                {
                    "authorArn": "arn:aws:iam::aaaaa:user/iam_user_name", #mask
                    "commentText": "変更に対するコメントテスト13"
                }
            ],
            "commentedLineNumber": null,
            "filePath": null
        }
    },
    "Timestamp": "2021-10-31T15:20:39.578Z",
    "SignatureVersion": "1",
    "Signature": "aaaaa", # mask
    "SigningCertUrl": "{pem_link}", #mask
    "UnsubscribeUrl": "{link}", #mask
    "MessageAttributes": {}
}

```

## Teams 投稿フォーマット

以下関数で定義される投稿フォーマットに沿うように実装しました。
指定可能なフォーマットオプション色々ありますが、サクッと実装したかったので実装経験あるフォーマットで実装しました。
https://docs.microsoft.com/ja-jp/microsoftteams/platform/webhooks-and-connectors/how-to/connectors-using?tabs=cURL

```python
def post_teams(title: str, detail: list, link: list) -> None:
    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": f'{AWS_ACCOUNT}: {title}',
        "sections": [{
            "activityTitle": f'{AWS_ACCOUNT}: CodeCommit Notifiction',
            "activitySubtitle": f'{title}',
            "activityImage": f'{link["icon_url"]}',
            "facts": detail,
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": f'{link["name"]}',
                "targets": [{
                    "os": "default",
                    "uri": f'{link["url"]}'
                }]
            }]
        }]
    }

    logger.info('post message:')
    logger.info(json.dumps(payload))

    try:
        response = requests.post(TEAMS_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)
```

## 最終的にできたコード

上部の環境変数を別途設定すれば動きます。
`get_message()`で、各 event 通知パターンごとに Teams 投稿データフォーマットに沿った整形をしてあげたうえで 、Teams に投稿を投げています。
相変わらずアイコン設定にセンスがないので、良いアイコン募集中です。

```python
import json
import os
import re
import requests
import sys
from logging import getLogger, INFO ,StreamHandler

logger = getLogger(__name__)
logger.setLevel(INFO)

# SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']
TEAMS_WEBHOOK_URL = os.environ['TEAMS_WEBHOOK_URL']
AWS_ACCOUNT = os.environ['ACCOUNT_NAME']

def lambda_handler(event, context) -> None:
    sns_dict = event['Records'][0]['Sns']
    sns_dict['Message'] = json.loads(sns_dict['Message'])
    sns_dict_str = json.dumps(sns_dict, ensure_ascii=False)
    logger.info('sns message:')
    logger.info(sns_dict_str)
    title, detail, link = get_message(sns_dict['Message'])
    post_teams(title, detail, link)

def get_message(message: dict) -> (str, list, dict):
    title = message['detailType']
    https_pattern = "https?://[\w/:%#\$&\?\(\)~\.=\+\-]+"
    if title == 'CodeCommit Comment on Pull Request':
        # title
        title_fixed = title
        # detail
        user_arn = message['detail']['callerUserArn']
        user_name = user_arn.split('/')[1]

        repo_name = message['detail']['repositoryName']

        comment_line = message['additionalAttributes']['commentedLine'] or '---'
        comment_filepath = message['additionalAttributes']['filePath'] or '---'
        comment_text = message['additionalAttributes']['comments'][0]['commentText']

        detail = [
            {'name':'user', 'value':user_name},
            {'name':'repo_name', 'value':repo_name },
            {'name':'comment_filepath', 'value':comment_filepath},
            {'name':'comment_line', 'value':comment_line},
            {'name':'comment_text', 'value':comment_text}
        ]
        # link
        url_list = re.findall(https_pattern, message['detail']['notificationBody'])
        url = url_list[0]
        url = url[:-1] if url[-1] == '.' else url
        link = {
            'icon_url': 'https://img.icons8.com/external-flatart-icons-outline-flatarticons/64/000000/external-comment-chat-flatart-icons-outline-flatarticons-2.png',
            'name': 'PullRequest',
            'url': url
        }
    elif title == 'CodeCommit Pull Request State Change':
        if message['detail']['event'] == "pullRequestCreated":
            # title
            title_fixed = 'CodeCommit Pull Request Created'
            # detail
            repo_name = message['detail']['repositoryNames'][0]
            src_branch = message['detail']['sourceReference']
            dst_branch = message['detail']['destinationReference']
            author_arn = message['detail']['author']
            author_name = author_arn.split('/')[1]
            description = message['detail']['description']

            detail = [
                {'name':'repo_name', 'value':repo_name },
                {'name':'src_branch', 'value':src_branch},
                {'name':'dst_branch', 'value':dst_branch},
                {'name':'author', 'value':author_name},
                {'name':'description', 'value':description}
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn-icons-png.flaticon.com/512/3659/3659866.png',
                'name': 'PullRequest',
                'url': url
            }
        elif message['detail']['isMerged'] == "True":
            # title
            title_fixed = 'CodeCommit Pull Request Mereged'
            # detail
            repo_name = message['detail']['repositoryNames'][0]
            src_branch = message['detail']['sourceReference']
            dst_branch = message['detail']['destinationReference']
            merge_option = message['detail']['mergeOption']

            detail = [
                {'name':'repo_name', 'value':repo_name},
                {'name':'src_branch', 'value':src_branch},
                {'name':'dst_branch', 'value':dst_branch}
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn0.iconfinder.com/data/icons/octicons/1024/git-pull-request-512.png',
                'name': 'PullRequest',
                'url': url
            }
    else:
        # title
        title_fixed = title
        # detail
        detail = [
            {'name':'notificationBody', 'value':message['detail']['notificationBody']}
        ]
        # link
        url_list = re.findall(https_pattern, message['detail']['notificationBody'])
        url = url_list[0]
        url = url[:-1] if url[-1] == '.' else url
        link = {
            'icon_url': 'https://cdn-icons.flaticon.com/png/512/2076/premium/2076218.png?token=exp=1635696056~hmac=6f323fa1adc2e6d6de8d2b961c96984b',
            'name': 'PullRequest',
            'url': url
        }

    return title_fixed, detail, link

def post_teams(title: str, detail: list, link: list) -> None:

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": f'{AWS_ACCOUNT}: {title}',
        "sections": [{
            "activityTitle": f'{AWS_ACCOUNT}: CodeCommit Notifiction',
            "activitySubtitle": f'{title}',
            "activityImage": f'{link["icon_url"]}',
            "facts": detail,
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": f'{link["name"]}',
                "targets": [{
                    "os": "default",
                    "uri": f'{link["url"]}'
                }]
            }]
        }]
    }

    logger.info('post message:')
    logger.info(json.dumps(payload))

    try:
        response = requests.post(TEAMS_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)

```

## 終わりに

event パターン無限にあり一個ずつ調べるのは非常にしんどいので仕様書がほしい。

PR 承認通知などを追加

```python:lambda_function.py
import json
import os
import re
import requests
import sys
from logging import getLogger, INFO ,StreamHandler

logger = getLogger(__name__)
logger.setLevel(INFO)

# SLACK_WEBHOOK_URL = os.environ['SLACK_WEBHOOK_URL']
TEAMS_WEBHOOK_URL = os.environ['TEAMS_WEBHOOK_URL']
AWS_ACCOUNT = os.environ['ACCOUNT_NAME']

def lambda_handler(event, context) -> None:
    sns_dict = event['Records'][0]['Sns']
    sns_dict['Message'] = json.loads(sns_dict['Message'])
    sns_dict_str = json.dumps(sns_dict, ensure_ascii=False)
    logger.info('sns message:')
    logger.info(sns_dict_str)
    msg_title, detail, link = get_message(sns_dict['Message'])
    post_teams(msg_title, detail, link)

def get_message(message: dict) -> (str, list, dict):
    msg_title = message['detailType']
    https_pattern = "https?://[\w/:%#\$&\?\(\)~\.=\+\-]+"
    # PRへのコメント
    if msg_title == 'CodeCommit Comment on Pull Request':
        # msg_title
        msg_title_fixed = msg_title
        # detail
        user_arn = message['detail']['callerUserArn']
        user_name = user_arn.split('/')[1]

        repo_name = message['detail']['repositoryName']

        comment_line = message['additionalAttributes']['commentedLine'] or '---'
        comment_filepath = message['additionalAttributes']['filePath'] or '---'
        comment_text = message['additionalAttributes']['comments'][0]['commentText']

        detail = [
            {'name':'user', 'value':user_name},
            {'name':'repo_name', 'value':repo_name },
            {'name':'comment_filepath', 'value':comment_filepath},
            {'name':'comment_line', 'value':comment_line},
            {'name':'comment_text', 'value':comment_text}
        ]
        # link
        url_list = re.findall(https_pattern, message['detail']['notificationBody'])
        url = url_list[0]
        url = url[:-1] if url[-1] == '.' else url
        link = {
            'icon_url': 'https://img.icons8.com/external-flatart-icons-outline-flatarticons/64/000000/external-comment-chat-flatart-icons-outline-flatarticons-2.png',
            'name': 'PullRequest',
            'url': url
        }
    elif msg_title == 'CodeCommit Pull Request State Change':
        # PR作成
        if message['detail']['event'] == "pullRequestCreated":
            # msg_title
            msg_title_fixed = 'CodeCommit Pull Request Created'
            # detail
            repo_name = message['detail']['repositoryNames'][0]
            src_branch = message['detail']['sourceReference']
            dst_branch = message['detail']['destinationReference']
            author_arn = message['detail']['author']
            author_name = author_arn.split('/')[1]
            description = message['detail']['description']

            detail = [
                {'name':'repo_name', 'value':repo_name },
                {'name':'src_branch', 'value':src_branch},
                {'name':'dst_branch', 'value':dst_branch},
                {'name':'author', 'value':author_name},
                {'name':'description', 'value':description}
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn-icons-png.flaticon.com/512/3659/3659866.png',
                'name': 'PullRequest',
                'url': url
            }
        # PR承認
        elif message['detail']['approvalStatus'] == 'APPROVE':
             # msg_title
            msg_title_fixed = 'CodeCommit Pull Request Approved'
            # detail
            repo_name = message['detail']['repositoryNames'][0]
            src_branch = message['detail']['sourceReference']
            dst_branch = message['detail']['destinationReference']

            detail = [
                {'name':'repo_name', 'value':repo_name},
                {'name':'src_branch', 'value':src_branch},
                {'name':'dst_branch', 'value':dst_branch}
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn-icons-png.flaticon.com/512/1236/1236666.png',
                'name': 'PullRequest',
                'url': url
            }
        # PRマージ
        elif message['detail']['isMerged'] == "True":
            # msg_title
            msg_title_fixed = 'CodeCommit Pull Request Mereged'
            # detail
            repo_name = message['detail']['repositoryNames'][0]
            src_branch = message['detail']['sourceReference']
            dst_branch = message['detail']['destinationReference']
            merge_option = message['detail']['mergeOption']

            detail = [
                {'name':'repo_name', 'value':repo_name},
                {'name':'src_branch', 'value':src_branch},
                {'name':'dst_branch', 'value':dst_branch},
                {'name':'merge_option', 'value':merge_option},
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn0.iconfinder.com/data/icons/octicons/1024/git-pull-request-512.png',
                'name': 'PullRequest',
                'url': url
            }
        else:
            # msg_title
            msg_title_fixed = msg_title
            # detail
            detail = [
                {'name':'notificationBody', 'value':message['detail']['notificationBody']}
            ]
            # link
            url_list = re.findall(https_pattern, message['detail']['notificationBody'])
            url = url_list[0]
            url = url[:-1] if url[-1] == '.' else url
            link = {
                'icon_url': 'https://cdn-icons.flaticon.com/png/512/2076/premium/2076218.png?token=exp=1635696056~hmac=6f323fa1adc2e6d6de8d2b961c96984b',
                'name': 'PullRequest',
                'url': url
            }
    else:
        # msg_title
        msg_title_fixed = msg_title
        # detail
        detail = [
            {'name':'notificationBody', 'value':message['detail']['notificationBody']}
        ]
        # link
        url_list = re.findall(https_pattern, message['detail']['notificationBody'])
        url = url_list[0]
        url = url[:-1] if url[-1] == '.' else url
        link = {
            'icon_url': 'https://cdn-icons.flaticon.com/png/512/2076/premium/2076218.png?token=exp=1635696056~hmac=6f323fa1adc2e6d6de8d2b961c96984b',
            'name': 'PullRequest',
            'url': url
        }

    return msg_title_fixed, detail, link

def post_teams(msg_title: str, detail: list, link: list) -> None:

    payload = {
        '@type': 'MessageCard',
        "@context": "http://schema.org/extensions",
        "themeColor": "0076D7",
        "summary": f'{AWS_ACCOUNT}: {msg_title}',
        "sections": [{
            "activityTitle": f'{AWS_ACCOUNT}: CodeCommit Notifiction',
            "activitySubtitle": f'{msg_title}',
            "activityImage": f'{link["icon_url"]}',
            "facts": detail,
            "markdown": 'true',
            "potentialAction": [{
                "@type": "OpenUri",
                "name": f'{link["name"]}',
                "targets": [{
                    "os": "default",
                    "uri": f'{link["url"]}'
                }]
            }]
        }]
    }

    logger.info('post message:')
    logger.info(json.dumps(payload))

    try:
        response = requests.post(TEAMS_WEBHOOK_URL, data=json.dumps(payload))
    except requests.exceptions.RequestException as e:
        print(e)
    else:
        print(response.status_code)

```
