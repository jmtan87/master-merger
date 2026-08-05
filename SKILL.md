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

- `MASTERRR MERGERRRR ACTIVATEEE!`
- `MAAASTER MERGERRR, ENGAGE!!`
- `I AM THE MASTER OF ALL MERGERS!!`
- `WHO DARES WAKE UP THE MASTER OF ALL MERGERS?!`
- `MASTER-MERGERRR ONLINEEE — LET'S LAND SOME BRANCHES!`
- `MMMMASTER MERGERRRR POWERRR UP!!!`

Then proceed normally with the phases below.

## Overview

Integrate finished feature branches/worktrees into the trunk **locally**, one at a
time, verifying green after each, then cleaning up **only** the branches/worktrees
that are safe to remove. When a repo is worked by **concurrent sessions** in
sibling worktrees, the whole procedure is written to avoid stepping on live
sessions and to avoid sweeping their untracked files into commits.

**Core principle:** merge deliberately and verify continuously. A branch is a
candidate only when it is *finished* and *ahead of* trunk. **Finished means the
holding session has ended** — a clean, fully-committed worktree is NOT enough: a
live session commits each slice as it goes, so a spotless tree held by a **live
lock** is a session *between tasks*, not a done one. A live lock is a **hard
exclusion** that "merge all / merge everything" does **not** override — you skip
it and report it. Never `git add -A` during a merge. Never force-remove a worktree
a live session still holds.

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

Classify each branch on **two** axes — commits AND liveness (check both now, not at
merge time):
- **ahead 0** → nothing to merge. Skip (empty/abandoned, or already merged).
- **live-locked worktree** (lock pid still running) → **NOT a candidate, whatever the
  ahead count.** A running session owns it; leave it and report it. Detect it here:
  ```bash
  git worktree list --porcelain    # lock lines carry the pid: "locked ... pid <N>"
  ps -p <N> 2>/dev/null && echo LIVE || echo dead   # Windows: Get-Process -Id <N>
  ```
- **ahead > 0 and not live-locked** → candidate. Confirm it's *finished* at the Phase 2 gate.

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

**Per-branch finished-gate — ALL must pass before merging a concurrent-session branch.**

**Gate 1 — the lock is not live (check FIRST; it overrides every other signal).**
```bash
git worktree list --porcelain   # find "locked ... pid <N>" for this worktree
ps -p <N> 2>/dev/null && echo LIVE || echo dead   # Windows: Get-Process -Id <N>
```
A **live** pid = the session is mid-flight → **STOP: do not merge, report it, move on.**
This holds *even when the worktree is spotless and every commit is in* — a live session
commits each task as it finishes and is working on the next, so a clean tree is **not**
consent to merge. "Merge all / merge everything" does **not** waive this gate. Only a
**stale** lock (pid dead) clears it — that means abandoned/finished work, a real candidate.

**Gate 2 — no uncommitted source in the worktree.**
```bash
wt="<path from git worktree list>"
git -C "$wt" status --porcelain | grep -vE '\.next|node_modules|dist|target|test-results'   # must be empty
```

**Gate 3 — the branch ref isn't moving under you.** If `git rev-parse <branch>` is *ahead
of* the worktree's HEAD, or changes between your read and your merge, a session is actively
committing to that branch → treat it as live and back off. (Seen for real: a branch head
advanced one commit past the worktree between detection and merge — the tell of a live
session.)

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

**First, sync dependencies if any merge touched the manifest/lockfile.** A merge that
lands new deps leaves the trunk checkout's installed packages **stale**, and letting a
stale tree get relinked mid-verify is how `node_modules` gets corrupted (half-removed
`.bin/`, empty packages) — especially on **pnpm + Windows**, where packages are
hard-links/junctions into a global store and any file lock aborts the relink partway.
Do it deliberately, with locks cleared:

```bash
git diff --name-only HEAD@{1} HEAD | grep -E 'package\.json|pnpm-lock\.yaml|yarn\.lock|package-lock\.json' \
  && echo "deps changed — sync before verifying"
# 1. STOP any dev server first (it locks node_modules/.bin/* → aborts the relink).
# 2. Install from the lockfile exactly (no resolution drift), with the repo's PM:
pnpm install --frozen-lockfile
```

If that install **errors partway**, treat `node_modules` as poisoned — a plain re-run
won't repair it (lockfile-trust makes pnpm skip the half-written tree). Wipe and reinstall:
`rm -rf node_modules && pnpm install --frozen-lockfile`. Only then run the gate.

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
#    Windows caveat: prefer `git worktree remove` to clear the dir — Git-Bash `rm -rf`
#    can FOLLOW a pnpm junction (node_modules/<pkg> → the store) and delete shared
#    content instead of just unlinking. If you must rm, ensure the worktree has no
#    live node_modules links first.
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
a newly-merged API).

**The merge record belongs in git + `docs/superpowers/STATUS.md`, NOT in project
memory.** Flip the feature's STATUS.md row at the merge boundary (that is the single
reconciliation point for "what's built"); `git log` + the merge sha already hold the
rest. Do **not** mint a per-branch "X MERGED (date + sha)" memory file — that bloats
the always-loaded MEMORY.md index with facts git already records, and the harness
memory rules say not to save what the repo already knows. Write project memory ONLY
when the merge surfaced something git can't give you — a genuine **parked/deferred
follow-up** or a **locked cross-branch decision** — and even then as a single concise
line, folded into the relevant domain memory, never a new file per branch.

## Red flags — STOP

- About to `git add -A` / `git add .` mid-merge → stage resolved files by path.
- About to `git worktree remove --force` → check the pid is dead AND step-1 rescue
  is clean first.
- Merging a branch whose worktree is **live-locked** (lock pid alive) → **hard stop,
  never merge it** — even under "merge all", even if the tree is spotless and fully
  committed. A live session commits slice-by-slice; merging its branch mid-plan lands a
  *partial* feature and makes the status doc you write falsely claim it's done. (Real
  incident: a sweep merged an in-progress `corporate-actions` branch — 1 of 5 planned
  tasks — into main, destroyed the live session's worktree, and wrote a "merged ✅" row
  for a feature that barely existed. The tree was clean; the lock was live. The clean tree
  is what fooled the sweep — check the pid, not the tree.)
- Merging a branch whose worktree has uncommitted source → not finished; leave it.
- Cleaning up before the verify gate is green → verify first.
- Verifying (or restarting the app) after a lockfile-changing merge **without syncing
  deps** → stop the dev server, `pnpm install --frozen-lockfile`, THEN verify. A stale
  tree relinked under a file lock corrupts `node_modules`; a partial install won't
  self-repair — wipe and reinstall.
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

# per branch: finished-gate (pid-not-live FIRST → clean tree → ref not moving) → merge → resolve BY PATH (never -A) → commit --no-edit
# live lock = hard skip, even under "merge all", even if the tree is spotless
git merge --no-ff <branch> -m "Merge branch '<short>' (<what>)"

# verify (trunk-only) with the repo's runner, excluding sibling worktrees, then reconcile status doc

# cleanup (dead sessions only): rescue check → remove (no --force first) → prune → git branch -d
```
