---
name: master-merger
description: >-
  Use when asked to integrate outstanding feature branches or worktrees into the
  trunk (main/master) of a git repo — triggers include "merge all into main",
  "find things to merge", "merge the finished branches", or cleaning up worktrees
  after merges. Especially for repos worked by concurrent agents/sessions in
  sibling worktrees, where finished branches must land without disrupting sessions
  still running.
---

# master-merger

## On activation — shout it

The instant this skill kicks in, before any git work, emit a single loud,
over-the-top battle-cry announcing that master-merger is live. Make it **random
each time** — riff on the name, vary the drawn-out letters and the verb. It's a
one-line hype shout, then get straight to work. Examples (don't just copy — invent
a fresh one):

- `⚡ MASTERRR MERGERRRR ACTIVATEEE! ⚡`
- `🔀 MAAASTER MERGERRR, ENGAGE!! 🔀`
- `👑 I AM THE MASTER OF ALL MERGERS!! 👑`
- `😤 WHO DARES WAKE UP THE MASTER OF ALL MERGERS?!`
- `🚀 MASTER-MERGERRR ONLINEEE — LET'S LAND SOME BRANCHES! 🚀`
- `🔥 MMMMASTER MERGERRRR POWERRR UP!!! 🔥`

Then proceed normally with the phases below.

## Overview

Integrate finished feature branches/worktrees into the trunk **locally**, one at a
time, verifying green after each, then cleaning up **only** the branches/worktrees
that are safe to remove. When a repo is worked by **concurrent sessions** in
sibling worktrees, the whole procedure is written to avoid stepping on live
sessions and to avoid sweeping their untracked files into commits.

**Core principle:** merge deliberately and verify continuously. A branch is a
candidate only when it is *finished* (clean worktree, not held by a live session)
and *ahead of* trunk. Never `git add -A` during a merge. Never force-remove a
worktree a live session still holds.

This skill is repo-agnostic; where it names a concrete tool/file (`pnpm`,
`STATUS.md`), treat it as an example and substitute the current repo's equivalent.
Detect the specifics first: trunk name (`git symbolic-ref refs/remotes/origin/HEAD`
or just `main`/`master`), test runner (`package.json` scripts / Makefile /
`cargo`/`go`/`pytest`), and whether a status/index doc is maintained.

## When to use

- "merge all into main" / "merge everything" / "find things to merge"
- Finishing concurrent work: several `.worktrees/*` (or similar) branches are done
  and need to land.
- Post-merge cleanup of worktrees and branches.

**When NOT to use:** a single branch you authored this session — use
`superpowers:finishing-a-development-branch` directly. Anything that must be
pushed/PR'd to a remote is a separate decision: this skill merges **locally** and
does **not** push unless the user explicitly asks.

## Phase 0 — Detect what's mergeable

```bash
git fetch --all --prune
TRUNK=main   # or master; verify with: git symbolic-ref --short refs/remotes/origin/HEAD
for b in $(git for-each-ref --format='%(refname:short)' refs/heads/ | grep -v "^$TRUNK$"); do
  echo "$b: ahead $(git rev-list --count $TRUNK..$b) / behind $(git rev-list --count $b..$TRUNK)"
done
git worktree list        # note which worktrees are 'locked'
```

Classify each branch:
- **ahead 0** → nothing to merge. Skip (empty/abandoned, or already merged).
- **ahead > 0** → candidate. Confirm it's *finished* before merging (Phase 2 gate).

## Phase 1 — Pre-flight (clean tree + baseline green)

1. `git status --short`. The tree must be clean apart from known-untracked noise.
   **Stray uncommitted work in the trunk checkout is a red flag** — usually another
   session's experiment. Stash it by explicit path, don't discard it:
   ```bash
   git stash push -u -m "stray <what>" -- "<path>"
   ```
2. Baseline the suite so you know green-to-green. **Exclude sibling worktree copies**
   (see the Verify gate for why):
   ```bash
   # example (vitest/pnpm); substitute the repo's runner:
   pnpm test --run --exclude '**/.worktrees/**' --exclude '**/.claude/**' | tail -6
   ```

## Phase 2 — Merge sequentially

**Order:** cleanest first. A branch that is `behind 0` sits right on trunk and
merges without conflict — do those first. Otherwise order only affects how many
conflicts you touch; foundational branches before their consumers reduces churn.

**Per-branch finished-gate (must pass before merging a concurrent-session branch):**
```bash
wt="<path from git worktree list>"
git -C "$wt" status --porcelain | grep -vE '\.next|node_modules|dist|target|test-results'   # must be empty
git worktree list --porcelain | grep -A3 "<name>" | grep -i locked                          # note if locked
# if locked, is the locking session ALIVE?
git worktree list --porcelain   # lock reasons often carry a pid; then:
ps -p <pid>                      # ALIVE → not yours to merge; leave it
```
If the worktree has uncommitted work or is **locked by a live session**, it isn't
finished — leave it and report it. A **stale** lock (pid dead) means abandoned/
finished work — safe to treat as a candidate.

**Merge:**
```bash
git merge --no-ff <branch> -m "Merge branch '<short>' (<one-line what>)"
git diff --name-only --diff-filter=U     # conflicts, if any
```

### The one rule that bites: staging after a conflict

**NEVER `git add -A` or `git add .` while resolving a merge.** The trunk checkout
carries untracked files — worktree gitlinks, local scratch, personal docs — and
`-A` sweeps them into the merge commit (real incident: `.claude/worktrees/*`
embedded-repo gitlinks and a personal PDF both got committed this way). **Stage
only the files you resolved, by explicit path:**
```bash
git add "path/to/conflicted-a" "path/to/conflicted-b"
git commit --no-edit
```
If junk already landed, untrack it (keep the file on disk) and commit the removal:
`git rm --cached <path>` / `git rm -r --cached <dir>`.

