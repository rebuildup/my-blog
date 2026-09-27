# my-blog

複数プラットフォーム(Qiita / Zenn / note.com)に同時公開する記事を一元管理するためのリポジトリ。
個人サイト(将来的にTanstack Start / Astro)からもサブモジュールとして取り込み可能。

## ディレクトリ構造

```
my-blog/
├── .gitignore
├── README.md
└── articles/
    └── YYYY-MM-DD-<slug>/
        ├── article.md      # 記事本文(純粋なMarkdown)
        ├── meta.yaml       # 記事のメタデータ
        └── images/         # 記事で使用する画像・素材
```

- 1記事 = 1ディレクトリ
- 記事ディレクトリ名は `YYYY-MM-DD-<slug>/` 形式。`YYYY-MM-DD` は作成日、`<slug>` はURL等で使う小文字ハイフン繋ぎの識別子
- 画像等の素材は記事ディレクトリ内の `images/` に置く(記事と一緒にコピー/移動しやすいように)

## 新しい記事の追加手順

1. `articles/YYYY-MM-DD-my-topic/` のようなディレクトリを作る
2. 中に `article.md`(Markdown本文)と `meta.yaml`(メタデータ)を作成
3. 画像を使う場合は `images/` ディレクトリに置く
4. 内容はMarkdownで書き、メタデータは次のスキーマに沿って `meta.yaml` に書く

## `meta.yaml` のスキーマ

```yaml
# 記事自体のメタデータ
title: "記事タイトル"
slug: my-topic              # URL等に使用する識別子
created: 2026-09-27         # 作成日 (ISO 8601)
updated: 2026-09-27         # 最終更新日 (ISO 8601)
status: published           # draft | published | archived
tags:
  - tech
  - example

# プラットフォーム別の公開情報
publish:
  qiita:
    published: true
    url: https://qiita.com/xxx/items/yyy
    published_at: 2026-09-27
  zenn:
    published: true
    url: https://zenn.dev/xxx/articles/yyy
    published_at: 2026-09-27
  note:
    published: false

# 著者のメモ(自由記述)
notes:
  audience: beginner
```

- `publish.<platform>.published` でプラットフォーム別の公開状態を管理する
- プラットフォーム固有の追加設定が必要になったら、各 `publish.<platform>` セクション内に自由キーで追加してよい
- `slug` は `article.md` のファイル名やディレクトリ名と揃えるのを推奨

## プラットフォームへの公開ワークフロー

当面は手動運用を前提とする:

1. `article.md` をプラットフォームにコピペして公開する
2. 公開後、`meta.yaml` の `publish.<platform>` に `url` と `published_at` を記録する
3. `status: draft` のままだと未公開。公開したら `status: published` に更新する

## 個人サイトからサブモジュールとして取り込む

個人サイト側(Tanstack Start / Astro)で記事を取り込む例:

```bash
# 個人サイトのリポジトリ内で
git submodule add <my-blogのURL> content/blog-source
```

- `content/blog-source/articles/<dir>` をそのままコピー、もしくは読み込んで使う
- `article.md` は純粋なMarkdownなので、frontmatterの追加やSSG固有の形式変換は取り込み側で行う
- 画像パスは記事ディレクトリからの相対パス(`images/foo.png`)で書かれているため、コピー先でもそのまま使える

## Conventions(命名・運用ルール)

- 記事ディレクトリ名: `YYYY-MM-DD-<slug>/` 形式
- スラッグ: 半角小文字英数字とハイフン
- 画像ファイル名: 内容を示す名前(例: `architecture-diagram.png`)
- コミットメッセージ: 1記事 = 1コミットを推奨(粒度の参考)

将来ここに運用ルールを追記していく。