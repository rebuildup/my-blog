# Hello, World

これはサンプル記事のリポジトリ構造を説明するためのデモ用Markdownです。
見出し・段落・箇条書き・コードブロック・画像参照が一通り含まれているか確認するために用意しました。

## 本文の例

記事の本文は純粋なMarkdownとして書かれています。frontmatterは使わず、
別途 `meta.yaml` でメタデータを管理します。これにより、Qiita / Zenn / note.com /
個人サイトのいずれにも同じソースから変換・配信できます。

### 箇条書き

- 記事は `articles/<dir>/article.md` に置く
- メタデータは `meta.yaml` に置く
- 画像は `images/` に置く

### コードブロック

```typescript
// 例: 個人サイトのロード処理イメージ
import { readFileSync } from "node:fs";
import { join } from "node:path";

function loadArticle(articleDir: string) {
  const body = readFileSync(join(articleDir, "article.md"), "utf8");
  const meta = readFileSync(join(articleDir, "meta.yaml"), "utf8");
  return { body, meta };
}
```

### 画像参照

画像を追加したら、こんな感じで参照します:

![アーキテクチャ図](images/architecture.png)

> 実際に画像ファイルを置く場合は `images/` に追加してください。
> このリポジトリではサンプル用画像は含めていません。

## まとめ

- 1記事 = 1ディレクトリ
- 本文は `article.md`、メタデータは `meta.yaml`
- 画像は `images/` に閉じて配置