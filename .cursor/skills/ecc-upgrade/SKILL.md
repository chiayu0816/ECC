---
name: ecc-upgrade
description: >-
  Upgrade Roy's ECC fork from upstream affaan-m/ECC, preserve Cursor user-home
  hook path fixes, push to chiayu0816/ECC, then reinstall into ~/.cursor.
  Use when the user says 更新 ECC, upgrade ECC, sync ECC, update everything-claude-code,
  refresh ECC into ~/.cursor, or asks to pull latest ECC and reinstall Cursor config.
---

# ECC Upgrade (fork → ~/.cursor)

Run this end-to-end without asking for confirmation unless a merge conflict needs a decision or git push fails.

## Constants

| Item | Value |
|------|-------|
| Local repo | `/Users/roy/Workspace/ECC` |
| Upstream remote | `origin` → `affaan-m/ECC` |
| Fork remote | `fork` → `chiayu0816/ECC` |
| Install target | `~/.cursor` (run install from `$HOME`) |
| Languages | `golang java javascript kotlin python typescript` |

Preserve these on merge conflicts (Cursor user-home hook path safety):

- `.cursor/hooks/adapter.js`
- `.cursor/hooks/before-shell-execution.js`
- `.cursor/hooks/before-shell-execution-block-no-verify.js`
- `scripts/lib/install/apply.js`
- `scripts/lib/install/cursor-hooks-json.js`
- `tests/scripts/cursor-hooks-json.test.js`

## Workflow

### 1. Sync fork with upstream

```bash
cd /Users/roy/Workspace/ECC
git remote -v   # expect origin=affaan-m/ECC, fork=chiayu0816/ECC
git fetch origin
git fetch fork
git checkout main
git merge origin/main
```

- Prefer `merge` over `rebase` unless the user asks for rebase.
- If conflicts touch the preserve list above, **keep the fork/path-safe versions** (layout-aware `getPluginRoot`, `../scripts` via `getPluginRoot`, install rewrite to `node ./hooks/`).
- If unrelated conflicts, resolve normally toward upstream intent while keeping the path-safe hook behavior.

```bash
git push fork main
```

### 2. Install from local fork checkout into ~/.cursor

Source must be `/Users/roy/Workspace/ECC` after step 1 (never install from a vanilla upstream tree that lacks the path-safe commits).

```bash
cd /Users/roy && /Users/roy/Workspace/ECC/install.sh --target cursor --enable-hooks \
  golang java javascript kotlin python typescript
```

### 3. Verify hooks still path-safe

```bash
# hooks.json must use user-home commands
rg 'node \./hooks/' /Users/roy/.cursor/hooks.json
# must NOT still use project-style paths for user hooks
rg 'node \.cursor/hooks/' /Users/roy/.cursor/hooks.json || true

# adapter resolves to ~/.cursor
node -e "console.log(require('/Users/roy/.cursor/hooks/adapter.js').getPluginRoot())"

# shell hooks load
printf '%s' '{"command":"git status"}' | node /Users/roy/.cursor/hooks/before-shell-execution.js >/dev/null
printf '%s' '{"command":"git commit --no-verify -m x"}' | node /Users/roy/.cursor/hooks/before-shell-execution-block-no-verify.js >/dev/null; echo "block_exit=$?"
```

Expect: `getPluginRoot()` → `/Users/roy/.cursor`; block hook exit `2` for `--no-verify`.

### 4. Report

Briefly tell the user:

- previous vs new upstream SHA / version (from `git log -1` and `ecc-install-state.json`)
- merge clean or what was preserved on conflict
- install completed into `~/.cursor`
- hook path verification pass/fail

## Do NOT

- Install from `affaan-m/ECC` without the path-safe commits
- `git reset --hard origin/main` on fork `main` (drops the hook path fix)
- Force-push fork `main` unless the user explicitly asks
- Open an upstream PR unless the user asks
- Modify `~/.cursor/skills-cursor/`
