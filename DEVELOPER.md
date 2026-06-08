# lab-code_puppy — Developer Guide (Vizio fork)

This file is **Vizio-only**. It is not present in upstream
[`mpfaffenberger/code_puppy`](https://github.com/mpfaffenberger/code_puppy)
and never will be — keeping our content out of `AGENTS.md` is what makes
upstream merges conflict-free.

If you came here from `AGENTS.md`'s Vizio-fork pointer: this is the
right place. If you're looking for the upstream plugin contract, hooks
table, or core rules, that's still in `AGENTS.md`.

## What is this branch?

`forked-main` on `BuddyTV/lab-code_puppy` is the Vizio downstream of
`mpfaffenberger/code_puppy`. Origin lives at **`BuddyTV/lab-code_puppy`**
(internal repo) — the canonical home, replacing the previous personal
fork at `breedx/forked-puppycode`. The upstream maintainer's repo is
unchanged.

The wheel built from `forked-main` is vendored into
[`lab-pack`](https://github.com/BuddyTV/lab-pack) via `make vendor-puppy`
and shipped to users as part of the lab stack. Don't edit `forked-main`
directly unless the change should land in production.

## Remotes

```
origin    git@github.com:BuddyTV/lab-code_puppy.git   # Vizio fork (internal)
upstream  https://github.com/mpfaffenberger/code_puppy.git
```

`upstream` is required for `make vendor-puppy` (which runs `git fetch
upstream && git merge upstream/main` from `lab-pack/Makefile`). Set it
on first clone:

```bash
git clone git@github.com:BuddyTV/lab-code_puppy.git puppy-code
cd puppy-code
git remote add upstream https://github.com/mpfaffenberger/code_puppy.git
git fetch upstream
```

## Branch model

```
upstream/main ──▶ origin/main ──▶ feature branches ──▶ origin/forked-main
   (mpf)         (clean mirror)    (off main)         (Vizio fork; ships)
```

- **`main`** — clean mirror of `upstream/main`. Fast-forward only; no
  Vizio commits ever land here. This is so a feature branch cut off
  `main` is a clean diff against upstream and can be PR'd back without
  carrying our local merges.
- **`forked-main`** — what we actually ship. Periodic `merge
  upstream/main` brings the upstream firehose in; Vizio-only commits
  land here only when they're not appropriate for upstream (e.g. a
  workaround for a deployed-config quirk). **Default branch on
  `BuddyTV/lab-code_puppy` is `forked-main`.**

## Day-to-day: feature development

Cut feature branches off **`main`** so the diff is clean against
upstream. Then merge into `forked-main` to ship.

```bash
# 1. Refresh main against upstream (fast-forward only).
git checkout main
git fetch upstream
git merge --ff-only upstream/main
git push origin main

# 2. Branch off main.
git checkout -b fix/mcp-tool-error-handling

# 3. Hack, commit, run linters.
ruff check --fix && ruff format .
git commit -am "fix(mcp): convert tool exceptions to RetryPromptPart"

# 4. Push to origin (BuddyTV) for internal review.
git push -u origin fix/mcp-tool-error-handling
gh pr create -R BuddyTV/lab-code_puppy --base forked-main

# 5. After internal review, merge into forked-main. The change ships
#    to users on the next `make vendor-puppy && make deploy-pack`
#    in lab-pack.
```

The branch is now also PR-ready against upstream — same commit, no
rebase, no carried Vizio changes. See "Contributing back to upstream"
below for the publish step.

## Pulling upstream updates

Two flows, two destinations:

**`main`** — fast-forward only, never gets Vizio commits:

```bash
git checkout main
git fetch upstream
git merge --ff-only upstream/main
git push origin main
```

If the fast-forward fails, something committed directly to `main` —
investigate, don't `--no-ff` it.

**`forked-main`** — production branch; takes upstream as a merge:

```bash
git checkout forked-main
git fetch upstream
git merge upstream/main
git push origin forked-main
```

`make vendor-puppy` in `lab-pack` does the `forked-main` flow then
rebuilds the wheel and bumps the pin in `lab-pack/pyproject.toml`. Use
the make target when shipping a new pack; the manual sequence is for
when you only want to pick up upstream without cutting a release.

## Contributing back to upstream

`BuddyTV/lab-code_puppy` is **internal-only** — github.com won't accept
its branches as PR head refs against the public upstream. The branch
needs to live on a public repo that GitHub considers a fork of
`mpfaffenberger/code_puppy`.

One-time setup — fork `mpfaffenberger/code_puppy` into your personal
GitHub account via the github.com UI, then add it as a remote:

```bash
git remote add publish git@github.com:<your-user>/code_puppy.git
```

Per-PR — push the same topic branch you used internally:

```bash
git push publish fix/mcp-tool-error-handling
gh pr create -R mpfaffenberger/code_puppy \
    --head <your-user>:fix/mcp-tool-error-handling \
    --base main
```

Because the branch was cut from `main` (which mirrors upstream), the
diff is clean — no Vizio commits sneak into the PR. The internal repo
stays the source of truth; the personal clone is a publish-only mirror
for the head ref.

After upstream merges, the change comes back to us via the standard
`git fetch upstream && git merge upstream/main` flow on `main` and
`forked-main` — no special handling.

## Why this file is separate from AGENTS.md

`AGENTS.md` is upstream-owned. Every commit upstream makes to it is
folded in by the next `git fetch upstream && git merge upstream/main`
on `forked-main`. If we embed Vizio-only content in `AGENTS.md`, every
upstream edit risks a merge conflict on our content even when the
edits are unrelated.

`DEVELOPER.md` (this file) doesn't exist upstream and never will, so it
never participates in upstream merges. Edit freely.

`AGENTS.md` keeps a single line pointing here so a contributor reading
upstream-shaped docs gets routed to the Vizio specifics:

```
> **Vizio fork:** see @DEVELOPER.md for branch model and dev workflow.
```
