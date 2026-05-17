# Troubleshooting

Fixes for the issues new users hit most. If your problem isn't here, [open an issue](https://github.com/aaif-goose/goosetown/issues) or ask on [Discord](https://discord.gg/goose-oss).

## Install / Launch

### `Error: goose not found`

The `./goose` wrapper couldn't find a `goose` binary on your `PATH`.

```bash
# Verify
which goose

# Fix A: install goose
# https://github.com/block/goose/releases

# Fix B: point the wrapper at a specific binary
export GOOSE_BIN=/path/to/goose
./goose
```

### `Error: Goose.app not found at /Applications/Goose.app`

You ran `./goose_gui` without the desktop app installed. Either install [Goose.app](https://github.com/block/goose/releases) for macOS, or use the CLI wrapper `./goose` instead.

### goose starts but asks for environment variables on every launch

goose needs an LLM provider configured the first time. Follow its prompts to set `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / etc. (see the [goose docs](https://block.github.io/goose/) for provider setup). Once configured, the values persist in goose's own config — you shouldn't see the prompt again.

### "command not found: uv"

The dashboard needs [uv](https://docs.astral.sh/uv/). Install it:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

You only need `uv` for the dashboard and for development. Plain `./goose` doesn't need it.

## Telepathy / Town Wall

### Delegates don't seem to coordinate (they duplicate work)

Most likely: you launched goose without the wrapper, so `GOOSE_GTWALL_FILE` and `GOOSE_MOIM_MESSAGE_FILE` aren't set. Verify:

```bash
echo $GOOSE_GTWALL_FILE
echo $GOOSE_MOIM_MESSAGE_FILE
```

Both should print non-empty paths. If they're empty, exit goose and re-launch via `./goose`.

### Desktop Goose doesn't see telepathy

You launched Goose.app from Finder/Spotlight/Dock instead of `./goose_gui`. The env vars are set by the wrapper at launch time; the app inherits them. There's no way to inject them into an already-running app instance.

Quit Goose.app fully, then:

```bash
./goose_gui
```

Keep the terminal open — closing it kills the cleanup trap, leaving stale wall files behind.

### `./gtwall` shows nothing in another terminal

The shell you're running `./gtwall` from doesn't have `GOOSE_GTWALL_FILE` set, so `gtwall` falls back to a default wall that no one else is writing to. Copy the env var from the goose terminal:

```bash
# In the goose terminal:
echo $GOOSE_GTWALL_FILE

# In the other terminal:
export GOOSE_GTWALL_FILE=<paste path>
./gtwall human
```

### Wall messages are stuck / lock held

`gtwall` uses a directory lock (`<wall>.lock/`) with stale detection. If a `gtwall` process crashes mid-write, the next caller should auto-recover after 30s. If it doesn't:

```bash
rm -rf "${GOOSE_GTWALL_FILE}.lock"
```

## Dashboard

### `GOOSE_GTWALL_FILE not set`

Same fix as above — the shell needs the env var that the goose wrapper sets. Launch `./dashboard` from the same terminal as `./goose`, or export the variable manually.

### `No session found`

The dashboard couldn't locate a goose session in `~/.local/share/goose/sessions/sessions.db`. Two reasons:

1. **You haven't said anything in goose yet.** Send one message in your goose chat, then retry.
2. **The sessions DB lives somewhere else.** Pin the session explicitly: `AGENT_SESSION_ID=<your-session-id> ./dashboard`.

### Dashboard launches but never becomes reachable (exit 4)

The `uv run scripts/goosetown-ui ...` process started inside `screen` but didn't bind its port within 5 seconds. Attach to the screen to see why:

```bash
# Find your screen session
screen -ls | grep goosetown-ui

# Attach (read-only, Ctrl-A then d to detach)
screen -r goosetown-ui-<WALL_ID>
```

Common causes: missing Python deps (run `uv sync` once in the repo root), a Python error in the script, or a port-bind collision on a non-default `GOOSETOWN_BIND_HOST`.

### Port is already in use but `--status` says nothing's running

The portfile is stale. Clear it:

```bash
rm -f ~/.goosetown/walls/dashboard-*.port
./dashboard
```

### Dashboard shows another session's data

It auto-resolved to the wrong session. Pin it:

```bash
./dashboard --stop
AGENT_SESSION_ID=<the-id-you-want> ./dashboard
```

### Multiple dashboards keep getting in each other's way

They shouldn't — instances are isolated by `WALL_ID` (derived from `GOOSE_GTWALL_FILE`). Check:

```bash
ls ~/.goosetown/walls/dashboard-*.port
screen -ls | grep goosetown-ui
```

You should see one portfile and one screen session per active goose instance. Stale entries from crashed instances are safe to remove with `rm` + `screen -S <name> -X quit`.

## Performance / Cost

### A flock is taking forever / burning tokens

Use the wall to figure out where it's stuck:

```bash
./gtwall human                # see latest messages
./gtwall --status             # see who's read what
```

If a delegate hasn't read in minutes, it's probably stuck in a tool call. You can cut in:

```bash
./gtwall user "🚨 wrap up now, give me what you have"
```

The orchestrator will see this on its next read and start the wrap-up protocol.

### Crossfire review is dispatching too many reviewers

You can constrain it in your prompt:

```
... and only run a single-reviewer pass, not the full crossfire.
```

Or, to skip review entirely on a small task: `... no review needed, this is throwaway.`

## Knowledge files / CATALOG

### `build-catalog` warns about stale `verified` dates

Guides have an optional `verified: YYYY-MM-DD` field; if it's >90 days old, `build-catalog` flags it. Re-test the guide and bump the date, or remove the field.

### YAML frontmatter parse error

The title contains an unquoted colon. Always quote:

```yaml
---
title: "OAuth: PKCE flow"   # ✓ works
title: OAuth: PKCE flow      # ✗ parser explodes
---
```

See [AGENTS.md](../AGENTS.md#knowledge-files) for the full schema.

## Still Stuck?

1. Check the [GitHub issues](https://github.com/aaif-goose/goosetown/issues) for similar reports.
2. Run `./gtwall --status` and `./dashboard --status` and include their output when filing a bug.
3. Ask in `#goose-oss` on [Discord](https://discord.gg/goose-oss).
