# ADR-0012: Release Authorization Boundary (`release-x-y-z -> main` PR merge)

- **Status**: Accepted
- **Date**: 2026-09-27
- **Context**: `my-blog` project initialization with multi-agent parallel driving.

## Context

The project's canonical delivery path is the `release-<major>-<minor>-<patch> -> main` release PR.
Direct push, direct web edit, force push, and deletion on `main` are forbidden in normal operations.
The merge of a release PR is a **consequential decision** that affects the canonical trunk.

Multiple AI agents (Root Coordinator + Workers) may concurrently drive work, but the project policy
explicitly forbids authority-less actors from determining consequential decisions.

## Decision

The merge of any `release-x-y-z -> main` PR is an **explicit user authorization boundary**.

1. Required approving review count is **0** by default (no other human gate).
2. Blocking review / unresolved conversation MUST be resolved before merge.
3. The merge authority itself is held **only by the user**.
4. Agents may advance a release PR to **ready-to-merge state** and then MUST stop.
5. Agents MUST NOT solicit authorization through additional questions (permission confirmation is left
   to user-initiated prompts).
6. When the user authorizes a release PR, the merge method MUST be `merge` (merge commit).
   The merge executor MUST declare the method explicitly.
7. Squash merge and rebase merge remain disabled at the repository level
   (`allow_squash_merge=false`, `allow_rebase_merge=false`).

## Consequences

- Coordinators report ready-to-merge state with: head SHA, validation evidence,
  outstanding review / conversation status, and any `needs_validation` items.
- Coordinators do not block the user with authorization-elicitation prompts.
- User-triggered authorization is the only path from `ready-to-merge` to `merged`.
- A failed / stale authorization does not silently fall back; the PR returns to
  `ready-to-merge` (or earlier) state with a report.

## Notes

- This ADR is referenced by `CLAUDE.md` (default policy entry point) and `docs/release.md`
  (release workflow).
- Numbering (`0012`) is preserved from the project policy text even though this is the first ADR
  in this repository; subsequent ADRs will fill the gap or be renumbered by the user.
