---
description: "Delete merged git branches locally (and optionally remotely). Protects main, master, and release* branches by default."
---

## Git Merge Cleanup

You are cleaning up merged git branches. Parse the user's arguments from: $ARGUMENTS

### Argument Parsing

Arguments can be:
- **No arguments**: Delete all locally merged branches except `main`, `master`, `release*`, and the current branch.
- **A pattern** (e.g., `manish*`): Delete only merged branches matching this glob pattern (still protects `main`, `master`, `release*`, and current branch).
- **`--remote`** or **`-r`** flag: Also delete the matching branches from the remote (`origin`). Can be combined with a pattern.

Examples:
- `/git-merge-cleanup` — delete all merged local branches (except protected ones)
- `/git-merge-cleanup manish*` — delete only merged local branches starting with `manish`
- `/git-merge-cleanup --remote` — delete all merged local + remote branches
- `/git-merge-cleanup manish* --remote` — delete merged `manish*` branches locally and remotely

### Steps

1. **Identify the current branch and default branch:**
   ```bash
   git branch --show-current
   ```

2. **Fetch and prune remote tracking refs:**
   ```bash
   git fetch --prune
   ```

3. **List merged branches:**
   Run `git branch --merged` to get all locally merged branches. Filter out:
   - The current branch (marked with `*`)
   - `main`
   - `master`
   - Any branch matching `release*`

   If a **pattern** argument was provided, additionally filter to only include branches matching that pattern (use glob matching — e.g., `manish*` matches `manish/feat-x`, `manish-test`, etc.).

4. **Show the user what will be deleted** before doing anything. List the branches that will be removed. If `--remote` flag is set, also note that remote branches will be deleted. **Ask for confirmation before proceeding.**

5. **Delete local branches** (after user confirms):
   ```bash
   git branch -d <branch-name>
   ```
   Use `-d` (not `-D`) to only delete branches that are fully merged. If `-d` fails for a branch, report it and skip — do NOT force delete.

6. **Delete remote branches** (only if `--remote` or `-r` flag was provided, after user confirms):
   ```bash
   git push origin --delete <branch-name>
   ```
   Do this for each matching branch that also exists on the remote. Check remote existence first:
   ```bash
   git branch -r | grep "origin/<branch-name>"
   ```

7. **Report results**: Summarize how many branches were deleted locally and remotely, and list any that failed.

### Safety Rules

- NEVER delete `main`, `master`, `release*` branches, or the current branch.
- ALWAYS ask for user confirmation before deleting anything.
- Use `git branch -d` (safe delete) not `-D` (force delete).
- For remote deletion, only delete branches that actually exist on the remote.
- If no branches match, report "No merged branches found to clean up" and exit.
