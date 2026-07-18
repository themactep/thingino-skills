---
name: worktree-parallel-dev
description: Run parallel Thingino tasks/agents in isolated git worktrees with correct submodule, patch, and dl-cache setup.
license: MIT
---
# worktree-parallel-dev

Use this skill when work should happen on a separate branch without disturbing
the main checkout — parallel agent sessions, an urgent fix while a long task
is in flight, or risky experiments that need cheap teardown.

Model: **one task, one branch, one worktree, one agent.**

## When to use

- The user asks for parallel work streams or multiple simultaneous agents.
- A hotfix is needed while the current tree has in-progress work (no stashing).
- A task is exploratory and may be abandoned (delete the worktree, keep the tree clean).
- A long build (30+ min) must not be interrupted by branch switching.

## Workflow

1. Create a worktree from the main checkout (never bare `git worktree add` — the
   helper also inits the buildroot submodule, applies the buildroot override
   patches from `package/all-patches/buildroot/`, and shares the `dl/` cache):
   - `scripts/worktree.sh create <branch> [base]`
   - Created at `../<repo-name>-<branch-with-dashes>/` (no spaces allowed in path).
2. Work inside the worktree only. Always pass `CAMERA=` explicitly:
   - `cd ../<repo>-<branch>` then `CAMERA=<camera> make fast`
   - Output is isolated: `output/<branch>/<camera>-.../` inside the worktree.
3. Commit checkpoints, then sync at the end of each significant session:
   - `scripts/worktree.sh sync` (rebases onto `origin/master`, re-syncs
     buildroot submodule, re-applies patches)
   - Push with `git push --force-with-lease` after a sync rebase.
4. Protect a worktree during long builds:
   - `git worktree lock <path> --reason "build running"` / `git worktree unlock <path>`
5. Cleanup after the PR merges (from the main checkout, user-confirmed):
   - `scripts/worktree.sh remove <branch>` — deletes the directory (incl. its
     `output/`), preserves the branch. Never remove a worktree unless the user
     asked for it or approved it.

## Hard rules for agents

- **Never run `make update` in a feature worktree.** It `git pull --rebase`s the
  current branch. Use `scripts/worktree.sh sync` instead.
- **Never touch files in another worktree.** Each worktree is foreign territory.
- A branch can be checked out in only one worktree at a time
  (`fatal: '<branch>' is already checked out` means someone owns it — do not force).
- Do not share `overrides/<pkg>/` source checkouts between worktrees; give each
  worktree its own clone and `local.mk` entry if an override is needed.
- Container builds (`build-container.sh`) break in linked worktrees (the `.git`
  gitfile points to an unmounted host path) — build on the host in worktrees.

## Notes

- Shared across worktrees automatically: git history, dl cache (symlinked),
  ccache (`~/.buildroot-ccache`), toolchain bundles (in dl).
- Not carried into a new worktree (gitignored): `local.mk`, `user/`,
  `overrides/`, `.selected_camera` — set up per worktree only if the task needs them.
- Full onboarding doc: `docs/worktrees.md` in the firmware repo.
