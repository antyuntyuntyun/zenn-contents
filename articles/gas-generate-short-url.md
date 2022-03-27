---
title: 'GASでbitly APIを叩いて短縮URLをまとめて生成'
emoji: '💬'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['GAS', 'javascript', 'web']
published: true
---

# GAS で短縮 URL をまとめて生成したい

GA 計測用の utm パラメータを付ける必要があるなどの理由で URL が長くなってしまった場合、短縮 URL を発行したいシーンがあると思います。(Twitter への投稿添付 URL で利用する場合など)
１つ 2 つならまだしも、何十あるものを手動でポチポチ発行すると日が暮れるので GAS でまとめて生成できるようにしました。

## GAS

冒頭の変数に必要な値を指定して利用します。
アクティブシートのみに対して処理を実行したいときには`wirteshortenUrlActiveSheet()`、全シートに対して実行したいときには`wirteshortenUrlAllSheet()`を実行します。
※全シート処理実行の場合、冒頭の変数定義が共通で使えるようにフォーマットが揃っている必要があります。

```js: generateShortenUrl.js
FIRST_ROW = 5 // 処理開始行数を指定
URL_COLUMN_CHAR = 'L' // 短縮処理するURLが入力されている列を指定
SHORTEN_URL_COLUMN_CHAR = 'M' // 短縮処理するURLを入力したい列を指定
BITLY_TOKEN = 'xxxxx' // bitlyから取得したAPIトークンを指定
BITLY_URL = 'https://api-ssl.bitly.com/v4/shorten' // bitly api v4

/**
 * 対象のスプレッドシートのアクティブシートに対して短縮URL生成処理を実行
 */
function wirteshortenUrlAllSheet() {
  // アクティブシートの取得
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet()

  // 短縮処理するURLが入力されている列の値と行数を取得
  const value = sheet
    .getRange(`${URL_COLUMN_CHAR}${FIRST_ROW}:${URL_COLUMN_CHAR}`)
    .getValues()
    .filter(String)
  const length = value.length
  const shortenUrls = [] // 短縮URL格納用
  for (var i = 0; i < length; i++) {
    console.log(
      `sheetName: ${sheet.getName()} cell:${URL_COLUMN_CHAR}${
        FIRST_ROW + i
      } (${i} / ${length})`
    )
    console.log(`url: ${value[i][0]}`)
    const shortenUrlCellValue = sheet
      .getRange(`${SHORTEN_URL_COLUMN_CHAR}${FIRST_ROW + i}`)
      .getValue()
    // 短縮URLがすでに入力済みの場合は処理済みとみなしてスキップ
    if (shortenUrlCellValue.length > 0) {
      shortenUrls.push([shortenUrlCellValue])
      console.log('Skip this URL as it has already been entered.')
      console.log(`already entered: ${shortenUrlCellValue}`)
    } else {
      let shortenUrl = getshortenUrl(value[i][0])
      console.log(`shortenUrl: ${shortenUrl}`)
      shortenUrls.push([shortenUrl])
    }
    Utilities.sleep(0.2 * 1000) // APIのrate制限対策のため2秒sleep
  }
  // 短縮URLをスプレッドシートに入力
  sheet
    .getRange(
      `${SHORTEN_URL_COLUMN_CHAR}${FIRST_ROW}:${SHORTEN_URL_COLUMN_CHAR}${
        FIRST_ROW + length - 1
      }`
    )
    .setValues(shortenUrls)
}

/**
 * 対象のスプレッドシートの全シートに対して短縮URL生成処理を実行
 */
function wirteshortenUrlAllSheet() {
  // スプレッドシートの全シートを取得
  const sheets = SpreadsheetApp.getActiveSpreadsheet().getSheets()

  for (const sheet of sheets) {
    // 短縮処理するURLが入力されている列の値と行数を取得
    const value = sheet
      .getRange(`${URL_COLUMN_CHAR}${FIRST_ROW}:${URL_COLUMN_CHAR}`)
      .getValues()
      .filter(String)
    const length = value.length
    const shortenUrls = [] // 短縮URL格納用
    for (var i = 0; i < length; i++) {
      console.log(
        `sheetName: ${sheet.getName()} cell:${URL_COLUMN_CHAR}${
          FIRST_ROW + i
        } (${i} / ${length})`
      )
      console.log(`url: ${value[i][0]}`)
      const shortenUrlCellValue = sheet
        .getRange(`${SHORTEN_URL_COLUMN_CHAR}${FIRST_ROW + i}`)
        .getValue()
      // 短縮URLがすでに入力済みの場合は処理済みとみなしてスキップ
      if (shortenUrlCellValue.length > 0) {
        shortenUrls.push([shortenUrlCellValue])
        console.log('Skip this URL as it has already been entered.')
        console.log(`already entered: ${shortenUrlCellValue}`)
      } else {
        let shortenUrl = getshortenUrl(value[i][0])
        console.log(`shortenUrl: ${shortenUrl}`)
        shortenUrls.push([shortenUrl])
      }
      Utilities.sleep(0.2 * 1000) // APIのrate制限対策のため2秒sleep
    }
    // 短縮URLをスプレッドシートに入力
    sheet
      .getRange(
        `${SHORTEN_URL_COLUMN_CHAR}${FIRST_ROW}:${SHORTEN_URL_COLUMN_CHAR}${
          FIRST_ROW + length - 1
        }`
      )
      .setValues(shortenUrls)
  }
}

/**
 * bitlyapiから短縮URLを生成
 * @param {string} longUrl - 短縮したい長いURL
 * @returns {string} - 短縮URL
 */
function getshortenUrl(longUrl) {
  const payload = { long_url: longUrl }
  const headers = {
    Authorization: 'Bearer ' + BITLY_TOKEN,
    'Content-Type': 'application/json',
  }
  const options = {
    method: 'POST',
    headers: headers,
    payload: JSON.stringify(payload),
  }
  const response = UrlFetchApp.fetch(BITLY_URL, options)
  const content = response.getContentText('UTF-8')

  return JSON.parse(content).link
}

```

## 処理イメージ

処理前

| longUrl                 | shortenUrl |
| ----------------------- | ---------- |
| https://longurl         |            |
| https://longlongurl     |            |
| https://longlonglongurl |            |

処理後

| longUrl                 | shortenUrl       |
| ----------------------- | ---------------- |
| https://longurl         | https://bit.ly/a |
| https://longlongurl     | https://bit.ly/b |
| https://longlonglongurl | https://bit.ly/c |

## 最後に注意点など

特にエラーハンドリング等を入れてないので, URL として不正な文字列で短縮変換ができないものであったり空文字が含まれていた場合、おそらく途中で処理が止まります。
bitly の無料アカウントだと月 100 件までしか短縮 URL を生成できないので注意する必要があります。bitly 以外で制限がもう少し緩い短縮 URL 生成サービスがあるならばそちらを使うことも検討したほうがよいかなと思うものの、何百も生成する必要が今のところないので一旦ここまで。

https://bitly.com/pages/pricing/v1
