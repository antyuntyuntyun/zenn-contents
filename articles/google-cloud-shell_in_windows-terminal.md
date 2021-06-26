---
title: "WindowsTerminalにGoogleCloudShellを導入"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["windows", "windowsterminal", "googlecloudplat"]
published: true
---

## はじめに

![WindowsTerminal](https://storage.googleapis.com/zenn-user-upload/4c89bc08a22c1aebc6d24ece.png)

WindowsTerminal に GoogleCloudShell のプロファイルを追加できるようにしたので、 設定方法についての紹介記事です。
WSL,WindowsTerminal,VScode 等のツールで Windows でも必要十分な開発環境が整えられるような時代になってきました。
WindowsTerminal ではデフォルトで AzureCloudShell がプロファイル設定項目として用意されていますが, GoogleCloudShell のプロファイルも追加してより充実させましょう！

### WindowsTerminal とは

WindowsTerminal は任意のコマンドラインアプリを実行できる Microsfot 制のターミナルエミュレータです。2020 年半ばから安定リリースされています。
開発が盛んで機能が豊富で進化し続けているのと, カスタマイズ性が高いのが魅力です。
公式ドキュメントは以下です。
https://docs.microsoft.com/ja-jp/windows/terminal/get-started

ユーザ領域内でインストールする必要がある業務 PC 等であれば、Windows の Powershell 環境で動くパッケージマネージャーの scoop を用いてインストールしましょう。
(chocolatey は管理者権限必要だったりで面倒なのと、winget はまだ出始めで未熟なのもあり、当面は scoop が推奨と個人的に思ってます。)

```bash
scoop install windows-terminal
```

### Google Cloud Shell とは

Google Cloud Platform が提供する、各クラウドリソースに対してアクセスすることができるブラウザアクセス可能なシェル環境です。Google Cloud Shell は無料で利用可能です。
Go がプリインストールされているので、Go の開発環境を構築することなく気軽に go のコードをサクッと書いて実行することができるのも 1 つの魅力です。
https://cloud.google.com/shell?hl=ja

## WindowsTerminal に GoogleCloudShell の導入

以下が実際の導入手順です。

## 対象ユーザと必要な環境

- Windows ユーザ
- wsl 導入済み
  - wsl 内に python3.2- 3.7 導入済み
- WindowsTerminal 導入済み

## gcloud 環境設定 in WSL

gcloud SDK を wsl にインストールします。
公式ドキュメントの以下の手順を参考に実施します。
https://cloud.google.com/sdk/docs/install?hl=JA#linux

私の場合は, wsl 上で以下コマンドで`~/.google-cloud-sdk`以下にインストールしました。

```bash
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-345.0.0-linux-x86_64.tar.gz
mkdir ~/.google-cloud-sdk && tar -zxvf google-cloud-sdk-345.0.0-linux-x86_64.tar.gz -C ~/.google-cloud-sdk --strip-components 1
rm -f google-cloud-sdk-345.0.0-linux-x86_64.tar.gz
~/.google-cloud-sdk/install.sh
```

インストールスクリプトを実施すると、2 点聞かれます。
1 問目について、は GoogleCloudSDK の開発のための自身の情報提供しても OK な方は Yes で。
2 問目については、シェルの設定ファイルを更新してよいかの質問で、してもらいたい方は Yes にしましょう。

私の場合は、PATH 通すの含めて以下のようにシェル設定ファイルに自身で追記しました。
(PATH を通さない場合は、`~/.google-cloud-sdk/bin/gcloud`といった形で直接バイナリを指定して以降コマンドを実行する必要があります。)

```bash:.zshrc
# google cloud sdk path
if [ -d $HOME/.google-cloud-sdk ]
then
    export PATH="$HOME/.google-cloud-sdk/bin:$PATH"
fi

# The next line updates PATH for the Google Cloud SDK.
if [ -f '$HOME/.google-cloud-sdk/path.zsh.inc' ];
then
    . '$HOME/.google-cloud-sdk/path.zsh.inc';
fi

# The next line enables shell command completion for gcloud.
if [ -f '$HOME/.google-cloud-sdk/completion.zsh.inc' ];
then
    . '$HOME/.google-cloud-sdk/completion.zsh.inc';
fi

```

また、利用したい Google アカウントの gcloud SDK の認証も済ませておきましょう。
以下コマンドを実行し、指定された URL をブラウザで開き、認証したい Google アカウントで認証を実施します。

```bash
gcloud auth login
```

## GoogleCloudShell 起動スクリプトの用意

wsl 上に glcoud を利用した GoogleCloudShell 起動スクリプトを用意します。
こちらのスクリプトを呼び出すことで、WindowsTerminal から GoogleCloudShell を起動できるようにします。
私の場合は`~/WindowsTerminal/set-gcloud-env.sh`に置きましたが、任意の場所で構いません。

```bash: ~/WindowsTerminal/set-gcloud-env.sh
#!/bin/sh
# $1: google acount (xxx@gmail.com)
# $2: project name

~/.google-cloud-sdk/bin/gcloud auth login $1
~/.google-cloud-sdk/bin/gcloud config set project $2

~/.google-cloud-sdk/bin/gcloud cloud-shell ssh
```

## WindowsTerminal の設定

WindowsTerminal 上で Ctrl + Shift + , キーを押下することで開ける、WindowsTerminal 設定ファイルの`settings.json`を編集します。
以下追記部分を追加することで、WindowsTerminal の起動シェル選択オプションに前章で書いたシェルスクリプト実行が選択肢として追加されます。

```
{
    ...

    "profiles":
    {
        "defaults":
        {
            ...
        },
        "list":
        [
            {
                ...
            },
            // 追記部分
            {
                "guid": "{{guid}}",
                "name": "Personal GCP",
                "commandline": "wsl ~/WindowsTerminal/set-gcloud-env.sh {google-accont} {project name}"
                "hidden": false,
            }
            // 追記部分
        ]
    },
}

```

`{guid}` は PowerShell コマンドで新規発行する必要があります。PowerShell 上で `[guid]::NewGuid()` コマンドを実行し、発行された ID を上記に記入しましょう。`{}` が含まれている状態で `"guid": "{aaa-bbb-ccc}"` のように入力する必要があるので注意してください。
`{google-accont}` は Google アカウントを指定します。基本的には`@gmail.com`のメールアドレスになります。
`{project name}` は既存のプロジェクト名を指定します。
既存プロジェクトないが取り急ぎ試してみたい方は、gcloud SDK
`gcloud projects create cloud-shell-env`コマンドを実行し、{project name} = cloud-shell-env
となるプロジェクトを発行し、上記設定ファイルを更新しましょう。

`name`属性は、WindowsTerminal 上で表示される名称になります。プロジェクト名など、自身が識別しやすい文字列に書き換えてください。

## 実際に試してみる

以上で設定は完了なので実際に試してみましょう。
新しいタブの選択メニューから今回追加したプロファイルを選択すると, GoogleCloudShell の起動が確認できると思います。
(初回ログインのみ、ssh キー生成と known_hosts 変更が発生します。)

![WindowsTerminal](https://storage.googleapis.com/zenn-user-upload/4c89bc08a22c1aebc6d24ece.png)

## 最後に

今回は省略しましたが、参考で載せている画像の通りプロファイル毎のアイコンの設定や、各プロファイルごとの背景・フォントのカスタマイズ、シェルの開始ディレクトリ、よく使う処理のショートカットキーを自由にキーバインディングなど、WindowsTerminal はカスタマイズ性が高いので色々楽しめます。
自分好みに改造していき、快適な Windows 開発ライフを送りましょう！

## 参考

https://medium.com/google-cloud/handle-multiple-google-cloud-shell-using-windows-terminal-f5ce0573eaf4
