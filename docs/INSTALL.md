# Installing Goosetown

Goosetown is a thin coordination layer on top of [goose](https://github.com/block/goose). You don't install Goosetown into your system — you clone the repo and launch goose through its wrapper scripts.

## System Requirements

| Component | Why | Notes |
| --- | --- | --- |
| **macOS or Linux** | Wrappers are bash + use `screen`, `lsof`, `mkfifo` semantics | Windows is not supported; use WSL2 |
| **bash 4+** | Wrapper scripts | macOS ships 3.x — `brew install bash` or use the system at `/bin/bash` (it works in compat mode) |
| **[goose](https://github.com/block/goose/releases) v1.25.0+** | The agent runtime itself | CLI binary on `PATH`, or set `GOOSE_BIN=/path/to/goose` |
| **[uv](https://docs.astral.sh/uv/)** | Runs the dashboard (`scripts/goosetown-ui`) | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| **`screen`** | Detached process for the dashboard | Preinstalled on macOS; `apt install screen` on Debian/Ubuntu |
| **`lsof`, `curl`, `sqlite3`** | Dashboard port checks and session resolution | Standard on macOS; `apt install lsof curl sqlite3` on Linux |
| **An LLM provider for goose** | goose itself needs API access to a model | Anthropic, OpenAI, Databricks, Ollama, etc. See the [goose docs](https://block.github.io/goose/) |

You do **not** need Python, Node.js, or `uv` unless you want to run the dashboard or contribute code. The orchestrator and delegates run entirely inside goose.

## Install

```bash
git clone https://github.com/aaif-goose/goosetown.git
cd goosetown
```

That's it. Nothing to build. The repo ships executable wrappers (`goose`, `goose_gui`, `gtwall`, `dashboard`) at the top level.

If you plan to launch goose from outside the repo, add the directory to your `PATH` or symlink the wrappers:

```bash
ln -s "$PWD/goose" ~/.local/bin/goosetown
ln -s "$PWD/dashboard" ~/.local/bin/goosetown-dashboard
ln -s "$PWD/gtwall" ~/.local/bin/gtwall
```

## First Run

```bash
./goose
```

The wrapper:

1. Locates your `goose` binary (or honors `GOOSE_BIN`).
2. Creates a per-session telepathy file at `/tmp/goose-telepathy-$$.txt`.
3. Creates a per-session Town Wall at `~/.goosetown/walls/wall-<pid>-<rand>.log`.
4. Exports `GOOSE_MOIM_MESSAGE_FILE` and `GOOSE_GTWALL_FILE` so delegates spawned inside goose can find them.
5. Launches goose. On exit, the telepathy and wall files are deleted.

If goose has never been configured on this machine, it will prompt you for a provider and API key before starting the chat. Follow its prompts, then re-run `./goose`.

Once at the prompt, see [USAGE.md](USAGE.md) for what to type next.

## GUI vs CLI

Goosetown supports both the goose CLI and the goose desktop app.

| | `./goose` (CLI) | `./goose_gui` (Desktop) |
| --- | --- | --- |
| What it launches | Terminal chat | The Goose.app on macOS at `/Applications/Goose.app` |
| Telepathy + Town Wall | ✅ Always on | ✅ Only when launched via this script |
| Survives terminal close | ❌ Chat dies with terminal | ❌ Keep terminal open — script blocks until Ctrl+C, then cleans up |
| Recommended for | Power users, scripting, server use | Visual chat experience, multiple chat tabs |

> [!IMPORTANT]
> Launching the desktop Goose.app directly from Finder, Spotlight, or the Dock will **not** enable Goosetown's telepathy or per-session wall. You must launch via `./goose_gui` from a terminal, and keep that terminal open.

## Environment Variables

All variables are optional; the wrappers set sensible defaults.

| Variable | Read by | Default | Purpose |
| --- | --- | --- | --- |
| `GOOSE_BIN` | `./goose` | First `goose` on `PATH` | Override the goose binary path |
| `GOOSE_MOIM_MESSAGE_FILE` | `./goose`, `./goose_gui`, tom extension | Auto-generated per session | Telepathy channel for orchestrator → delegate pings |
| `GOOSE_GTWALL_FILE` | `./goose`, `./goose_gui`, `./gtwall`, `./dashboard` | Auto-generated per session | The per-session Town Wall file |
| `GTWALL_DIR` | `./gtwall` | `~/.goosetown` | Root directory for walls and positions |
| `GOOSETOWN_BIND_HOST` | `./dashboard` | `127.0.0.1` | Network interface the dashboard binds to. Change with care — see [DASHBOARD.md](DASHBOARD.md#security) |
| `AGENT_SESSION_ID` | `./dashboard` | Auto-resolved from `~/.local/share/goose/sessions/sessions.db` | Pin the dashboard to a specific goose session |

The wrappers set `GOOSE_MOIM_MESSAGE_FILE` and `GOOSE_GTWALL_FILE` automatically per session — you generally don't set them yourself. They exist so that subprocesses (delegates, the dashboard, manual `./gtwall` calls in another shell) can join the same session.

## Updating

```bash
cd goosetown
git pull
```

There is no build step. Wrappers are bash; the dashboard uses `uv` which auto-syncs Python deps on next launch.

## Uninstalling

```bash
# Remove the repo
rm -rf /path/to/goosetown

# Remove per-user state (walls, positions, dashboard portfiles)
rm -rf ~/.goosetown

# Remove any symlinks you created
rm ~/.local/bin/goosetown ~/.local/bin/goosetown-dashboard ~/.local/bin/gtwall
```

Goose itself is uninstalled separately — see the [goose docs](https://block.github.io/goose/).
