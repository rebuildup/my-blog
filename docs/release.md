# Release

> release PR workflow, version bump policy, merge authorization boundary.

## Cadence

- 1 sprint = **1 週間 (planning cadence)**。
- 工期保証ではない。release date / roadmap / milestone / capacity / agent 数効果を見積もる場合は
  **`agent-delivery-estimation` Skill** を使う。
- material unknown を任意の保守値で埋めず、必要なら `complete / conditional / unavailable` を返す。

## Version bump policy (default)

| Change profile | Bump |
|---|---|
| production / stable cut | **major** |
| 通常 sprint | **minor** |
| hotfix / typo / 微調整 | **patch** |

例外は ADR で明示。

## Release branch

```
release-<major>-<minor>-<patch>
```

例: `release-0-1-0`, `release-1-2-0`。

## Release PR (`release-x-y-z -> main`)

### 性質

- **唯一の `main` 正規 delivery path**。
- ticket / arbitrary branch からの merge は禁止。
- source branch 制約のために存在しない required status check を **捏造しない**。

### Review / approval baseline

- required approving review count: **0**
- blocking review / unresolved conversation は **解消必要**
- conversation resolution: **required**
- 固定 required status checks: **none by default**

### Evidence

- **CI / workflow run が available な repo**:
  - native CI checks は validation evidence の一部として current SHA との対応と失敗有無を確認。
  - CI success や特定 check 名を ready-to-merge の普遍的必須条件にしない。
  - validated SHA pinned reference と workflow run status の PR 本文への pin / 転写は
    **reader が evidence を再 fetch する必要が生じた場合に限る**。
- **CI が configured でない repo (policy / docs only 等)**:
  - validated SHA pinned reference と `evals/` 配下の canonical control reproduction block を
    release evidence として本文へ残す。
  - `evals/` controls は fresh agent / reviewer / CI runner から再取得できる形で残す。

### Merge authorization (ADR-0012)

- `release-x-y-z -> main` PR merge は **explicit user authorization 境界**。
- Agent は release-wide verification 完了 + ready-to-merge 状態まで進めた時点で **ready-to-merge で停止**。
- 現状 (head SHA / validation evidence / outstanding review conversations) を report。
- **authorization 取得のためだけの追加質問は禁止** (permission 確認は user 側の発火に委ねる)。
- merge そのものは **user が明示的に authorization した時にのみ** 実行。
- authorization 後に PR を land する場合は `merge` method を明示し、merge commit を生成。

### False-green 禁止

- skipped test / `.only` / ignored exit code / `|| true` / blanket suppression / CI disabling 等で green を偽装しない。
- unit test だけで smoke / integration correctness を証明した扱いにしない。
