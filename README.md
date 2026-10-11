# prime-worker

An agent skill for [Claude Code](https://claude.com/claude-code), [Codex](https://developers.openai.com/codex), [Pi](https://github.com/earendil-works/pi), [OpenCode](https://opencode.ai) and [Antigravity](https://antigravity.google) for delegating **multi-turn** work to a local [`prime-agent`](https://github.com/PrimeIntellect-ai/prime-agent) subagent that remembers context across turns.

Your agent spawns a cheap model as a named worker, keeps talking to it, branches it, and lets it learn from its own mistakes — while the expensive context stays free for the work that needs it.

```bash
pw spawn auth-review --cwd ~/proj/src/auth --model deepseek-v4-flash --cite "Map the token refresh path."
pw say auth-review "Now trace where the refresh token is persisted."
pw fork auth-review auth-alt "What breaks if the TTL drops to 60s?"
pw refine auth-review          # let it record what it learned
pw retire auth-review --yes
```

## The idea: a worker is a named directory

`~/.claude/prime-workers/<name>/` holds one session `.jsonl` plus a `worker.env` recording the `--cwd`, `--model` and flags it was spawned with. The name is the only handle you need — no session ids, no `ls -t` races, and concurrent workers cannot collide.

That matters more than it sounds. Sessions are UUID-named `.jsonl` files, so without a convention you end up carrying ids between turns, racing `ls -t` whenever two workers run at once, and guessing which of thirty files in the store is the 50MB one. A named directory gives each worker a stable, readable handle, lets concurrent workers coexist, records the flags it was spawned with so follow-up turns need none, and makes retiring one an `rm -rf`.

## Requirements

- [`prime-agent`](https://github.com/PrimeIntellect-ai/prime-agent) on your `PATH` (developed against 0.8.0)
- A configured model provider. Defaults target an **OpenCode Go** subscription with `deepseek-v4-flash`; override with `PW_PROVIDER` and `PW_DEFAULT_MODEL` for any provider `prime-agent` supports.
- `bash`, `python3`

## Install

Every host below loads the same `skills/prime-worker/` folder, so the skill and `pw` arrive together.

### Claude Code

```
/plugin marketplace add alperiox/prime-worker
/plugin install prime-worker@alperiox
```

Claude Code keeps it updated.

### Codex

```bash
codex plugin marketplace add alperiox/prime-worker
codex plugin add prime-worker@alperiox
```

Start a fresh session afterwards; the skill appears as `prime-worker:prime-worker`.

### Pi

```bash
pi install git:github.com/alperiox/prime-worker
```

### OpenCode

OpenCode has no plugin catalog for skills; point its config at a clone instead. Merge this into your existing `opencode.json`, using an absolute path:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "skills": {
    "paths": ["/absolute/path/to/prime-worker/skills"]
  }
}
```

### Antigravity

`agy` installs plugins from a local directory, so clone first:

```bash
git clone https://github.com/alperiox/prime-worker ~/src/prime-worker
agy plugin install ~/src/prime-worker
```

### By hand

If you would rather not use plugins, clone it anywhere and link the skill into `~/.claude/skills/`:

```bash
git clone https://github.com/alperiox/prime-worker ~/src/prime-worker
ln -s ~/src/prime-worker/skills/prime-worker ~/.claude/skills/prime-worker
~/src/prime-worker/skills/prime-worker/scripts/pw --help
```

Claude Code picks the skill up from `~/.claude/skills/`. `pw` also works standalone from any shell, with or without Claude.

Put it on your `PATH` if you like:

```bash
ln -s ~/src/prime-worker/skills/prime-worker/scripts/pw ~/.local/bin/pw
```

## Commands

| | |
|---|---|
| `pw spawn <name> --cwd <dir> --model <m> "<task>"` | create a worker; records cwd, model and flags |
| `pw say <name> "<follow-up>"` | resume with context intact — no flags to repeat |
| `pw fork <src> <dst> "<what-if>"` | branch without disturbing the trunk |
| `pw list` | name, model, turns, tokens, size, cwd |
| `pw history <name> [n]` | last n exchanges, tool noise stripped |
| `pw refine <name> [--global]` | let the worker learn from its own session |
| `pw learned [<name>]` | inspect what it learned |
| `pw retire <name> --yes` | delete the worker |
| `pw models [pattern]` | live model list from `prime-agent` |
| `pw unwedge` | recover a stuck session |

Prompts come from trailing arguments **or stdin**, so a long brief goes in as a heredoc instead of a quoting nightmare:

```bash
pw spawn refactor --cwd ~/proj/src <<'BRIEF'
Multi-paragraph task description, freely containing "quotes", $vars and `backticks`.
BRIEF
```

### Citations (`--cite`)

Injects a contract requiring every claim to carry a `path:line` citation, an explicit `UNVERIFIED` marker where it cannot, and a closing `## Not read` section exposing coverage gaps. It is the default for review and analysis work, persisted per worker, and disabled with `--no-cite`.

This is what turns a cheap model's output from prose you must trust into claims you can check in seconds. Expect citations to land in the right neighbourhood — they are occasionally off by a line or two, but not fabricated.

### Autonomous mode (`--gate`)

```bash
pw spawn fixer --cwd ~/proj --gate "pytest -q" --gate "ruff check ." --max-turns 12 \
  "Fix the failing tests in tests/test_parser.py. Do not modify any file under tests/."
```

The worker loops, feeding each failure back to itself, until every gate exits 0 or a limit trips. Gates are shell commands run in the worker's `--cwd` and persist across turns.

**A gate proves a command exits 0. It does not prove the work is correct.** The worker can read its own gate and reason about what would satisfy it — the classic failure is making tests pass by editing the tests. Name off-limits paths in the prompt, and read the diff regardless.

**Run your gate by hand first.** A gate that can never pass burns every retry before giving up, and a typo'd path is enough to cause it. Current `prime-agent` ends such a run cleanly, exiting non-zero with `autonomous limit reached: maxContinuations reached`; older builds could stall instead, so every `pw` run is also bounded by a wall clock (`PW_TIMEOUT`, default 900s, exit 124).

### Refinement

`prime-agent` can inspect its own trajectory and write durable notes that future runs read automatically — environment quirks, tool APIs, path conventions. In practice it picked up things like *"`python` is not on PATH, use `python3`"* and the exact signature of the kernel's file-edit function, both learned from real failures.

`pw refine <name>` is **local** by default and cannot affect anything outside that worker. `--global` promotes to `~/.prime/agent/harness/`, which *every* `prime-agent` session reads — so the skill instructs Claude to show you the entries and ask first. Promotions are reversible (`pw refine <name> rollback <id> --global`) and the audit log is append-only.

A useful tell, not a rule: **if an entry names a repository, a path, or a line number, it probably isn't durable.**

## Safety

`--cwd` is the only sandbox boundary, and `prime-agent`'s sole tool is `ipython` — it reads and edits files by executing Python. Delegating grants arbitrary code execution inside `--cwd`, and file contents leave your machine to whichever provider you configured.

Scope `--cwd` to the narrowest directory the task needs. For broad or risky changes, spawn against a `git worktree` and review the branch diff.

## Environment

| variable | default |
|---|---|
| `PW_HOME` | `~/.claude/prime-workers` |
| `PW_PROVIDER` | `opencode-go` |
| `PW_DEFAULT_MODEL` | `deepseek-v4-flash` |
| `PW_TIMEOUT` | `900` |
| `PW_AUTO_TIMEOUT` | `900` |
| `PW_REFINE_TIMEOUT` | `600` |

## License

MIT
