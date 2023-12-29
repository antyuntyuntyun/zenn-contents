---
title: 'LLM速習ログ'
emoji: '🤗'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

# はじめに

仕事・プライベート上での様々な環境変化や私自身の変化があり、最近ではコード書かない(書けない?)おじさんになってしまいつつありますが、年末年始休暇のタイミングで LLM 周りを学んで時代にキャッチアップしたくなったので、備忘として残します。

## LLM 概要をサクッと学ぶ

NTT データさんの記事。概要理解のための資料としてとてもわかりやすいです。
https://speakerdeck.com/kyoun/llm-introduction-ses2023

tty さんの連載記事。実装例も掲載されており、とてもわかりやすいです。
https://zenn.dev/ttya16/articles/ce89dcab833d32cadb39#fn-b532-1

## 感情分類のお試し実装

### 事前学習済みモデル/トークナイザの下調べ

東北大乾研究室が公開している BERT の事前学習済みモデルを利用しているネット記事が多い。
https://github.com/cl-tohoku/bert-japanese
https://huggingface.co/cl-tohoku/bert-base-japanese-whole-word-masking

- トークナイザ: BertJapaneseTokenizer
- トークナイズには MeCab
- コーパスは Wikipedia 日本語版
  - github 上記載
    - `モデルは、CC-100 データセットの日本語部分とウィキペディアの日本語版でトレーニングされています。Wikipedia については、 2023 年 1 月 2 日の時点でWikipedia Cirrussearch ダンプ ファイルからテキスト コーパスを生成しました。`
  - HuggingFace 上記載
    - `The model is trained on Japanese Wikipedia as of September 1, 2019. To generate the training corpus, WikiExtractor is used to extract plain texts from a dump file of Wikipedia articles. The text files used for the training are 2.6GB in size, consisting of approximately 17M sentences.`

日本語の感情分類分析(ポジネガ判定など)のために、`daigo/bert-base-japanese-sentiment`を利用しているネット記事が多い。
しかし、現在 Hugginface 上から削除されていて利用できない模様。
https://huggingface.co/daigo/bert-base-japanese-sentiment

### 環境構築

Docker でコンテナ環境構築してみたかったが、参考記事が見つからなかったのもあり断念。colab で実装する方針に変更しました。

### お試し実装

感情分類のためのトークナイザは、Huggingface 上でダウンロード数が多そうだった`koheiduck/bert-japanese-finetuned-sentiment`を利用。
概要セクションで参照させていただいている記事を参考にコードを書かせていただいています。(トークナイザ指定の変更、およびパッケージインストール部分を変更して colab 上で動くようにしています。)

```python
# transformersのインストール
! pip install transformers[torch]
# 東北大学の日本語用BERT使用に必要なパッケージをインストール
! pip install fugashi ipadic

from transformers import AutoModelForSequenceClassification, pipeline, BertJapaneseTokenizer

model = AutoModelForSequenceClassification.from_pretrained('koheiduck/bert-japanese-finetuned-sentiment')
tokenizer = BertJapaneseTokenizer.from_pretrained('cl-tohoku/bert-base-japanese-whole-word-masking')
classifier = pipeline("sentiment-analysis",model=model,tokenizer=tokenizer)

jp_texts = [
    "今日は雨が凄いので外出したくないな。",
    "今日は天気が良いので外に出かけたいな。",
    "先週見た映画はびっくりするぐらい退屈だったぜ！",
    "昨夜見た映画は面白かったなあ。"
]

classifier(jp_texts)
```

## わからないこと

- 事前学習済みモデル/トークナイザは何を使うべきか？
  - 東北大学乾研究室が公開しているモデルを利用することがスタンダードっぽいが、、、
  - その中でいくつか派生モデルあるが、ネット上よく使われていそうな`cl-tohoku/bert-base-japanese-whole-word-masking`を盲目的に利用するで良いのか？どういうユースケースごとに使い分けるべきなのか？
  - `cl-tohoku/bert-base-japanese-whole-word-masking`は結局 2019 年時点の Wiki データを利用しているのか、2023 年 1 月時点データなのか
- colab 上では簡単にコード実行できたが、Huggingface の transformer 付近を扱える環境を作れるコンテナファイルどこかにないのか
- 事前学習済みモデル/トークナイザ/ファインチューニングのためのデータセット をどういう基準で選ぶべきか。
  - (ダウンロード数多いとかネット記事でよく参照されているとか、BERT だったら多分大丈夫だよねみたいな基準しか今ない)
  - Huggingface 公開されている大学などの公的機関と紐づいていないモノを使って良いものか(何を持って信用できるとするか)
  - `daigo/bert-base-japanese-sentiment` が以前は利用できて現在利用できなくなるように、公開されている事前学習済みモデルが突然使えなくなることもあるが、どうメンテしていくべきか

# おわりに

LLM 周りは技術的にとても面白くてワクワクするモノですが、各業界・各企業が自社にどう適用してどう活用していくかというところがチャレンジングだなとお試し実装しながら感じました。
下記記事でまとめられている通り、自社データと組み合わせて使うという大きな方向性はありますが、言語解析してその結果をじゃあどう活用するのか？文脈なり感情分類なりテキスト生成なりができて、それをじゃあ何に使うのか？というのが難しいところだと感じています。
(よくあるチャットサービス活用以外にどのように活用していくべきか、、)

https://zenn.dev/yoshiyuki_kono/articles/63e731ab98ffca