## Conflict playbook (generic categories)

| Kind | Resolution |
|---|---|
| Status/index/changelog doc both sides appended to (e.g. a `STATUS.md`) | Usually **additive** — union both sides' entries now; do the real reconciliation once in Phase 4. |
| A shared list/enum/seed file both sides extended (e.g. a config or fixtures array) | **Union** both sides' entries; keep sensible ordering. |
| A count/length assertion that depends on such a list | After unioning, count the real list and set the number — don't guess. |
| Two independent blocks added at the same spot in one file (e.g. two `describe`/`test` blocks) | Keep **both**; watch shared trailing braces/brackets — give each block its own closing. |
| modify/delete where one branch **retires** the file | The **delete wins** if that branch's design removes it; confirm no dangling refs (`grep -rn "<symbol/route>" <src dirs>`). |

After resolving: `grep -rn '<<<<<<<\|>>>>>>>\|^=======' <files>` returns nothing,
run the affected tests, then stage-by-path + commit.

## Phase 3 — Verify gate (after all merges)

Test/type runs from the trunk checkout often **crawl sibling worktrees** (inflated
counts, stale copies). Exclude them so the signal is trunk-only. Both the test
suite and the type/lint gate must be **green before any cleanup**:

```bash
# examples — substitute the repo's commands:
pnpm test --run --exclude '**/.worktrees/**' --exclude '**/.claude/**' | tail -6
pnpm exec tsc --noEmit 2>&1 | grep 'error TS' | grep -v worktrees   # must print nothing
```
If a merge went red, stop and fix — nothing is pushed, so it's fully recoverable
(`git reset --hard`, `git merge --abort` mid-merge).

## Phase 4 — Reconcile the status doc (if the repo has one)

If the repo maintains a status/index doc that tracks what's landed (some repos flip
rows "at the merge boundary"): mark each merged feature done, point it at the merge,
and remove it from any "in-flight/unmerged" list. Leave genuinely-unmerged branches
(empties, live sessions) listed. Commit separately, e.g.
`docs(status): mark <features> merged`. Skip this phase if no such doc exists.

## Phase 5 — Cleanup (safety first)

Only after green. Remove a worktree **before** deleting its branch (a checked-out
branch can't be deleted).

```bash
git worktree list --porcelain      # find lock reasons like "... session ... pid <N>"
ps -p <pid>                        # STILL RUNNING → do NOT touch it
```

**NEVER `git worktree remove --force` a worktree locked by a live pid** — it
destroys that session's uncommitted work. Merged work is safe on trunk; the orphan
can wait.

For sessions confirmed dead / worktrees with no live holder:
```bash
# 1. Rescue first: real untracked/modified work (NOT build artifacts)?
git -C "<wt>" status --porcelain | grep -vE '\.next|node_modules|dist|target|test-results'
#    Anything printed = uncommitted source. Copy it out / commit it BEFORE removing.
git worktree unlock  "<wt>"        # locked worktrees only
# 2. Try WITHOUT --force first — it refuses if real work would be lost:
git worktree remove "<wt>" || git worktree remove --force "<wt>"   # escalate ONLY
#    once step 1 confirms the block is build output / a stale lock, not source
[ -d "<wt>" ] && rm -rf "<wt>"     # git often leaves the dir behind
git worktree prune
git branch -d <branch>             # -d = merged-only safety; refuses unmerged
```

> `--force` bypasses git's untracked/modified guard — it will silently delete an
> uncommitted file living only in that worktree. Never reach for it until step 1
> confirms the only thing in the way is build output or a stale lock.

An **empty** orphan dir that stays "busy"/"Directory not empty" is usually a
still-running process handle (a dev-server file-watcher). It's de-registered and
harmless — **don't kill processes to delete a folder**; note it and move on.

`git branch -d` refusing ("not fully merged") means the branch **isn't** merged —
investigate, don't reach for `-D`.

## Phase 6 — Report (and memory, if you keep one)

Report: what merged (with conflicts resolved), the green gate numbers, what you
deliberately left (live-locked worktrees, empties), and that nothing was pushed.
Note any cross-branch follow-up you spotted (e.g. a consumer that should now adopt
a newly-merged API). If you maintain project memory, record each feature as merged
(date + merge sha) and flip any "unmerged" notes.

## Red flags — STOP

- About to `git add -A` / `git add .` mid-merge → stage resolved files by path.
- About to `git worktree remove --force` → check the pid is dead AND step-1 rescue
  is clean first.
- Merging a branch whose worktree has uncommitted changes or is live-locked → it's
  not finished; leave it.
- Cleaning up before the verify gate is green → verify first.
- `git branch -D` (force) to delete a branch that "won't delete" → `-d` refused
  because it's **not merged**; investigate, don't force.
- Test run reporting a huge/weird file count → you forgot the worktree `--exclude`
  flags; the number includes sibling copies.
- About to `git push` → only if the user explicitly asked; this skill merges local.

## Quick reference

```bash
# detect
git fetch --all --prune; TRUNK=main
for b in $(git for-each-ref --format='%(refname:short)' refs/heads/ | grep -v "^$TRUNK$"); do
  echo "$b: ahead $(git rev-list --count $TRUNK..$b) / behind $(git rev-list --count $b..$TRUNK)"; done

# per branch: finished-gate (clean + lock/pid) → merge → resolve BY PATH (never -A) → commit --no-edit
git merge --no-ff <branch> -m "Merge branch '<short>' (<what>)"

# verify (trunk-only) with the repo's runner, excluding sibling worktrees, then reconcile status doc

# cleanup (dead sessions only): rescue check → remove (no --force first) → prune → git branch -d
```
