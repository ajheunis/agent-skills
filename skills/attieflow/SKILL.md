---
name: attieflow
description: Attie's preferred GitHub-issue-driven coding workflow. Use whenever the user wants to start work on a new task, fix, or feature in a Git repo — even if they don't say "attie-flow" by name. Trigger on phrases like "let's work on issue #123", "start a new branch for this", "begin work on this bug", "I want to fix X", "let's tackle this issue", "push and PR", "merge this PR", "clean up after the merge", or any time the user is about to start, finish, or merge a piece of coding work in a repo with a GitHub remote. Always orchestrate the full flow: GitHub issue → linked branch (via `gh issue develop`) → local work → push → PR → user-gated squash merge with branch delete → local prune and refresh. Do not skip the approval gate before merging.
---

# attieflow

Every change traces from issue → linked branch → squash-merged commit on `main`. Five phases; figure out which one Attie is entering at and pick up there.

## Preconditions

Confirm `gh auth status` works and `git remote -v` points at GitHub. Resolve the default branch with `gh repo view --json defaultBranchRef -q .defaultBranchRef.name` and use that name wherever this skill says `main`. If `gh` is missing or unauthenticated, stop — `gh issue develop` is the linkage and plain `git checkout -b` is not a substitute.

## Phase 1 — Issue → linked branch

Require a clean tree (`git status --porcelain`); if it isn't, ask Attie what to do. Refresh with `git pull --ff-only origin main` — fail loudly on divergence rather than silently merging.

Ask for the issue number. If there isn't one, help draft a short title and body and run `gh issue create`. Then:

```bash
gh issue develop <issue-number> --checkout
```

This is the load-bearing command. It registers the branch as a development branch on the issue, which is what surfaces the link under "Development" on the issue page on GitHub.com. Plain `git checkout -b` will not do that. Confirm with `git branch --show-current` and hand off.

## Phase 2 — Work

Attie's coding phase. Help with implementation if asked, but do not push the workflow forward until he says he's ready to ship.

## Phase 3 — Push and PR

Read back what's about to ship: `git log main..HEAD --oneline`. Then:

```bash
git push -u origin HEAD
gh pr create --fill --assignee @me
```

The PR auto-links to the issue (and the issue auto-closes on merge) because the branch came from `gh issue develop` — no `Closes #N` keyword needed. Show the PR URL and stop.

## Phase 4 — Merge gate

**Do not merge without an explicit "merge it" / "go ahead" from Attie in this turn.** Reviewer approval on GitHub is necessary but not sufficient — the final go-ahead must come from him here, because he treats merging as a change-control event. When he gives it:

```bash
gh pr merge <pr-number> --squash --delete-branch
```

Both flags are required. `--squash` is the chosen merge strategy; `--delete-branch` removes the remote branch as part of the merge.

## Phase 5 — Cleanup

```bash
git checkout main
git fetch --prune
git for-each-ref --format='%(if)%(HEAD)%(then)%(else)%(refname:short) %(upstream:track)%(end)' refs/heads/ \
  | grep '\[gone\]$' \
  | cut -d' ' -f1 \
  | xargs -r git branch -D
git pull --ff-only origin main
git status && git log -1 --oneline
```

Use exactly that pruning command. Do not "simplify" it to `git branch -vv | awk ...` — the reasons are structural, not stylistic, and `-D` force-deletes regardless of merge state:

- `-D` (not `-d`) is correct. After a squash merge the local branch tip isn't an ancestor of `main` (the squashed commit is a different SHA), so `-d` refuses.
- `for-each-ref` never prints the commit subject, so a subject containing `gone]` cannot be matched. `git branch -vv` does print it, which is why a `branch -vv` filter has to escape its way around subjects like `fix: gone] handling` — that branch has an intact upstream, and a naive filter force-deletes it.
- `%(if)%(HEAD)%(then)%(else)...%(end)` emits nothing at all for the current branch, so `-D` can never target `HEAD`. That is a structural guard, not a regex one. (`%(HEAD)` expands to a single space for non-current branches, so a naive `'%(HEAD) %(refname:short) ...'` format gives two leading spaces and `cut -d' ' -f2` lands on an empty field — hence the `%(if)` form.)
- `%(upstream:track)` prints `[gone]` only when the remote counterpart was deleted. `main`, live-upstream branches, and local-only never-pushed branches all fall through untouched.
- No positional argument (`$N`, for any digit N) appears anywhere in the pipeline. That is deliberate — see "Never" below.

Show the `[gone]` list first if there's any doubt — same command, minus the `xargs`:

```bash
git for-each-ref --format='%(if)%(HEAD)%(then)%(else)%(refname:short) %(upstream:track)%(end)' refs/heads/ \
  | grep '\[gone\]$' | cut -d' ' -f1
```

## Edge cases

- **Squash disabled on the repo**: `gh pr merge --squash` errors. Surface it; ask Attie before switching to `--rebase` or `--merge`.
- **Push rejected** (protected branch, fork required): surface the error; don't auto-fork.
- **Working in a fork**: same flow; prune both remotes with `git fetch --all --prune`.
- **Branch already exists from a previous attempt**: `gh issue develop` reuses it. Check `git log main..HEAD` for prior work before continuing.

## Never

Auto-merge. Touch `main` with anything other than `--ff-only`. Delete branches with a live upstream.

Never write a bare positional argument — a dollar sign followed by a digit — into a shell or `awk` snippet in this file, and never into prose either. When `/attieflow` is invoked with arguments, each such token in the skill body is replaced by the corresponding whitespace-separated token of those arguments *before the body reaches the agent*. An `awk` field reference like `{print <dollar>1}` therefore arrives as `{print the}` — an uninitialised `awk` variable that prints a blank line, after which `xargs -r` declines to run and the cleanup silently no-ops while reporting success. That is why this file contains no such token anywhere, including in this paragraph. If a field reference is ever unavoidable, pass it through a named `awk -v` variable, or restructure to a `git for-each-ref --format` string, which needs none.
