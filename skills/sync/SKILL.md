---
name: sync
description: Bring the current git repo and its remote to the same, newest state — fetch, then pull / rebase / commit / push as the divergence requires. Use when the user says /sync, "sync", "원격이랑 맞춰줘", "동기화", "최신으로 맞춰줘", "pull 받고 push 해줘", or otherwise asks to reconcile local and remote.
---

# sync — reconcile local and remote

One command that inspects the divergence between the current branch and its upstream, then does the smallest thing that leaves both sides on the same newest commit. Works in any repo. Never rewrites published history, never discards work without asking.

## 1. Read the state

Run these first and base every decision on the output:

```bash
git rev-parse --abbrev-ref HEAD          # current branch
git rev-parse --abbrev-ref --symbolic-full-name '@{u}'   # upstream (may fail)
git fetch --all --prune
git status --porcelain=v1 -b
git rev-list --left-right --count '@{u}...HEAD'          # behind<TAB>ahead
```

Use `git rev-list`, not `git log`, for the count — `git log --count` prints nothing through some shells.

Read the first line of `git status -sb` before anything else — it reports branch, upstream, unborn state, and a deleted upstream in one shot, and it succeeds in cases where `rev-parse` errors out. Stop and report, rather than proceeding, on any of these:

- `fatal: not a git repository` — the directory is not a repo, or its `.git` is incomplete (a `.git` holding only `objects`, say). Nothing to sync; do not run `git init` to "fix" it.
- `## No commits yet on <branch>` — unborn branch. `rev-parse HEAD` fails and no count exists. If the remote has commits, `git pull` can still populate it; otherwise there is nothing to sync until the first commit.
- `## <branch>...<upstream> [gone]` — the upstream branch was deleted on the remote (usually merged and cleaned up). Counting against it fails. Tell the user, and offer either re-pointing at the default branch (`git switch main && git pull`) or re-publishing this one (`git push -u origin <branch>`). Never delete the local branch for them.
- No upstream at all (`rev-parse '@{u}'` fails). Push it with `git push -u origin <branch>` if the user wants it published; otherwise say so and stop.
- No remote configured (`git remote` prints nothing) — there is nothing to sync against.

Detached HEAD (`## HEAD (no branch)`) has no upstream by definition — report it and stop rather than guessing a branch name.

## 2. Handle uncommitted work

If `git status --porcelain` is non-empty:

- Show the user the changed paths and a short `git diff --stat`.
- Ask whether to commit them or stash them for the duration. Do not decide silently — uncommitted work is the user's, and the answer changes what the rest of the run does.
- If committing: stage the files the user named (`git add`), write a commit message in the repo's existing style (check `git log --oneline -20` for the convention — this repo family uses Conventional Commits with Korean bodies), and commit.
- If stashing: `git stash push -u -m "sync: <timestamp>"`, finish the sync, then `git stash pop` and report any conflicts.
- Never `git checkout --`, `git reset --hard`, or `git clean` to get a clean tree.

## 3. Reconcile by divergence

Four cases, from the ahead/behind counts:

| behind | ahead | Action |
|---|---|---|
| 0 | 0 | Already in sync. Report and stop. |
| >0 | 0 | `git pull --ff-only`. Fast-forward, nothing else needed. |
| 0 | >0 | `git push`. |
| >0 | >0 | Diverged — see below. |

**Diverged.** Show both sides first (`git log --oneline '@{u}..HEAD'` and `git log --oneline 'HEAD..@{u}'`), then pick:

- Local commits are unpushed and private → `git pull --rebase`, then `git push`. Preferred default: linear history, no merge noise.
- Local commits are already pushed elsewhere, or the branch is shared → `git pull --no-rebase` (merge), then `git push`. Rebasing published commits rewrites history other people have.
- If unsure which, ask. Do not guess on a shared branch.

Never use `git push --force` or `--force-with-lease` unless the user explicitly asks for it in this conversation.

## 4. Conflicts

If the pull/rebase stops on conflicts:

- List the conflicted files (`git diff --name-only --diff-filter=U`).
- Resolve only conflicts whose intent is unambiguous (lockfiles, generated types, an import list where both sides clearly want their line). For anything touching real logic, show both sides and ask.
- Rebase: `git rebase --continue` after staging. Merge: `git commit`. Offer `git rebase --abort` / `git merge --abort` if the user would rather back out.
- After resolving, run the repo's typecheck/test command if one is obvious from `package.json` before pushing.

## 5. Report

End with a short summary the user can verify at a glance:

- branch and upstream
- what happened (fast-forwarded / rebased N commits / merged / pushed N commits / already in sync)
- final `git log --oneline -1` for both local and `@{u}`, showing they match
- anything left for the user: stash popped with conflicts, a branch with no upstream, tests not run

## Notes

- `git fetch --prune` deletes remote-tracking refs for branches deleted on the remote. That is local bookkeeping only — it never touches the user's local branches.
- If the current branch is the default branch (`main`/`master`) and the user has local commits on it, mention that the repo convention may be to work on a feature branch — but still complete the sync they asked for.
- With multiple remotes, sync against the branch's own upstream, not `origin` by assumption.
