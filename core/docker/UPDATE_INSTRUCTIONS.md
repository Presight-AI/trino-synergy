# Syncing the Fork with Upstream Trino

How to pull the latest changes from upstream `trinodb/trino` into your fork and rebase them into the `synergy-develop` branch.

## Repository Layout

This fork has two remotes:

| Remote     | Points to                              | Purpose                               |
|------------|----------------------------------------|---------------------------------------|
| `origin`   | `github.com/Presight-Ai/trino-synergy` | Trino synergy repository (forked one) |
| `upstream` | `github.com/trinodb/trino`             | Original Trino repo (read-only)       |

Branches:

| Branch            | Where it lives        | Purpose                                         |
|-------------------|-----------------------|-------------------------------------------------|
| `master`          | both remotes          | Mirror of upstream — never commit here directly |
| `synergy-develop` | `origin` only         | Working branch with custom changes              |

## One-Time Setup

If `upstream` isn't configured yet, add it:

```bash
git remote add upstream https://github.com/trinodb/trino.git
git remote -v
```

You should now see both `origin` and `upstream` listed.

## Sync Procedure

The flow is:

1. Fetch upstream changes
2. Fast-forward your local `master` to match `upstream/master`
3. Push the updated `master` to your fork
4. Rebase `synergy-develop` onto the new `master`
5. Force-push `synergy-develop` to your fork
### Step 1: Make sure your working tree is clean

Rebasing will fail if you have uncommitted changes. Either commit them or stash:

```bash
git status
git stash push -m "wip before upstream sync"   # if needed
```

### Step 2: Fetch the latest from upstream

```bash
git fetch upstream
```

This downloads upstream's commits but doesn't change any of your branches yet.

### Step 3: Update your local `master`

```bash
git checkout master
git merge --ff-only upstream/master
```

`--ff-only` ensures the merge is a fast-forward — it will fail if `master` has diverged, which protects you from accidentally committing to it. If this fails, someone has committed to `master` directly; see [Troubleshooting](#troubleshooting).

### Step 4: Push the updated `master` to your fork

```bash
git push origin master
```

### Step 5: Rebase `synergy-develop` onto the new `master`

```bash
git checkout synergy-develop
git pull --ff-only origin synergy-develop   # make sure local is up to date with remote
git rebase master
```

Git will replay your synergy commits one at a time on top of the latest upstream code.

**If conflicts occur**, Git pauses on the conflicting commit. Resolve it like this:

```bash
# Edit the conflicted files
git add <resolved-files>
git rebase --continue
```

Repeat until the rebase finishes. To abort and return to the pre-rebase state at any point:

```bash
git rebase --abort
```

### Step 6: Push `synergy-develop` back to your fork

Because rebase rewrites history, a normal `git push` will be rejected. Use `--force-with-lease` — it's safer than `--force` because it refuses to overwrite work someone else may have pushed in the meantime:

```bash
git push --force-with-lease origin synergy-develop
```

### Step 7: Restore stashed work (if any)

```bash
git stash pop
```

## Quick Reference

The full sync, assuming a clean working tree and no conflicts:

```bash
git fetch upstream
git checkout master
git merge --ff-only upstream/master
git push origin master
 
git checkout synergy-develop
git pull --ff-only origin synergy-develop
git rebase master
git push --force-with-lease origin synergy-develop
```
