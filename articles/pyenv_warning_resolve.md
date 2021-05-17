---
title: "シェル起動時のpyenvのWarning解決"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "pyenv", "anyenv"]
published: true
---

## pyenv 仕様変更による Warning

2021/5 初旬から pyenv の仕様変更により, シェルを起動時に以下 warning が出るようになりました.本記事は, Warning 解決方法メモです.

```bash
WARNING: `pyenv init -` no longer sets PATH.
Run `pyenv init` to see the necessary changes to make to your configuration.
```

## pyenv レポジトリであがっていた issue

こちらです. こちら参考に解決しました.
https://github.com/pyenv/pyenv/issues/1906

## Warning 解決方法

`.bashrc`や`.zshrc`へ以下の追記で解決します.

素の pyenv を利用している場合

```bash
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
if command -v pyenv 1>/dev/null 2>&1; then
  eval "$(pyenv init --path)"
fi
```

anyenv 経由の pyenv を利用している場合(インストールフォルダが`~/.anyenv`の場合)

```bash
export PYENV_ROOT="$HOME/.anyenv/envs/pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
if command -v pyenv 1>/dev/null 2>&1; then
  eval "$(pyenv init -)"
fi
```

## 最後に

身の回りに anyenv 使っている人がとても少なくて悲しいので, 最後に aynenv の宣伝しておきます.
https://github.com/anyenv/anyenv
