# lab-code_puppy — Developer Guide (VIZIO fork)

## ⚠️ Pushing to the public mirror is a deliberate act.

Daily work in `BuddyTV/lab-code_puppy` stays **internal** — pushes to
any branch (including `main` and `forked-main`) do not leave the
BuddyTV org automatically.

Going public requires an explicit action: running the
**"Mirror to breedx fork"** workflow against `main` or `forked-main`.
That pushes the chosen ref to **`breedx/forked-puppycode`**, which is
on the public internet. The button is the gate; auto-mirror was
removed precisely because every push was a potential leak window.

**Before clicking the button, audit the ref. No secrets, ever.**

Once a commit lands on the public mirror it's public for keeps —
GitHub keeps refs reachable by SHA for ~90 days even after force-push,
and clones / CDN copies are out of your control immediately.

**Never commit:**

- API keys, tokens, passwords, JWTs, signing keys, deploy keys
- AWS / Azure / GCP credentials (in code, in tests, in fixtures)
- `.env` files, `puppy.cfg` with real values, anything from `~/.lab/`
- Internal hostnames (`*.vizio.com`, `*.vizio-rad.com`, etc.)
- Internal URLs, Slack permalinks, Jira tickets with sensitive titles,
  paste-bin output, screenshots from internal tools
- Customer names, employee names, IP addresses from internal networks
- Output of internal API calls (responses, logs, traces)

**If you commit a secret to a branch that's been mirrored**, assume
it's compromised. Rotate immediately at the source (regenerate the
AWS key, mint a new JWT, etc.). Don't try to scrub history first —
that takes hours and the secret is already gone.

If a secret was committed to a branch that hasn't been mirrored yet,
you're fine — just rewrite the branch on BuddyTV before mirroring.

## Repos at a glance

| Repo | Visibility | Role | How content gets there |
|---|---|---|---|
| [`BuddyTV/lab-code_puppy`](https://github.com/BuddyTV/lab-code_puppy) | **internal** | Source of truth. Default branch `forked-main`. | Daily work; PRs internal |
| [`mpfaffenberger/code_puppy`](https://github.com/mpfaffenberger/code_puppy) | **public** | Upstream maintainer's repo. Pull only; never push. | n/a |
| [`breedx/forked-puppycode`](https://github.com/breedx/forked-puppycode) | **public** | PR-staging surface for upstream contributions. | Manual: button-press for `main`/`forked-main`, `git push breedx <branch>` for topic branches |

`main` is byte-identical to `mpfaffenberger/main` — mirroring it
exposes nothing new. `forked-main` carries upstream + VIZIO commits —
that's the surface where leaks happen. Audit before mirroring.

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

Step 4 stays internal — internal-repo PRs don't expose anything
publicly. The branch is also PR-ready against upstream when you want
to send it; see "Contributing back to upstream" below for the publish
step (it's a separate, opt-in act).

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

Neither sequence pushes to `breedx` — both stay internal. Mirror
manually if/when you want to expose the new state publicly.

## Contributing back to upstream

`BuddyTV/lab-code_puppy` is internal-only — github.com won't accept its
branches as PR head refs against the public upstream. We use
`breedx/forked-puppycode` as a public PR-staging surface.

`main` and `forked-main` are mirrored by **manual button-press**
(see "The mirror button" below). Topic branches don't go through the
button at all — push them directly:

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
(`git push breedx --delete <branch>`). The mirror button only handles
`main` and `forked-main`; nothing GCs topic branches for you.

## The mirror button

`.github/workflows/mirror-vizio-fork.yml` is **manual-only**. It
**does not** trigger on push. Mirroring to a public repo is a
deliberate act per ref. To run it:

```bash
# Mirror forked-main as it stands on BuddyTV right now.
gh workflow run "Mirror to breedx fork" \
    -R BuddyTV/lab-code_puppy --ref forked-main

# Same for main.
gh workflow run "Mirror to breedx fork" \
    -R BuddyTV/lab-code_puppy --ref main
```

Or click "Run workflow" in the BuddyTV repo's Actions tab and pick
the ref from the dropdown.

Auth is an ed25519 deploy keypair: public half on breedx as a write
deploy key, private half on BuddyTV as the `BREEDX_DEPLOY_KEY` repo
secret.

The workflow file gets mirrored to breedx along with the rest of the
tree, so the workflow technically exists on breedx too. A
`if: github.repository == 'BuddyTV/lab-code_puppy'` guard at the job
level makes it skip immediately on breedx — no perms needed there.

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
breedx. If something internal leaked through a previous mirror run,
deal with it on breedx itself (delete branch + force-push if you must,
but remember GitHub keeps refs reachable for ~90 days via direct SHA).

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
