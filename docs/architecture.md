# Architecture

> durable architectural overview. Multi-agent runtime / data flow / non-negotiable boundaries.

## High-level shape

```
+------------------------------+
|  my-blog repository (main)   |
+------------------------------+
              |
              |  git submodule (将来)
              v
+------------------------------+
|  Personal site (TanStack     |
|  Start / Astro / etc.)       |
+------------------------------+
              |
              | 同期 publish (manual / CI 将来)
              v
+------------------------------+      +-----------+      +-----------+
| Qiita                       |      | Zenn      |      | note.com  |
+------------------------------+      +-----------+      +-----------+
```

source of truth は **repository の `articles/`**。
各プラットフォームは **consumer** で canonical ではない。

## Repo layout

```
my-blog/
├── README.md                # 記事 schema / 運用規約 (既存)
├── CLAUDE.md                # durable project entry point
├── .gitignore
├── articles/                # 記事の canonical home
│   └── YYYY-MM-DD-<slug>/
│       ├── article.md       # 純粋な Markdown 本文
│       ├── meta.yaml        # 記事 metadata + platform publish state
│       └── images/          # 記事で使う画像
├── docs/                    # durable reader-facing docs
│   ├── architecture.md
│   ├── development.md
│   ├── release.md
│   ├── security.md
│   ├── troubleshooting.md
│   └── adr/
├── .claude/                 # project-local AI agent config
│   ├── settings.json
│   └── agents/              # 任意 (必要時に追加)
├── skills/                  # project-local Skills (必要時に追加)
├── scripts/                 # validation / generation (TS / shell / PowerShell only)
└── evals/                   # validation control reproduction
```

## Multi-agent runtime

### Roles

- **Root Coordinator**
  - durable branch remote publication / Draft PR lifecycle coordination
  - logs / status / preview routing
  - durable branch identity / expected Draft PR identity 管理
- **Worker**
  - implementation / refactor / test / migration / generation / runtime verification
  - **独立 mutable environment** 必須 (自分の worktree / branch / container)

### Handoff rule

remote publication / PR mutation 権限がない Worker は:

1. first meaningful commit を打つ。
2. **ただちに** Coordinator へ handoff (追加 implementation は Coordinator が publish + remote head SHA 確認 + Draft PR 作成を完了するまで停止)。
3. Coordinator は publish 完了 + SHA 確認 + Draft PR 作成後に Worker へ resume 信号を送る。

### Authority boundaries

| Decision | Authority |
|---|---|
| `release-x-y-z -> main` PR merge | **user only** (ADR-0012) |
| non-release branch force push / reset | Worker (own branch のみ) |
| Issue close / status transition | Worker (assignee) または Coordinator |
| label / reviewer / CODEOWNERS 編集 | Coordinator / user |
| branch protection / repo settings | user only |
| secret rotation / Infisical project 変更 | user only |

### Execution environment isolation

Worker は少なくとも以下を **独立 mutable 環境** で扱う:

- working tree (branch / worktree)
- dependency install (`node_modules` 等)
- local port (preview / dev server)
- `.env` / secret injection

同一 environment を Worker 間で共有しない。

## Data flow

```
article.md  ──►  meta.yaml  ──►  publish.<platform>.{published,url,published_at}
                                ──►  status: draft | published | archived
```

- `article.md` は純粋 Markdown で platform 固有 syntax を持たない。
- `meta.yaml` の `publish.<platform>` セクションが platform 別の canonical 公開状態。
- frontmatter / SSG 固有の形式変換は **取り込み側** (個人サイト) で行う。

## Versioning

- 通常 sprint = minor bump。
- 1 sprint = 1 target semantic version (`release-<major>-<minor>-<patch>` branch)。
- production cut は major bump。
- patch bump は hotfix / typo / 微調整のみ。
