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

### Pulling upstream updates

`forked-main` tracks upstream. Periodic merges keep us close to head:

```bash
git checkout forked-main
git fetch upstream
git merge upstream/main
git push origin forked-main
```

`make vendor-puppy` in `lab-pack` does this same sequence then rebuilds
the wheel and bumps the pin in `lab-pack/pyproject.toml`. Use the make
target when shipping a new pack; the manual sequence above is for
when you only want to pick up upstream without cutting a release.

### Contributing back to upstream

`BuddyTV/lab-code_puppy` is **internal-only** — github.com won't accept
its branches as PR head refs against the public upstream. To submit a
fix back:

1. Push the topic branch to a public personal fork of upstream as well:
   ```bash
   git remote add publish git@github.com:<your-user>/forked-puppycode.git
   git push publish fix/<branch>
   ```
2. Open the PR from there:
   ```bash
   gh pr create -R mpfaffenberger/code_puppy \
       --head <your-user>:fix/<branch> --base main
   ```

The internal repo stays the source of truth; the personal clone is a
publish-only mirror for the head ref.
