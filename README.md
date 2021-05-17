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

# プレビュー開始
npx zenn preview
npx zenn preview --port 3000
```
