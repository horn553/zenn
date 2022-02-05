# Zenn執筆環境

## 記事を追加

ref: [Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)

1. .md ファイルを作成

```
$ npx zenn new:article
```

2. プレビュー

```
$ npx zenn preview
```

3. 記事を公開

.md ファイルのオプション`preview`を`true`に設定する。

## 画像を追加

ref: [GitHubリポジトリ連携で画像をアップロードする方法](https://zenn.dev/zenn/articles/deploy-github-images)

1. ディレクトリ`/images/articles/[slug]`に画像を配置
1. `/images/articles/[slug]/...`で画像を参照
