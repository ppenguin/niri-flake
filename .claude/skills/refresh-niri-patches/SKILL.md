---
name: refresh-niri-patches
description: Regenerate this repo's patches/*.patch files (inhibit-idle.patch, column-anchor.patch) against a current niri-unstable revision, so they keep applying without fuzz/conflicts. Use when `just check`/`just check-docs` fails with a patch hunk that doesn't apply, when niri-unstable is bumped, or when the user asks to "update/refresh the niri patches".
---

# Refresh niri patches against upstream main

This repo carries local source patches for niri features that either aren't upstream
yet, or that this flake's Nix module (`settings.nix`) exposes ahead of niri-flake's
usual `niri-unstable` pin catching up. Patches live in `patches/` and are wired into
`niri-unstable`'s build in `flake.nix` (`make-package-set.niri-unstable.patches`).

Each patch corresponds to a feature branch on `https://github.com/ppenguin/niri`
(a personal fork that tracks `niri-wm/niri` main via fast-forward, i.e. never diverges
— it just has extra commits on top on feature branches):

| patch file | source branch | notes |
|---|---|---|
| `patches/inhibit-idle.patch` | `ppenguin/niri:inhibit_idle` | upstream PR YaLTeR/niri#2373; excludes docs/wiki |
| `patches/column-anchor.patch` | `ppenguin/niri:monitor-position-awareness-column-fill-direction` | `column-anchor` layout option + output gravity; excludes docs/wiki and the two local `*.local.md`/`feature-request-*.md` planning files in that branch |

Each patch is a plain unified diff (`--- a/`/`+++ b/`, `-p1`-style paths) with a short
prose header (what it does, `Source: <branch URL>`, and the "regenerate when it stops
applying" note) — not a `git format-patch` email. Keep that format when regenerating:
regenerate the hunks, keep/update the header.

## Procedure

Do this once per patch that needs refreshing (a hook/check failure names the file; if
asked to "refresh both", do both, independently — a failure in one must not block the
other).

**Never run `git rebase`, `git checkout`, or `git branch -f` in the user's actual
`/home/jeroen/devel/github.com/ppenguin/niri` checkout** — that repo's currently
checked-out branch is meaningful working state (it's also referenced directly as a
flake input elsewhere). Do all rebasing in a disposable `git worktree` instead, and
remove the worktree when done.

1. **Pick the target revision.** This is whatever `niri-unstable` resolves to for this
   check — normally the rev already pinned in this repo's own `flake.lock` (run
   `nix flake lock` first if you intend to also bump it: `nix flake lock --update-input
   niri-unstable`, which fetches `niri-wm/niri`'s current default branch head). Note the
   resulting rev.

2. **Rebase the feature branch onto that rev**, in a scratch worktree, without touching
   the user's checkout:
   ```bash
   cd /home/jeroen/devel/github.com/ppenguin/niri
   git fetch https://github.com/niri-wm/niri <target-rev-or-HEAD>   # ensure the commit is present locally
   git worktree add --detach /tmp/niri-patch-refresh <target-rev>
   git worktree add --detach /tmp/niri-patch-refresh-src <feature-branch>   # e.g. inhibit_idle
   ```
   Then, from a throwaway branch (create it detached, e.g. `git branch _refresh-tmp
   <feature-branch>` — do NOT reuse or move any real branch name), rebase:
   ```bash
   git branch _refresh-tmp <feature-branch>
   git rebase --onto <target-rev> $(git merge-base <feature-branch> main) _refresh-tmp
   ```
   `git rebase --onto` checks out `_refresh-tmp` as a side effect — since it's a
   throwaway branch name this is fine, but afterwards explicitly `git checkout
   <whatever-branch-was-checked-out-before>` in the main checkout to restore it, then
   `git branch -D _refresh-tmp`.

3. **Export the trimmed diff.** Compare the rebased tip against its new parent, restricted
   to the same file list the existing patch already covers (check the current
   `patches/<name>.patch` for the file list — do not silently add new files like
   `docs/wiki/*.md`, `*.local.md`, `feature-request-*.md`, or `resources/default-config.kdl`
   unless the existing patch already touched them):
   ```bash
   git diff <new-parent> _refresh-tmp -- <same file list as before> > /tmp/body.patch
   ```

4. **Verify it applies cleanly** against a clean checkout of the target rev before
   touching anything in this repo:
   ```bash
   cd /tmp/niri-patch-refresh   # the worktree from step 2, at <target-rev>
   git apply --check --verbose /tmp/body.patch
   ```
   If this reports failed/fuzzy hunks, the feature branch itself has drifted from
   upstream and needs a manual rebase-conflict resolution in the fork first — stop and
   report which hunks failed rather than forcing a fuzzy patch in.

5. **Write the final patch file**: prepend the existing header (unchanged, unless the
   feature's behavior actually changed) to the verified body, replacing
   `patches/<name>.patch` in place.

6. **Update `settings.nix`'s `# implemented by patches/<name>.patch` comment** only if it
   doesn't already reference the file (it should already, for both existing patches).

7. **Clean up**: `git worktree remove` both worktrees, delete any throwaway branches, and
   confirm `git status` in `/home/jeroen/devel/github.com/ppenguin/niri` shows the
   original branch checked out with a clean tree.

8. **Re-run the check**: `nix shell nixpkgs#just nixpkgs#fd --command just check-docs`
   (plain `just` may not be on PATH). This runs `nix flake check` (which builds
   `niri-unstable` with the patches applied — the real end-to-end test) plus a docs-eval
   sanity check. Fix and retry before committing.

9. Commit with a message describing what was refreshed and against which rev, e.g.
   `refresh column-anchor.patch against niri-unstable <shortrev>`. Do not push without
   the user's OK — the branches feeding downstream flakes (e.g. nixos-den's
   `niri` input) are fetched by ref from GitHub, so a push changes what those flakes
   resolve to.
