# qiita_project
生地を作ります

```bash
npx qiita new [FileName]
```

```bash
---
title: newArticle001 # 記事のタイトル
tags:
  - "" # タグ（ブロックスタイルで複数タグを追加できます）
private: false # true: 限定共有記事 / false: 公開記事
updated_at: "" # 記事を投稿した際に自動的に記事の更新日時に変わります
id: null # 記事を投稿した際に自動的に記事のUUIDに変わります
organization_url_name: null # 関連付けるOrganizationのURL名
slide: false # true: スライドモードON / false: スライドモードOFF
---
# new article body

```

プレビュー

```bash
npx qiita preview
```

公開

```bash
npx qiita publish [FileName]
```
