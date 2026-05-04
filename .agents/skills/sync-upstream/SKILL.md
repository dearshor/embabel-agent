---
name: sync-upstream
description: Sync the fork's main branch with upstream/main and update developer-guide/ docs. Use this skill whenever the user asks to sync upstream changes, merge upstream, update the fork, pull from upstream, or maintain the repository against its upstream source. Also trigger when the user mentions "upstream", "sync", or "fork maintenance" in the context of this repository.
---

# Sync Upstream into Fork

Merge new commits from `upstream/main` into the fork's `main` branch, optionally update `developer-guide/` to reflect upstream API/module/config changes, and open a PR for the result.

## Remotes

| Remote   | URL                                                    | Role        |
|----------|--------------------------------------------------------|-------------|
| origin   | https://github.com/dearshor/embabel-agent.git          | The fork    |
| upstream | https://github.com/embabel/embabel-agent.git           | The source  |

## Workflow

Execute steps 1–11 in order. Stop early where indicated.

### 1. Verify remotes

```bash
git remote -v
```

Confirm `origin` and `upstream` match the table above. Add/fix if missing.

If the repo is a partial clone (`git config remote.origin.partialclonefilter` returns non-empty), configure upstream as a promisor remote before any fetch:

```bash
git config remote.upstream.promisor true
git config remote.upstream.partialclonefilter "$(git config remote.origin.partialclonefilter)"
```

### 2. Fetch latest refs

```bash
git fetch upstream main
git fetch origin main
```

### 3. Check for new upstream commits

```bash
git log --oneline origin/main..upstream/main
```

If the output is empty, upstream/main is NOT ahead of origin/main — **stop here, no sync needed**.

### 4. Create a sync branch (no tracking)

Generate a timestamp and create the branch. The `--no-track` flag is critical — it prevents the branch from tracking `origin/main`, which would cause `git push` to push directly to main instead of creating a new remote branch.

```bash
TIMESTAMP=$(date +%Y%m%dT%H%M%S)
BRANCH="chore/sync-upstream-${TIMESTAMP}"
git checkout -b "$BRANCH" --no-track origin/main
```

### 5. Merge upstream/main

Use `-X theirs` to automatically resolve conflicts by keeping the upstream version. This is the correct default because the fork should stay aligned with upstream for all non-fork-only files.

```bash
git merge upstream/main -X theirs --no-edit -m "chore: merge upstream/main into fork

Co-Authored-By: Oz <oz-agent@warp.dev>"
```

If `-X theirs` still cannot resolve a conflict (e.g. file-level add/add conflicts that the recursive strategy cannot auto-merge), abort and report:

```bash
git merge --abort
```

Then list conflicting files and **stop** — do not push, do not create a PR. Report the conflicts to the user.

### 6. Analyze upstream changes for developer-guide/ relevance

`developer-guide/` is a fork-only directory that does NOT exist upstream. Upstream never touches it, so it will never conflict.

a. List upstream-changed files (excluding developer-guide/):

```bash
git diff --name-only origin/main upstream/main | grep -v '^developer-guide/'
```

b. Review the actual diffs of key source files — public APIs, annotations, new modules, changed config properties, build files, README changes — to understand what changed.

c. Read the existing `developer-guide/` files to understand current coverage.

d. Determine relevance. Examples of relevant changes:
   - New or modified public APIs, classes, interfaces, annotations
   - Changed build/configuration steps or property prefixes
   - New modules or removed modules
   - Changed dependencies or version requirements
   - New features or behavioral changes affecting developer usage

e. If relevant changes exist, update the affected `developer-guide/` files. Only modify sections that are impacted — do not rewrite unrelated content.

f. If no upstream changes are relevant, skip to step 8.

### 7. Commit developer-guide/ changes

```bash
git add developer-guide/
git commit -m "docs: update developer-guide/ to reflect upstream changes

Co-Authored-By: Oz <oz-agent@warp.dev>"
```

Do NOT touch any files outside `developer-guide/` (the merge commit already covers upstream files).

### 8. Push the sync branch

```bash
git push origin "$BRANCH"
```

Because the branch was created with `--no-track`, this pushes to a new remote branch `origin/$BRANCH`, not to `origin/main`.

### 9. Open a pull request

Use the `gh` CLI:

```bash
gh pr create \
  --base main \
  --head "$BRANCH" \
  --title "chore: sync upstream/main + update developer-guide/ (<date>)" \
  --body "<summary of upstream commits and developer-guide changes>"
```

Omit the "update developer-guide/" part from the title if no docs were updated.

The body should include:
- A summary of the upstream commits merged (use the log from step 3)
- Which `developer-guide/` sections were updated and why (if applicable)

### 10. Check mergeability and merge

```bash
gh pr checks "$BRANCH" --watch   # wait for status checks if any
gh pr merge "$BRANCH" --merge     # merge commit strategy, not squash/rebase
```

If the PR is mergeable with no conflicts and no blocking checks, merge it using **merge commit** strategy.

### 11. Handle blockers

If the PR cannot be merged (conflicts, failing checks), leave it open and report a concise summary of blockers to the user.

## Constraints

- Do not edit files outside `developer-guide/` (except for the merge commit itself).
- Do not force-push.
- Keep commit/PR messages concise and descriptive.
- Include the co-author line in every commit message: `Co-Authored-By: Oz <oz-agent@warp.dev>`
- Do not commit or push unless the workflow calls for it.
