---
title: "fzfを用いてssh先を簡単指定"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["fzf", "ssh", "cli"]
published: false
---

# はじめに

先日、aws-cli のプロファイルを簡単に切り替えられるように fzf で自作関数を書いてシェルに登録した。
同じことを ssh ホストの切り替えでやりたくなったのでやってみました。
https://qiita.com/antyuntyuntyun/items/5976ef838ec160f6b027

## fzf

fzf とは、cli 上でのあいまい検索および結果選択が可能になる、GO 製の CLI ツールです。
peco とよく比較されます。
詳細は以下記事等参照

https://qiita.com/kamykn/items/aa9920f07487559c0c7e

### fzf のインストール

```bash
$ git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
$ ~/.fzf/install
```

mac の方は`brew install fzf`でも大丈夫です。詳細は本家レポジトリをご確認ください。
https://github.com/junegunn/fzf

## シェル設定ファイルに関数追加

~/.ssh/config を参照して、Host アドレス一覧を取得し、その中から ssh 接続先を fzf で選択する、という挙動の関数を作成します。
.bashrc や.zshrc に以下関数定義を追記します。

```bash
function sshsp() {
  local host=$(grep -E "^Host|^$" ~/.ssh/config | sed 's/Host //' | fzf)
  ssh $host
}
```

## 自作 fzf 関数を呼び出して SSH 接続先を指定

関数定義後に、シェル再起動(exec $SHELL -l)や設定ファイル再読み込み(source ~/.zshrc)を実施しシェルに反映させます。
あとは、シェルで`sshsp`コマンドを打つだけでプロファイルを切り替えられるようになります。
