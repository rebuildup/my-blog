# evals/release-pr-controls

> Canonical control reproduction block for `release-0-1-0 -> main` release PR validation.
> Per project policy (`docs/release.md`): CI-unconfigured repository の release evidence は
> validated SHA pinned reference と `evals/` 配下の canonical control reproduction block を
> 本文へ残す。

## Purpose

A fresh agent / reviewer / CI runner MUST be able to re-execute these controls
without prior context and observe whether the current state matches the policy baseline.

## Controls

### 1. default branch

```bash
gh repo view rebuildup/my-blog --json defaultBranchRef --jq '.defaultBranchRef.name'
```

Expected: `main`

### 2. repository merge settings

```bash
gh api repos/rebuildup/my-blog \
  --jq '{allow_merge_commit, allow_squash_merge, allow_rebase_merge}'
```

Expected:

```json
{ "allow_merge_commit": true, "allow_squash_merge": false, "allow_rebase_merge": false }
```

### 3. main branch protection

```bash
gh api repos/rebuildup/my-blog/branches/main/protection \
  --jq '{required_pull_request_reviews, enforce_admins, required_conversation_resolution, allow_force_pushes, allow_deletions, required_linear_history}'
```

Expected (key fields):

| Field | Expected value |
|---|---|
| `required_pull_request_reviews.required_approving_review_count` | `0` |
| `enforce_admins.enabled` | `true` |
| `required_conversation_resolution.enabled` | `true` |
| `allow_force_pushes.enabled` | `false` |
| `allow_deletions.enabled` | `false` |
| `required_linear_history.enabled` | `false` |

### 4. branch inventory

```bash
git ls-remote origin
```

Expected: only `main`, `release-0-1-0` and any prior-merged tags (no stray branches).

> Note: at the time this block was authored the remote also contained a stray
> branch `refs/heads/1` pointing to the same commit as `release-0-1-0`. This is a
> transient artifact and is expected to be cleaned up; this control is informational
> until `1` is removed.

### 5. policy framework file presence (commit-pinned)

```bash
git ls-tree -r <release-sha> --name-only \
  | grep -E '^(CLAUDE\.md|docs/|evals/|\.claude/settings\.json|\.editorconfig)$'
```

Expected: every entry listed in the file inventory above appears under the pinned SHA.

### 6. policy file content checks

```bash
git show <release-sha>:CLAUDE.md | grep -E '^## [0-9]+\. '
```

Expected: 10 top-level sections numbered `## 1.` through `## 10.`.

```bash
git show <release-sha>:docs/adr/0012-release-authorization-boundary.md \
  | grep -E '^## Decision$'
```

Expected: section `## Decision` is present (ADR-0012 anchor).

## How to invoke

A reviewer can run all controls in one pass:

```bash
./scripts/run-release-evals.sh <release-sha>
```

> The wrapper script is not yet committed; reviewers may run the controls above
> directly. A future Sprint may promote this block into a checked-in script
> (`scripts/run-release-evals.{sh,ts,ps1}`) once project tooling matures.

## Re-runnability

These controls depend only on:

- `gh` CLI authenticated to a principal with read access to the repo
- `git` (read-only operations)
- network access to `https://api.github.com`

No local state, no chat history, no private memory is required.
