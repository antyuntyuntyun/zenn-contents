# Zenn Contents

- [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [📘 Markdown guide](https://zenn.dev/zenn/articles/markdown-guide)

## How to use

```bash
# zenn-cli のインストール
npm install zenn-cli

# 記事の作成
npx zenn new:article
npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea --emoji ✨
# 記事はGithubレポジトリと同期されているので, git pushにより更新されます
# ファイル名を後から変更すると, zennで記事が複製されてしまうので, push前にファイル名変更推奨です

# プレビュー開始
npx zenn preview
npx zenn preview --port 3000
```

### 記事への画像の追加

下記 URL に画像をアップロードし、本文挿入用 URL を取得して利用。
https://zenn.dev/dashboard/uploader
