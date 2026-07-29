# master-merger

A [Claude Code](https://claude.com/claude-code) **Agent Skill** that integrates
outstanding feature branches / git worktrees into a repo's trunk (`main`/`master`)
**safely**, then cleans up — built for repos worked by **concurrent agents or
sessions** in sibling worktrees.

It encodes the discipline that prevents the classic merge-automation mistakes:

- **Never `git add -A` mid-merge** — stage only resolved files by path, so untracked
  worktree gitlinks / local scratch don't get swept into a merge commit.
- **Check the locking session is dead (`ps -p <pid>`) before removing a worktree** —
  never force-remove a workspace a live session still holds.
- **Verify green with sibling worktrees excluded** — trunk-checkout test/type runs
  otherwise crawl stale worktree copies.
- **`git worktree remove` without `--force` first** — `--force` silently deletes
  untracked source living only in that worktree.
- Merges **locally**; never pushes unless you ask.

The skill is **repo-agnostic**: trunk name, test runner, and any status/index doc
are detected per-repo.

## Install

```bash
git clone https://github.com/jmtan87/master-merger.git ~/.claude/skills/master-merger
```

That makes it a **personal skill**, available in every project. (For a single repo
instead, clone into `<repo>/.claude/skills/master-merger`.)

Start a fresh Claude Code session so the skill is discovered.

## Use

Invoke it with `/master-merger`, or just ask — e.g. *"merge all into main"*,
*"find things to merge"*, *"clean up the merged worktrees"*.

## Phases

detect mergeable branches → pre-flight (clean tree + baseline green) → sequential
conflict-resolving merges → verify gate → reconcile status doc (if any) → safe
cleanup → report.

See [`SKILL.md`](./SKILL.md) for the full runbook.

## License

MIT — see [LICENSE](./LICENSE).
