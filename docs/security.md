# Security

> secret handling, identity scope, and security-audit handoff.

## Secret management

### Default

- **Infisical** を current default SoT。
  - control plane: self-hosted `https://secrets.rebuildup.dev`
  - API: `https://secrets.rebuildup.dev/api`
- encrypted secret-in-Git が **explicit requirement** でない限り Infisical を default とする。

### Repository に書くもの (discoverable, non-secret)

- secret schema (key 名)
- required key 一覧
- non-secret metadata (用途 / consumer)
- provider endpoint
- project ID
- environment / path mapping

### Repository に書かないもの

- secret value そのもの
- API token / personal access token
- private key / signing material

### 通常操作の原則

- **CLI-first** (Infisical CLI)
- **runtime injection** を優先 (process env / file mount)
- wrapper / CI は endpoint を明示し managed Cloud へ **暗黙 fallback させない**
- GitHub Actions は可能なら self-host 側の **OIDC + scoped Machine Identity** を使用
- 詳細は `secrets-management` Skill へ progressive disclosure

### Identity scope

- machine identity は **scope を最小化**。
- project ごとに identity を分離。
- rotation は release cadence と独立して必要時に実施 (drift 検知で起動)。

## Security audit (project に必要な場合)

未知の security defect を能動的に探す必要がある project では
`security-audit` Skill を導入する。

### Coverage unit の作り方

- source-visible principal / trust boundary / entry surface から unit を作成。
- candidate を hunter 自身で確定せず fresh verifier へ渡す。
- source 外の deployment / provider / runtime fact が決定的なら推測せず `needs_validation` とする。

### Audit 中の execution

- target-controlled execution は `sandbox-runtime` の bounded local isolation に従う。
- confirmed finding は `security-maintenance` の project-aware priority 判定を経て
  通常の Issue / remediation / quality / release workflow へ handoff。

## Vulnerability disclosure (外部向け)

- 外部報告窓口は **organization 単位の security policy** に従う。
- 報告受領 → triage → reproduction (sandbox) → fix branch → release PR の流れ。
- 詳細は organization の disclosure policy を参照 (本 repo では雛形のみ)。

## GitHub repository visibility

- public / private の選択は **組織ポリシー** に従う。
- public repo の場合:
  - `main` protection / ruleset 有効
  - direct push / edit 禁止
  - release PR だけが正規更新経路
- private repo の場合:
  - access scope を最小化
  - 外部 contributor は fork 経由のみ
