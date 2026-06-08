# Contributing to Code Puppy

> **Golden rule:** nearly all new functionality should be a **plugin** under `code_puppy/plugins/`
> that hooks into core via `code_puppy/callbacks.py`. Don't edit `code_puppy/command_line/`.

## How Plugins Work

Create `code_puppy/plugins/my_feature/register_callbacks.py` (builtin) or `~/.code_puppy/plugins/my_feature/register_callbacks.py` (user):

```python
from code_puppy.callbacks import register_callback

def _on_startup():
    print("my_feature loaded!")

register_callback("startup", _on_startup)
```

That's it. The plugin loader auto-discovers `register_callbacks.py` in subdirs.

## Available Hooks

`register_callback("<hook>", func)` — deduplicated, async hooks accept sync or async functions.

| Hook | When | Signature |
|------|------|-----------|
| `startup` | App boot | `() -> None` |
| `shutdown` | Graceful exit | `() -> None` |
| `invoke_agent` | Sub-agent invoked | `(*args, **kwargs) -> None` |
| `agent_exception` | Unhandled agent error | `(exception, *args, **kwargs) -> None` |
| `agent_run_start` | Before agent task | `(agent_name, model_name, session_id=None) -> None` |
| `agent_run_end` | After agent run | `(agent_name, model_name, session_id=None, success=True, error=None, response_text=None, metadata=None) -> None` |
| `load_prompt` | System prompt assembly | `() -> str \| None` |
| `run_shell_command` | Before shell exec | `(context, command, cwd=None, timeout=60) -> dict \| None` (return `{"blocked": True}` to block) |
| `file_permission` | Before file op | `(context, file_path, operation, ...) -> bool` |
| `pre_tool_call` | Before tool executes | `(tool_name, tool_args, context=None) -> Any` |
| `post_tool_call` | After tool finishes | `(tool_name, tool_args, result, duration_ms, context=None) -> Any` |
| `custom_command` | Unknown `/slash` cmd | `(command, name) -> True \| str \| None` |
| `custom_command_help` | `/help` menu | `() -> list[tuple[str, str]]` |
| `register_tools` | Tool registration | `() -> list[dict]` with `{"name": str, "register_func": callable}` |
| `register_agents` | Agent catalogue | `() -> list[dict]` with `{"name": str, "class": type}` |
| `register_model_type` | Custom model type | `() -> list[dict]` with `{"type": str, "handler": callable}` |
| `register_skills` | Skill catalogue | `() -> list[dict]` with `{"name": str, "skill_md" \| "skill_md_path" \| "frontmatter"+"body"}` |
| `load_model_config` | Patch model config | `(*args, **kwargs) -> Any` |
| `load_models_config` | Inject models | `() -> dict` |
| `load_model_descriptions` | Inject description overlays | `() -> dict[str, str]` |
| `get_model_system_prompt` | Per-model prompt | `(model_name, default_prompt, user_prompt) -> dict \| None` |
| `stream_event` | Response streaming | `(event_type, event_data, agent_session_id=None) -> None` |
| `pre_mcp_autostart` | Before bound MCP servers auto-start | `(agent_name, server_names) -> None` (refresh tokens / mint creds here) |

Full list + rarely-used hooks: see `code_puppy/callbacks.py` source.

## Rules

1. **Plugins over core** — if a hook exists for it, use it
2. **One `register_callbacks.py` per plugin** — register at module scope
3. **600-line hard cap** — split into submodules
4. **Fail gracefully** — never crash the app
5. **Return `None` from commands you don't own**
6. **Always run linters - `ruff check --fix`, `ruff format .`
7. **NEVER ALLOW A CLAUDE CO-AUTHOR COMMIT**

## Vizio fork (BuddyTV/lab-code_puppy)

This branch (`forked-main`) is the Vizio downstream of
[`mpfaffenberger/code_puppy`](https://github.com/mpfaffenberger/code_puppy).
Origin lives at **`BuddyTV/lab-code_puppy`** (internal repo) — that's
the canonical home, replacing the previous personal fork at
`breedx/forked-puppycode`. The upstream maintainer's repo is unchanged.

The wheel built from `forked-main` is vendored into
[`lab-pack`](https://github.com/BuddyTV/lab-pack) via `make vendor-puppy`
and shipped to users as part of the lab stack. Don't edit `forked-main`
unless the change should land in production.

### Remotes

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

### Branch model

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

### Day-to-day: feature development

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

### Pulling upstream updates

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

### Contributing back to upstream

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
