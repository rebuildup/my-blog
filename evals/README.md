# evals/

Validation control reproduction artifacts (policy: CI-unconfigured repository の release evidence).

## Purpose

For policy / docs-only repositories where native CI / GitHub Actions workflow runs are not configured,
this directory hosts **canonical control reproduction blocks** that a fresh agent / reviewer / CI runner
can re-fetch and re-run to validate release-candidate state.

## Conventions

- One subdirectory per validation concern (e.g., `evals/branch-protection/`, `evals/merge-method/`).
- Each block MUST be self-contained: explicit inputs, expected outputs, command invocations.
- Re-runnability is mandatory; a fresh agent with no prior context MUST be able to execute the block.
- Result artifacts (timestamps, exit codes, observed values) are recorded separately and pinned to the
  validating SHA -- do NOT bake ephemeral results into reader-facing prose.

## When this directory is required

- Repository with no native CI workflow configured.
- `release-x-y-z -> main` PR whose body references `evals/<block>` evidence (per docs/release.md).

## When this directory is optional

- Repository with native CI / GitHub Actions workflow runs providing equivalent evidence.
- The PR body may then cite the workflow run URL + SHA pinning instead of `evals/` blocks.
