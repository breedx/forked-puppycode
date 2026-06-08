# lab-code_puppy — Developer Guide (VIZIO fork)

## ⚠️ Treat this repo as public. No secrets, ever.

Every push to `main` or `forked-main` on `BuddyTV/lab-code_puppy`
auto-mirrors to **`breedx/forked-puppycode`, which is on the public
internet**. Anything you commit to those two branches is public from
that moment on — there is no "delete and it goes away," GitHub keeps
refs reachable by SHA for ~90 days even after you force-push.

**Never commit:**

- API keys, tokens, passwords, JWTs, signing keys, deploy keys
- AWS / Azure / GCP credentials (in code, in tests, in fixtures)
- `.env` files, `puppy.cfg` with real values, anything from `~/.lab/`
- Internal hostnames (`*.vizio.com`, `*.vizio-rad.com`, etc.)
- Internal URLs, Slack permalinks, Jira tickets with sensitive titles,
  paste-bin output, screenshots from internal tools
- Customer names, employee names, IP addresses from internal networks
- Output of internal API calls (responses, logs, traces)

**If you commit a secret, assume it's compromised.** Rotate it
immediately at the source (regenerate the AWS key, mint a new JWT,
etc.). Don't try to scrub history first — that takes hours and the
secret is already in someone's clone or GitHub's CDN.

**Topic branches are not mirrored** — they stay internal on BuddyTV
unless you explicitly `git push breedx <branch>` (see
"Contributing back to upstream"). Use a topic branch for anything
sensitive while you sort it out.

## Repos at a glance

| Repo | Visibility | Role | Auto-mirrored |
|---|---|---|---|
| [`BuddyTV/lab-code_puppy`](https://github.com/BuddyTV/lab-code_puppy) | **internal** | Source of truth. Default branch `forked-main`. | n/a — origin |
| [`mpfaffenberger/code_puppy`](https://github.com/mpfaffenberger/code_puppy) | **public** | Upstream maintainer's repo. Pull only; never push. | n/a |
| [`breedx/forked-puppycode`](https://github.com/breedx/forked-puppycode) | **public** | PR-staging surface for upstream contributions. | `main` and `forked-main` (every push) |

`main` is byte-identical to `mpfaffenberger/main` — mirroring it
exposes nothing new. `forked-main` carries upstream + VIZIO commits —
that's the surface where leaks happen. Review every commit going in.

## What this file is

VIZIO-only. Not present in upstream `mpfaffenberger/code_puppy` and
never will be — keeping our content out of `AGENTS.md` is what makes
upstream merges conflict-free.

If you came here from `AGENTS.md`'s VIZIO-fork pointer: this is the
right place. If you're looking for the upstream plugin contract, hooks
table, or core rules, that's still in `AGENTS.md`.

## What is this branch?

`forked-main` on `BuddyTV/lab-code_puppy` is the VIZIO downstream of
`mpfaffenberger/code_puppy`. Origin lives at **`BuddyTV/lab-code_puppy`**
(internal repo) — the canonical home, replacing the previous personal
fork at `breedx/forked-puppycode` (which is now demoted to "public
mirror used as a PR-staging surface", per the table above).

The wheel built from `forked-main` is vendored into
[`lab-pack`](https://github.com/BuddyTV/lab-pack) via `make vendor-puppy`
and shipped to users as part of the lab stack. Don't edit `forked-main`
directly unless the change should land in production.

## Remotes

```
origin    git@github.com:BuddyTV/lab-code_puppy.git   # VIZIO fork (internal)
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
   (mpf)         (clean mirror)    (off main)         (VIZIO fork; ships)
```

- **`main`** — clean mirror of `upstream/main`. Fast-forward only; no
  VIZIO commits ever land here. This is so a feature branch cut off
  `main` is a clean diff against upstream and can be PR'd back without
  carrying our local merges.
- **`forked-main`** — what we actually ship. Periodic `merge
  upstream/main` brings the upstream firehose in; VIZIO-only commits
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
rebase, no carried VIZIO changes. See "Contributing back to upstream"
below for the publish step.

## Pulling upstream updates

Two flows, two destinations:

**`main`** — fast-forward only, never gets VIZIO commits:

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

`BuddyTV/lab-code_puppy` is internal-only — github.com won't accept its
branches as PR head refs against the public upstream. We use
`breedx/forked-puppycode` as a public PR-staging surface.

`main` and `forked-main` mirror automatically (see "Mirror automation"
below). For topic branches you want to send upstream, push them
manually:

```bash
# One-time: add breedx as a remote.
git remote add breedx git@github.com:breedx/forked-puppycode.git

# Per-PR: push your topic branch to breedx, then PR upstream.
git push breedx fix/mcp-tool-error-handling
gh pr create -R mpfaffenberger/code_puppy \
    --head breedx:fix/mcp-tool-error-handling \
    --base main
```

Because the branch was cut from `main` (which mirrors upstream), the
diff is clean — no VIZIO commits sneak into the PR. After upstream
merges, the change comes back via the standard `git fetch upstream &&
git merge upstream/main` flow.

Manually-pushed topic branches stay on `breedx` until you delete them
(`git push breedx --delete <branch>`). The mirror automation only
manages `main` and `forked-main`; nothing GCs topic branches for you.

## Mirror automation

`.github/workflows/mirror-vizio-fork.yml` runs on every push to `main`
or `forked-main` on `BuddyTV/lab-code_puppy` and pushes that ref to
`breedx/forked-puppycode`. Auth is an ed25519 deploy keypair: public
half on breedx as a write deploy key, private half on BuddyTV as the
`BREEDX_DEPLOY_KEY` repo secret.

The workflow filename is VIZIO-specific so upstream merges never
conflict on it. Upstream's own workflows (`ci.yml`, `publish.yml`,
`pypi-downloads.yml`) are **disabled** at the repo-settings level
(state lives outside git, files are byte-identical to upstream — also
conflict-free).

Rotation — generate a new ed25519 pair, replace the deploy key on
`breedx/forked-puppycode`, set the new private half as
`BREEDX_DEPLOY_KEY` on `BuddyTV/lab-code_puppy`. The old key stops
working the moment the deploy key is removed.

To disable the mirror entirely:

```bash
gh workflow disable "Mirror to breedx fork" -R BuddyTV/lab-code_puppy
```

⚠️ Disabling the mirror does NOT scrub history that's already public on
breedx. If something internal leaks through `forked-main`, deal with
it on breedx itself (delete branch + force-push if you must, but
remember GitHub keeps refs reachable for ~90 days via direct SHA).

## Why this file is separate from AGENTS.md

`AGENTS.md` is upstream-owned. Every commit upstream makes to it is
folded in by the next `git fetch upstream && git merge upstream/main`
on `forked-main`. If we embed VIZIO-only content in `AGENTS.md`, every
upstream edit risks a merge conflict on our content even when the
edits are unrelated.

`DEVELOPER.md` (this file) doesn't exist upstream and never will, so it
never participates in upstream merges. Edit freely.

`AGENTS.md` keeps a single line pointing here so a contributor reading
upstream-shaped docs gets routed to the VIZIO specifics:

```
> **VIZIO fork:** see @DEVELOPER.md for branch model and dev workflow.
```
