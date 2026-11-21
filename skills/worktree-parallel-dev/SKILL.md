---
name: worktree-parallel-dev
description: Run parallel Thingino tasks/agents in isolated git worktrees with correct submodule, patch, dl-cache, and user-file setup.
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
   - Add `IP=<camera_ip>` when the user tree has a device-scoped layer
     (`user/<camera>/<ip>/`); without it those overrides are silently skipped.
   - Output is isolated: `output/<branch>/<camera>-.../` inside the worktree.
3. Commit checkpoints, then sync at the end of each significant session:
   - `scripts/worktree.sh sync` (rebases onto `origin/master`, re-syncs
     buildroot submodule, re-applies patches)
   - Push with `git push --force-with-lease` after a sync rebase.
4. Protect a worktree during long builds:
   - `git worktree lock <path> --reason "build running"` / `git worktree unlock <path>`
5. Cleanup after the PR merges (from the main checkout, user-confirmed):
   - Merge with `gh pr merge --merge --delete-branch` (or the repo's merge
     method) so the remote head branch is deleted with the merge.
   - `scripts/worktree.sh remove <branch>` — deletes the directory (incl. its
     `output/`), preserves the branch. Never remove a worktree unless the user
     asked for it or approved it.
   - Delete the merged branch: `git branch -D <branch>`; if the merge did not
     already remove it, `git push origin --delete <branch>`. Do not leave a
     merged branch or its worktree behind.

## User files are not carried into a worktree

`user/` (`THINGINO_USER_DIR`) is gitignored and defaults to `<checkout>/user`.
A fresh worktree has no `user/`, and the build does not fail or warn: it
silently produces firmware with no user customization. That means no camera
account passwords, no WiFi credentials, no streamer settings, no third-party
service accounts. A camera flashed with such an image stays in the onboarding
portal and direct-connect AP — it never joins the regular network.

### Give every worktree the same user tree

Preferred: export one canonical tree in the shell that starts the build. All
worktrees inherit it and nothing is copied:

```bash
export THINGINO_USER_DIR=$HOME/.thingino/user
```

An exported value replaces the default outright — the per-worktree `user/` is
then not consulted at all.

Alternative: symlink the main checkout's tree into the worktree. Do not copy —
copies drift and duplicate secrets:

```bash
ln -s ../<main-checkout>/user ../<worktree>/user
```

### Verify what the build consumed

Build targets print a user-input banner near the top of the log. Read it before
flashing:

```text
=== User Build Inputs ===
THINGINO_USER_DIR: /home/paul/.thingino/user
repo local.mk: none
local.fragment:
  - /home/paul/.thingino/user/common/local.fragment
user local.mk: none
thingino.json:
  - /home/paul/.thingino/user/common/thingino.json
prudynt.json:
  - /home/paul/.thingino/user/common/prudynt.json
local.uenv.txt: none
overlay files:
  - ...
opt files: none
=========================
```

Some lines legitimately say `none`. The red flag is `THINGINO_USER_DIR`
pointing inside the worktree (or at `/dev/null`) with no `overlay`, JSON, or
fragment files listed: the build carried no user files. Treat that as a
blocker for any image that must reach the regular network. `PRISTINE=1`
deliberately sets `THINGINO_USER_DIR=/dev/null`; use it for upstream/CI
validation only, never for a device that must come online.

### Device-scoped layers need `IP=`

`user/<camera>/<ip>/` overrides apply only when the build gets `IP=`:
`CAMERA=<camera> IP=<camera_ip> make fast`. If the IP is unknown, ask the user
instead of guessing; a wrong IP silently selects the wrong layer, or none.

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
- **Never flash or OTA a worktree build whose user-input banner lists no
  `user/` files.** That image has no WiFi or account credentials and boots into
  the portal/direct-connect AP. Fix the user tree and rebuild first.
- Never use `PRISTINE=1` for an operational image; it disables `user/` on
  purpose.

## Notes

- Shared across worktrees automatically: git history, dl cache (symlinked),
  ccache (`~/.buildroot-ccache`), toolchain bundles (in dl).
- Not carried into a new worktree (gitignored): `local.mk`, `user/`,
  `overrides/`, `.selected_camera`. `user/` is mandatory for device builds —
  see "User files are not carried into a worktree" above. `overrides/` needs
  its own clone per worktree. Pass `CAMERA=` (and `IP=` when device-scoped
  user files exist) explicitly.
- Full onboarding doc: `docs/dev/worktrees.md` in the firmware repo.
