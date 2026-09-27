# CLAUDE.md

> AI エージェント / 開発者向けの **プロジェクトの入口 (durable entry point)**。
> Project knowledge は **repository-controlled docs** に置き、chat/private memory に依存しない。
> 詳細な判断根拠・運用ルールは `docs/` 配下を参照。

## 1. Project summary

- **名前**: `my-blog`
- **目的**: Qiita / Zenn / note.com 等複数プラットフォームへ同時公開する記事を一元管理する。
- **成果物ルート**: `articles/YYYY-MM-DD-<slug>/{article.md, meta.yaml, images/}`
- **取り込み先**: 個人サイト (TanStack Start / Astro 等) から submodule として参照可能。
- **既存規約**: `README.md` のディレクトリ構造 / `meta.yaml` schema / 命名規則を引き継ぐ。

## 2. Default branch / merge policy

- **default branch**: `main` (policy standard。legacy `master` は初期化時に `main` へ rename する)。
- **`main` への正規 delivery path**: `release-<major>-<minor>-<patch> -> main` の release PR **だけ**。
- **branch protection baseline**:
  - Pull Request required: yes
  - required approving review count: 0
  - conversation resolution: required
  - required status checks (固定): none by default
  - direct push / direct web edit / force push / deletion: 通常運用で禁止
- **PR merge method**: `merge` (merge commit のみ)。
  - repository settings: `allow_merge_commit=true` / `allow_squash_merge=false` / `allow_rebase_merge=false`
  - merge executor は method を **明示** する。
- **authorization boundary**: `release-x-y-z -> main` PR の merge は **explicit user authorization** が必要。
  - 詳細は `docs/adr/0012-release-authorization-boundary.md`。

## 3. Sprint / release cadence

- **planning cadence**: 1 sprint = **1 週間**。
- **active sprint branch**: `release-<major>-<minor>-<patch>`。
  - **version bump policy (default)**:
    - production / stable cut → **major** bump
    - 通常 sprint → **minor** bump
    - 微調整 / hotfix → **patch** bump
- 1 週間は **planning cadence** であり、工期保証ではない。
- release date / roadmap / milestone / capacity / agent 数の効果を見積もる場合は
  `agent-delivery-estimation` Skill を使う (project knowledge を主観日数で埋めない)。

## 4. Durable state / canonical SoT

| Concern | Canonical SoT |
|---|---|
| durable ticket | **GitHub Issue** |
| durable work / dependency state | **GitHub Issues** (dependency graph) |
| release planning / health / portfolio | **Linear** |
| GitHub Projects (standard) | **使用しない** |
| secret value | **Infisical** (`https://secrets.rebuildup.dev`, API `https://secrets.rebuildup.dev/api`) |
| secret schema / required key / non-secret metadata / provider endpoint / project ID / env mapping | **repository** (`docs/`, `.env.example`, `docs/security.md`) |
| git branch topology | dependency SoT として **扱わない** (Issue dependency graph が canonical) |

## 5. Stack-aware delivery model

- Issue status を **2 種類** 区別する:
  - `stack-ready`: reviewable immutable predecessor snapshot が存在し dependent work 開始可能
  - `integrated`: ticket changes が target release trunk へ land 済み
- native stacked PR では **contiguous stack landing** で target release trunk へ到達した ticket だけ `integrated` (Done) にする。
- ordinary nested PR fallback では `124 -> 123` のような intermediate predecessor merge だけで Issue #124 を close/Done にしてはいけない。
- stacked delivery を安全に維持できない場合は dependency SoT を壊さず predecessor merge 後の通常 ticket workflow へ縮退する。

## 6. Multi-agent coordination

- **Root Coordinator**: durable branch remote publication / Draft PR lifecycle coordination。
- **Worker**: implementation / refactor / test / migration / generation / runtime verification。**独立 mutable environment** 必須。
- remote publication / PR mutation 権限がない Worker は first meaningful commit 後ただちに Coordinator へ handoff。
- 詳細: `docs/development.md` / `docs/architecture.md`。

## 7. AI agent config convention

- AI agent 関連設定は **project-local** 原則。
- **禁止** (project-local 設定の host-level install): 詳細スコープは `docs/development.md` 参照。
- Skills 導入: `bunx skills` (Bun 不在時 `npx skills`)。`--global` を既定にしない。
- 既存 `skills/` / repository policy を優先確認し、source / trust / maintenance / reproducibility を評価してから導入。
- 既存 Skill の presence だけで skip せず freshness を確認して reconcile。

## 8. Writing & docs discipline

- README / docs / ADR / Issue / PR / commit / code comment / release note 等の
  **persistent reader-facing prose** を作成・更新する前に `writing-discipline` Skill を load する。
- **recovery / handoff の operational state** は reader-facing prose に混在させない (designated checkpoint へ)。
- 既存実装は evidence だが、old / migration 途中 / generated / vendor / example を除外して canonical を読む (1 file だけで convention 断定しない)。
- **新しい `.py` script を automation / migration / validation 目的で追加しない**。project-native 言語 (TS/JS / shell / PowerShell) を使う。

## 9. Docs map

- `README.md` — プロジェクト入口 + 記事運用の現行規約 (記事 schema は既存)。
- `docs/architecture.md` — 全体構成 / multi-agent runtime / data flow。
- `docs/development.md` — ローカル開発 / branch / commit / handoff / agent rules。
- `docs/release.md` — release PR workflow / version bump / merge authorization。
- `docs/security.md` — Infisical / secret handling / identity scope。
- `docs/troubleshooting.md` — よくある障害と復旧。
- `docs/adr/` — 重要な architectural decision の記録 (canonical durable form)。
- `evals/` — validation control reproduction (policy: CI 未構成 repo の release evidence)。

## 10. Outstanding setup (manual)

> 外部サービス / GitHub 設定に依存し、ローカル clone だけ完了しない項目。

- [ ] GitHub repository の作成 (`gh repo create`) — main protection / merge settings 設定の前提。
- [ ] `main` branch protection / ruleset の有効化 (上記 baseline を `gh api` で適用)。
- [ ] PR merge method の repository settings 適用 (`allow_merge_commit=true` 他)。
- [ ] Infisical project 作成 + `https://secrets.rebuildup.dev` での machine identity / scoped OIDC。
- [ ] Linear workspace 連携。
- [ ] `agent-delivery-estimation` / `writing-discipline` / `github-delivery` / `parallel-orchestration` / `sandbox-runtime` / `worktree-workflow` / `security-audit` / `security-maintenance` / `secrets-management` Skill の評価と project-local 導入。
- [ ] `docs/adr/` の歴史番号整合 (現状 policy text は ADR-0012 を release authorization として参照)。
