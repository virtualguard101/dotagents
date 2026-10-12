Apply worktree changes to main.

Usage:
1. Determine the source worktree path. If unknown:
```bash
git worktree list
```
2. Pair source and main (main = that repo's primary checkout root):
```text
SOURCE_WORKTREE_PATH=<created worktree path>
MAIN_WORKTREE_PATH=<matching repo root>
```
3. If `MAIN_WORKTREE_PATH` is omitted, set it to `(git rev-parse --show-toplevel)` with shell cwd inside the main checkout.
4. Run one of the blocks below (substitute paths). Abort if `git-common-dir` differs. If there are no tracked or untracked changes, stop after printing that nothing applies.

Unix/macOS:
```bash
set -euo pipefail
SOURCE_WORKTREE_PATH="<source>"
MAIN_WORKTREE_PATH="<main>"
SOURCE_ABS="$(cd "$SOURCE_WORKTREE_PATH" && pwd -P)"
MAIN_ABS="$(cd "$MAIN_WORKTREE_PATH" && pwd -P)"
SOURCE_COMMON="$(cd "$(git -C "$SOURCE_ABS" rev-parse --git-common-dir)" && pwd -P)"
MAIN_COMMON="$(cd "$(git -C "$MAIN_ABS" rev-parse --git-common-dir)" && pwd -P)"
if [ "$SOURCE_COMMON" != "$MAIN_COMMON" ]; then echo "ERROR: source and main do not share the same repository." >&2; exit 1; fi
UNTRACKED_FILES="$(git -C "$SOURCE_ABS" ls-files --others --exclude-standard)"
HAS_TRACKED=false
if [ -n "$(git -C "$SOURCE_ABS" diff --name-only HEAD)" ] || [ -n "$(git -C "$SOURCE_ABS" diff --cached --name-only)" ]; then HAS_TRACKED=true; fi
if [ "$HAS_TRACKED" = false ] && [ -z "$UNTRACKED_FILES" ]; then echo "Nothing to apply."; exit 0; fi
if [ "$HAS_TRACKED" = true ]; then
  git -C "$SOURCE_ABS" add -u -- .
  git -C "$SOURCE_ABS" commit -m "tmp: worktree apply snapshot" --no-verify --allow-empty
  TEMP_COMMIT="$(git -C "$SOURCE_ABS" rev-parse HEAD)"
  git -C "$MAIN_ABS" cherry-pick --no-commit "$TEMP_COMMIT" || true
  git -C "$MAIN_ABS" reset HEAD -- . >/dev/null 2>&1 || true
  git -C "$SOURCE_ABS" reset --mixed HEAD~1 >/dev/null 2>&1
fi
if [ -n "$UNTRACKED_FILES" ]; then echo "$UNTRACKED_FILES" | while IFS= read -r f; do [ -z "$f" ] && continue; mkdir -p "$(dirname "$MAIN_ABS/$f")"; cp "$SOURCE_ABS/$f" "$MAIN_ABS/$f"; done; fi
git -C "$MAIN_ABS" status --short
echo "MAIN_WORKTREE=$MAIN_ABS"
echo "SOURCE_WORKTREE=$SOURCE_ABS"
```

Windows (PowerShell):
```powershell
$ErrorActionPreference = 'Stop'
$SourceAbs = (Resolve-Path -LiteralPath "<SOURCE_WORKTREE_PATH>").Path
$MainAbs = (Resolve-Path -LiteralPath "<MAIN_WORKTREE_PATH>").Path
$SourceCommon = (Resolve-Path (git -C $SourceAbs rev-parse --git-common-dir).Trim()).Path
$MainCommon = (Resolve-Path (git -C $MainAbs rev-parse --git-common-dir).Trim()).Path
if ($SourceCommon -ne $MainCommon) { Write-Error "source and main do not share the same repository."; exit 1 }
$untrackedRaw = git -C $SourceAbs ls-files --others --exclude-standard
$untrackedFiles = @(); if ($untrackedRaw) { $untrackedFiles = $untrackedRaw -split "`n" | Where-Object { $_.Trim() -ne '' } }
$hasTracked = ((git -C $SourceAbs diff --name-only HEAD) -or (git -C $SourceAbs diff --cached --name-only))
if (-not $hasTracked -and $untrackedFiles.Count -eq 0) { Write-Output "Nothing to apply."; exit 0 }
if ($hasTracked) {
  git -C $SourceAbs add -u -- .
  git -C $SourceAbs commit -m "tmp: worktree apply snapshot" --no-verify --allow-empty
  $tempCommit = (git -C $SourceAbs rev-parse HEAD).Trim()
  try { git -C $MainAbs cherry-pick --no-commit $tempCommit } catch {}
  git -C $MainAbs reset HEAD -- . 2>$null
  git -C $SourceAbs reset --mixed HEAD~1 2>$null
}
foreach ($f in $untrackedFiles) {
  $dst = Join-Path $MainAbs $f
  $dstDir = Split-Path $dst -Parent
  if (-not (Test-Path $dstDir)) { New-Item -ItemType Directory -Path $dstDir -Force | Out-Null }
  Copy-Item -Path (Join-Path $SourceAbs $f) -Destination $dst -Force
}
git -C $MainAbs status --short
Write-Output "MAIN_WORKTREE=$MainAbs"
Write-Output "SOURCE_WORKTREE=$SourceAbs"
```

Rules:
- Apply each repo worktree independently.
- The main worktree does not need to be clean.
- After applying, ask the user if they want to delete the source worktree (see `/delete-worktree`).
