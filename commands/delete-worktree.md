Delete git worktree for this task.

Usage:
1. Determine the worktree path to delete. If unknown, list worktrees:
```bash
git worktree list
```
2. Resolve the worktree path to an absolute path if possible.
3. Verify that the path is registered by checking `git worktree list --porcelain` for an exact `worktree <absolute-path>` entry.
4. Remove it forcefully, then prune stale metadata.

Unix/macOS:
```bash
WORKTREE_ABS="$(cd "$WORKTREE_PATH" 2>/dev/null && pwd -P || echo "$WORKTREE_PATH")"
git worktree list --porcelain | grep -Fxq "worktree $WORKTREE_ABS"
git worktree remove --force "$WORKTREE_ABS"
git worktree prune
```

Windows (PowerShell):
```powershell
$WorktreeAbs = (Resolve-Path -LiteralPath $WorktreePath -ErrorAction SilentlyContinue).Path
if (-not $WorktreeAbs) { $WorktreeAbs = $WorktreePath }
$registered = git worktree list --porcelain | Select-String "^worktree $([regex]::Escape($WorktreeAbs))$"
if (-not $registered) { Write-Error "ERROR: '$WorktreeAbs' is not a registered worktree for this repo."; exit 1 }
git worktree remove --force $WorktreeAbs
git worktree prune
Write-Output "DELETED=$WorktreeAbs"
```

Rules:
- Deletion is forceful by default (`git worktree remove --force`) and removes worktrees even with uncommitted changes.
- Report the result for each deleted worktree path to the user.
