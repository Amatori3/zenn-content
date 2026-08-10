# zenn-content

[Zenn](https://zenn.dev/) の GitHub 連携用リポジトリ。`articles/` に置いた
Markdown ファイルが、push 後数分でそのまま Zenn 上に反映される。

## セットアップ(最初の1回だけ、手動)

Zenn 側の GitHub 連携は Web UI からしか行えないため、以下は自分で行う。

1. https://zenn.dev/dashboard/deploys を開く
2. 「GitHubからのデプロイ」からこのリポジトリ(`Amatori3/zenn-content`)を連携
3. 連携が完了すれば、以降はこのリポジトリに push するだけで公開・更新される

## 記事の置き方

- パスは `articles/<slug>.md`
- `slug` は半角英小文字・数字・ハイフン・アンダースコアのみ、12〜50文字
- frontmatter は以下の形式

```yaml
---
title: "記事タイトル"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["flutter", "docker"]
published: false # true にすると公開される
---
```

- `published: false` のまま push すれば下書きとして同期される(Zenn上で
  非公開のまま確認できる)。`published: true` にして push すると公開される

## ローカルプレビュー(任意)

zenn-cli のインストールは必須ではない(Zenn 側は `articles/*.md` を見るだけ)。
手元でプレビューしたい場合のみ:

```sh
npx zenn-cli preview
```
