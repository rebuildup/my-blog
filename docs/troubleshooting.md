# Troubleshooting

> よくある障害と復旧。

## Branch / merge conflict

### 症状

- `release-x-y-z` で複数 Worker が同じファイルへ競合する変更を入れた。
- PR に merge conflict。

### 復旧

1. Conflict 発生 PR の base を `release-x-y-z` 最新 tip に rebase。
2. conflict marker を解消。
3. CI (configured な場合) を再実行。
4. blocking review / conversation 解消を確認。
5. ready-to-merge 状態を report。

## Stack-PR landing 失敗

### 症状

- predecessor が release trunk 未到達。
- dependent PR が stack landing できない。

### 復旧

1. predecessor PR を先に ready-to-merge まで進める。
2. dependent PR の base を predecessor merged commit に更新。
3. landing が安全でない場合は dependency SoT を壊さず、
   predecessor merge 後の通常 ticket workflow へ縮退する。
4. Issue の status を `stack-ready` のまま据え置き、`integrated` 化を保留。

## AI agent / session 喪失

### 症状

- Worker session が crash / lost。
- Coordinator が context を失った。

### 復旧 (canonical path)

1. **fresh agent** を作成。
2. durable repo state から reconstruct:
   - `git log --all --oneline | head -50`
   - `gh pr list --state all --limit 50`
   - `gh issue list --state all --limit 50`
   - `release-x-y-z` branch tip
   - Issue dependency graph
3. native session / subagent resume は **高速経路** として並行試行可。
4. safe な最小 verification で reconstructed state を確認。
5. **recovery / handoff journal** は designated checkpoint へ (reader-facing prose には書かない)。

## CI false green 疑い

### 症状

- CI は pass しているが挙動が壊れている。

### 確認

- skipped test / `.only` / ignored exit code / `|| true` / blanket suppression / CI disabling がないか。
- smoke / integration が unit test だけで substitution されていないか。
- 外部 service への connectivity は独立に検証したか。

### 復旧

- 偽装箇所を撤去して再実行。
- 復旧完了まで ready-to-merge 状態にしない。

## Secret 漏洩疑い

### 症状

- secret value が repository / log / chat に露出した疑い。

### 即時対応

1. 露出 source を記録 (commit SHA / log timestamp / chat id)。
2. Infisical で該当 secret を即時 rotate。
3. machine identity の scope を再評価。
4. Issue を `security` ラベルで起票、`security-maintenance` の priority 判定へ。
5. 復旧まで release PR を block。
