---
title: "pythonからGoogleDriveAPIを叩く"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [python, googledrive]
published: true
---

# はじめに

python から GoogleDriveAPI を叩けるようになるまでのセットアップ手順です。

## GCP コンソール上での API/サービスアカウントセットアップ

### GCP コンソール

処理したい GoogleDrive にアクセス権のある Google アカウントは既にある前提とします。
以下から GCP コンソールにアクセスします。
初回のアクセスの方は、チュートリアルに沿ってプロジェクトを作成します。
以降の操作は、任意のプロジェクト内の操作として実施していきます。
https://console.cloud.google.com/?hl=ja

### API 有効化

API とサービスから有効化する API を設定します。左上のハンバーガーメニューから
`API とサービス -> ライブラリ`
と辿っていきます。今回は、`google drive api`および`google sheets api`で検索してヒットする API を有効化します。
以上の操作で、このプロジェクトでの GoogleDrive およびスプレッドシートの操作の API が有効化されました。

### サービスアカウントの発行

実際に API を呼び出すアカウント(認証情報)を発行していきます。
左上のハンバーガーメニューから
`API とサービス -> 認証情報 -> 認証情報の作成`
と辿っていきサービスアカウントを作成します。

作成できる認証情報は

1. API キー
2. OAuth
3. サービスアカウント

の三つがあります。
Robot 実行的なことをさせるのであれば、基本的にはサービスアカウント発行で良いと思います。
認証情報の違いについては、以下の記事でわかりやすくまとまっているので、是非参照してください。

https://messefor.hatenablog.com/entry/2020/10/08/080414

### 認証キーの発行

鍵(キー)メニューから認証キーを発行します。
JSON 形式を選択し発行しましょう。

## 処理対象となる GoogleDrive フォルダへのアクセス権付与

認証情報のトップページに、作成したサービスアカウントの Email アドレスが表示されているのでコピーします。
GoogleDrive で操作したいフォルダにコピーしたメアドで権限付与します。
GoogleDrive 上のフォルダで 右クリック -> 共有 でメアドを入力して権限付与しましょう。

## python から API を呼び出す

ここまでの操作で、 API 有効化とサービスアカウントの発行および認証情報の取得が完了したので、あとはコードを書いていきます。
`python3.8.7`で動作確認しています。

### ライブラリインストール

```bash
pip install google-api-python-client
```

### セットアップ

まず、任意のパスに配置された認証情報を読み込みます。

必要なライブラリをインポートします

```python
import glob
import json
import os
import pandas as pd
from apiclient.http import MediaFileUpload, MediaIoBaseDownload
from genericpath import exists
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError as HTTPError
from oauth2client.service_account import ServiceAccountCredentials
import io
```

任意のパスに配置したサービスアカウントの認証情報を取得します。

```python
# 認証トークンJSONを取得
credential_folder_path = 'credential_folder_path/'
files = glob.glob(f'{credential_folder_path}/*.json')
try:
    jsonf = files[0]
    google_app_credential = json.load(open(jsonf, 'r'))
except Exception as e:
    print('Error: Get Google Drive API Credential')
    raise e
```

GoogleDrive 操作をするためのインスタンスを作成します。
SCOPES で操作する範囲の API エンドポイントを指定しています。

```python
SCOPES = [
    'https://www.googleapis.com/auth/drive',
    'https://www.googleapis.com/auth/spreadsheets']
# SHARE_FOLDER_ID = 'your folder id'
sa_creds = service_account.Credentials.from_service_account_file(self.jsonf)
scoped_creds = sa_creds.with_scopes(SCOPES)
drive_service = build('drive', 'v3', credentials=scoped_creds)
```

これで準備が整いました。
実際にお試し操作していきます。

### pd.DataFrame をスプレッドシートとしてアップロード

```python
tmp_csv_file_path = './tmp.csv'
parent_folder_id = 'xxxxxxx' # アップロード先フォルダID。https://drive.google.com/drive/folders/xxxxxxxxxx <- ここの部分

df.to_csv(tmp_csv_file_path , index=False)

file_metadata = {
    'name': name,
    'parents': [parent_folder_id],
    'mimeType': 'application/vnd.google-apps.spreadsheet',
}
media = MediaFileUpload(
    tmp_csv_file_path ,
    mimetype='text/csv',
    resumable=True
)

response = drive_service.files().create(
    body=file_metadata, media_body=media, fields='id'
).execute()

# tmpファイルの削除
os.remove(tmp_csv_file_path)
```

### GoogleDrive 上のファイルを pd.DataFrame として読み込み

```python
file_id = 'xxxxxxx' # 読み込みたいファイルのID。https://drive.google.com/drive/folders/xxxxxxxxxx <- ここの部分

request = drive_service.files().get_media(fileId=file_id)
fh = io.BytesIO()
downloader = MediaIoBaseDownload(fh, request)
done = False
while done is False:
    status, done = downloader.next_chunk()

return pd.read_csv(io.StringIO(fh.getvalue().decode()))
```

### 　ファイルの削除

```python
ile_id = 'xxxxxxx' # 読み込みたいファイルのID。https://drive.google.com/drive/folders/xxxxxxxxxx <- ここの部分

self.drive_service.files().delete(
    fileId=file_id
).execute()
```

### スプレッドシート操作

(体力切れしたので割愛)

## API ドキュメント

利用しているパラメータについて、詳細は公式ドキュメントを見ましょう。
`update()`による PATCH 操作は、csv ファイルで試行したところ意図しない変更結果になってしまったので、PATCH でなく削除してアップロードがおすすめです。
https://developers.google.com/drive/api/v3/reference/files

## PyDrive

ラッパーライブラリもあるみたいです。
API 直叩きしたほうが、他言語で API 呼ぶ時に転用できる知識になるから良いかなと思っているんですが、それを超える便利さがこのライブラリにあるかもしれません。
しかし、体力切れなのでまた今度確認します。
利用したことある方いればコメントで良さげかどうか教えてください。
https://github.com/googlearchive/PyDrive
