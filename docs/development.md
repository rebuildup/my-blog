# Development

> branch / commit / handoff / agent rules for local development.

## Default branch & merge policy

- default branch: **`main`**
- PR merge method: **`merge`** (merge commit のみ)
  - `allow_squash_merge=false`
  - `allow_rebase_merge=false`
- direct push to `main`: 禁止
- direct web edit on `main`: 禁止

## Branch model

```
main                          (protected, no direct push)
└── release-<major>-<minor>-<patch>     (active sprint)
    ├── feat/<scope>-<ticket>          (Worker branch)
    ├── fix/<scope>-<ticket>
    └── chore/<scope>-<ticket>
```

- `release-x-y-z` が **active sprint trunk**。
- Worker は `release-x-y-z` から分岐し、PR で戻す。
- 例外 (hotfix): `release-x-y-z` から直接 `fix/<scope>-<ticket>` を分岐可。

## Commit / handoff rules

- commit は **小さく・意味単位で・頻繁に**。
- 1 Worker の commit message は CLAUDE.md / README の persistent 規約に従う (writing-discipline 適用)。
- Worker に remote publication / PR mutation 権限がない場合:
  - first meaningful commit 後 **即座に Coordinator へ handoff**。
  - Coordinator が publish + remote head SHA 確認 + Draft PR 作成完了するまで追加 implementation を進めない。

## AI agent config: project-local

project-local AI agent 設定 (`.claude/`) を **原則** とし、host-level install を **禁止**:

- 禁止する host-level install スコープ:
  - `.claude/settings.json` を `~/.claude/` 配下に書く行為
  - `parallel-orchestration` / `sandbox-runtime` / `github-delivery` 等の Skill / agent 定義を host-level に置く行為
  - project-local `.claude/skills/` の代わりに `~/.claude/skills/` を使う運用
- 必要に応じ `bunx skills add <source>` / `bunx skills add <source> --skill <name>` で project-local へ導入する。
- `--global` は明示的に必要性が立証されない限り使用しない。

## Skills discovery & install

```bash
# candidate 確認
bunx skills add <source> --list

# project-local 導入
bunx skills add <source>
bunx skills add <source> --skill <name>
```

- 既存 `skills/` と repository policy を優先確認。
- source / trust / maintenance / reproducibility を評価してから導入。
- 既存 Skill は presence だけで skip せず、freshness を確認して reconcile。

## Tooling

- Bun 利用可能 → `bunx` を優先。
- Node.js fallback → `npx`。
- 自動化 / migration / validation script は **TS / JS / shell / PowerShell** のみ。**新しい `.py` script を追加しない**。
- ローカル validation entry point: `scripts/validate.ts` (導入時定義) または `package.json` script。

## Code conventions

- ファイル名: kebab-case
- ディレクトリ: kebab-case
- Markdown: 1 行目は `# ` 始まりではなく本文 (frontmatter は YAML)。

## Recovery from agent / session loss

- canonical recovery path は **fresh agent が durable repo state から reconstruct**。
- native session / thread / subagent resume は **高速経路** であり canonical ではない。
- recovery 時の minimum verification:
  - current branch と remote head SHA 一致
  - `release-x-y-z` branch tip の整合
  - `gh pr list --state open` で active PR 確認
  - Issue dependency graph と open PR の対応
