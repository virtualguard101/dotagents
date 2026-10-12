Run task in a separate git worktree.

**Optional argument:** If the user wrote `/worktree branch=<git-ref>` or `/worktree <branch=<git-ref>>`, use that ref as `WORKTREE_START_REF` (any commit-ish Git accepts: branch name, tag, full SHA, `origin/main`, etc.). Before the create block: Unix/macOS `export WORKTREE_START_REF='<git-ref>'`; Windows PowerShell `$env:WORKTREE_START_REF = '<git-ref>'`. If omitted, the script defaults to `HEAD`.

Only do this if the user invoked `/worktree` or explicitly asked for a git worktree in this chat. On first touch for a repo, run the create block once with shell `working_directory` set to that repo's workspace folder, record `workspace folder -> REPO_ROOT -> WORKTREE_PATH`, and then use that `WORKTREE_PATH` for all later reads, edits, shell, and repo-local git in that repo until the user says to stop or uses `/apply-worktree` / `/delete-worktree`. Do not fall back to `REPO_ROOT` while that mapping exists.

Multi-root: keep one chat-wide mapping, reuse it if two workspace folders share a repo, and create at most one worktree per `REPO_ROOT` unless the user explicitly asks for another.

Use the create blocks below; do not hand-roll a different flow. Shared id: filesystem-safe `<NAME>` + random 8-char lowercase hex `<SUFFIX>` -> `WORKTREE_ID=<NAME>-<SUFFIX>` used for every repo in this chat.

Unix/macOS (multi-repo safe: each repo gets its own subfolder under `~/.cursor/worktrees/<WORKTREE_ID>/`):
```bash
set -euo pipefail
WORKTREE_ID="<NAME>-<SUFFIX>"
REPO_ROOT="$(git rev-parse --show-toplevel)"
REPO_BASENAME="$(basename "$REPO_ROOT")"
if command -v shasum >/dev/null 2>&1; then
  REPO_HASH="$(printf '%s' "$REPO_ROOT" | shasum -a 256 | cut -c1-12)"
else
  REPO_HASH="$(printf '%s' "$REPO_ROOT" | sha256sum | cut -c1-12)"
fi
REPO_KEY="${REPO_BASENAME}-${REPO_HASH}"
WORKTREE_SET_DIR="$HOME/.cursor/worktrees/$WORKTREE_ID"
WORKTREE_DIR="$WORKTREE_SET_DIR/$REPO_KEY"
mkdir -p "$WORKTREE_SET_DIR"
if [ -d "$WORKTREE_DIR" ]; then echo "ERROR: worktree directory already exists: $WORKTREE_DIR" >&2; exit 1; fi
WORKTREE_START_REF="${WORKTREE_START_REF:-HEAD}"
git worktree add --detach "$WORKTREE_DIR" "$WORKTREE_START_REF"
HEAD_COMMIT="$(git -C "$WORKTREE_DIR" rev-parse HEAD)"
echo "WORKTREE_ID=$WORKTREE_ID"
echo "REPO_KEY=$REPO_KEY"
echo "WORKTREE_PATH=$WORKTREE_DIR"
echo "REPO_ROOT=$REPO_ROOT"
echo "HEAD_COMMIT=$HEAD_COMMIT"
echo "WORKTREE_START_REF=$WORKTREE_START_REF"
```

Windows (PowerShell; one worktree path per WORKTREE_ID—use a distinct WORKTREE_ID per repo if you work in multiple repos):
```powershell
$ErrorActionPreference = 'Stop'
$Name = "<NAME>-<SUFFIX>"
$RepoRoot = (git rev-parse --show-toplevel).Trim()
$WorktreeDir = Join-Path $env:USERPROFILE ".cursor\worktrees\$Name"
if (Test-Path $WorktreeDir) { Write-Error "ERROR: worktree directory already exists: $WorktreeDir"; exit 1 }
$WorktreeStartRef = if ($env:WORKTREE_START_REF) { $env:WORKTREE_START_REF } else { "HEAD" }
git worktree add --detach $WorktreeDir $WorktreeStartRef
$HeadCommit = (git -C $WorktreeDir rev-parse HEAD).Trim()
Write-Output "WORKTREE_ID=$Name"
Write-Output "WORKTREE_PATH=$WorktreeDir"
Write-Output "REPO_ROOT=$RepoRoot"
Write-Output "HEAD_COMMIT=$HeadCommit"
Write-Output "WORKTREE_START_REF=$WorktreeStartRef"
```

After creation, machine-readable lines include:
```text
WORKTREE_ID=<shared id>
REPO_KEY=<repo-specific key; Unix only>
WORKTREE_PATH=<absolute path>
REPO_ROOT=<absolute repo root>
HEAD_COMMIT=<sha>
WORKTREE_START_REF=<ref used for git worktree add; HEAD if omitted>
```

**Setup is not optional to *check*; only optional to *run* when no config applies.** Immediately after each successful `git worktree add`, and before any other reads, edits, shell, or git against `WORKTREE_PATH`, you MUST complete the setup discovery and execution step below. Do not start task work, open files for editing, or run installs in the worktree until this step is done or you have explicitly verified there is nothing to run.

1. Search for `.cursor/worktrees.json` in `REPO_ROOT`, then `WORKTREE_PATH`. You must actually perform this lookup on disk for each new worktree (do not assume "no config" from memory or prior sessions). If a file exists at either path, use it to determine setup (if both exist, prefer the copy under `WORKTREE_PATH` when it defines `setup-worktree*` / `setup-worktree` keys; otherwise use `REPO_ROOT`).
2. If a config file exists and defines a setup for this OS, you MUST run it exactly once for this worktree. Prefer `setup-worktree-unix` / `setup-worktree-windows` when those keys exist; otherwise `setup-worktree` (string = one shell command, array = run commands in order). Resolve script paths relative to the config file's directory; execute with shell cwd = `WORKTREE_PATH` and export `ROOT_WORKTREE_PATH=<REPO_ROOT>`.
3. You may skip running commands only when: no `.cursor/worktrees.json` exists in either location, or the file(s) exist but contain no applicable setup keys for this platform. You MUST NOT skip because you assume dependencies are already installed or the repo "looks ready".

Keep for the chat (update as you add repos):
```text
workspace folder -> repo root -> worktree path
```

Setup failure -> stop and report. Do not switch branches or change the main working tree.

End: report shared `WORKTREE_ID` and each `WORKTREE_PATH`, `REPO_ROOT`, `HEAD_COMMIT`, and `WORKTREE_START_REF`; state whether worktree setup ran, was skipped after checking both paths, or failed. Tell the user merge-back is `/apply-worktree` (per repo) and cleanup is `/delete-worktree` (full scripts there).
