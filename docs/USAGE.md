# Using Goosetown

This guide assumes you've installed Goosetown and have a working `./goose` launch. If not, start with [INSTALL.md](INSTALL.md).

## Starting a Session

Always launch goose through the wrapper, never the bare `goose` binary:

```bash
cd /path/to/goosetown
./goose
```

The wrapper enables Goosetown's two coordination primitives — telepathy and the Town Wall — by exporting per-session file paths into the goose process. A bare `goose` invocation won't have these, and delegates won't be able to coordinate.

## Your First Prompt

Once at the goose prompt, tell the main session to load the orchestrator skill. It's the brain that decomposes work and dispatches delegates:

```
Load the goosetown-orchestrator skill.
```

You only need to do this once per session. From then on, every request you make will go through the orchestrator's decompose → dispatch → synthesize loop.

> [!TIP]
> If you forget and just start asking for things, the main goose session will try to do everything itself. You'll get a slower, single-threaded answer. Loading the orchestrator turns one goose into a flock.

## Example Prompts to Try

Goosetown shines on tasks that benefit from parallel research, multi-file edits, or adversarial review. A few starters:

**Research-heavy:**
```
Research how production teams handle OAuth PKCE refresh flows in Rust.
Compare the top 3 crates and write a GUIDE.
```

**Build with review:**
```
Build me a CLI in Python that watches a directory and posts file changes
to a Discord webhook. Include tests. Run the crossfire review before
handing it back.
```

**Investigation:**
```
The dashboard occasionally loses live updates after ~10 minutes.
Investigate the SSE stream code under scripts/goosetown-ui and
ui/js/, propose a root cause with citations, then fix it.
```

**Knowledge consolidation:**
```
Read everything tagged `weatherapp` in CATALOG.md and synthesize
a single landscape document under GUIDES/.
```

When in doubt, describe the *outcome* you want, not the steps. The orchestrator decides which delegates to spawn and in what order.

## What Happens After You Hit Enter

1. **Plan** — the orchestrator decomposes your request into phases.
2. **Research flock** — researchers (local, GitHub, Stack Overflow, arXiv, Reddit, etc.) run in parallel, broadcasting findings on the Town Wall.
3. **Synthesis** — the orchestrator reads the wall, picks an approach, and writes a brief.
4. **Worker flock** — workers claim files and build in parallel, again coordinating on the wall to avoid conflicts.
5. **Crossfire review** — multiple reviewers (often different models) score the result independently. The orchestrator reconciles their feedback.
6. **Hand-back** — you get the final deliverable plus a summary of what changed.

You can watch all of this happen in real time — see [DASHBOARD.md](DASHBOARD.md) or [GTWALL.md](GTWALL.md).

## Working Directory Layout

When the orchestrator and its delegates run, they read and write to these top-level folders. **They are all gitignored** — they're per-installation working state, not shared code.

| Folder | What goes there | Created by |
| --- | --- | --- |
| `GUIDES/` | Actionable runbooks synthesized from research | Writer delegates |
| `PLANS/` | Planning documents for upcoming work | Writer delegates |
| `RESEARCH/` | Research findings and source material | Researcher + writer delegates |
| `WORK_LOGS/` | Session logs — what was tried, learned, decided | Orchestrator |
| `UTILITIES/` | Helper scripts you write or accumulate | You |
| `REPOS/` | Cloned third-party repos for exploration | Researchers |
| `.scratch/` | Throwaway experiments | Anyone |
| `.beads/` | Local issue tracker state ([beads](https://github.com/steveyegge/beads)) | `bd` CLI |
| `CATALOG.md` | Auto-generated index of all knowledge files | `scripts/build-catalog` |
| `TAGS.md` | Canonical tag vocabulary | You |

All knowledge files (anything in `GUIDES/`, `PLANS/`, `RESEARCH/`, `WORK_LOGS/`) use YAML frontmatter — see [AGENTS.md](../AGENTS.md#knowledge-files) for the schema.

To regenerate the catalog after delegates have produced new docs:

```bash
uv run scripts/build-catalog
```

## Skills That Ship with Goosetown

These live in `.claude/skills/` and are auto-discovered by goose when launched from the repo root. The orchestrator calls them out by name when dispatching delegates.

| Skill | Role |
| --- | --- |
| `goosetown-orchestrator` | Coordinates the flock. Load this in your main session. |
| `goosetown-worker` | Builds code, config, scripts |
| `goosetown-writer` | Synthesizes research into GUIDES/PLANS/research docs |
| `goosetown-reviewer` | Quality review with scoring |
| `goosetown-researcher-local` | Searches your local knowledge base |
| `goosetown-researcher-github` | GitHub issues, PRs, code (via `gh`) |
| `goosetown-researcher-stackoverflow` | Stack Exchange API |
| `goosetown-researcher-arxiv` | arXiv papers |
| `goosetown-researcher-reddit` | Reddit (anecdotal evidence) |
| `goosetown-researcher-slack` | Slack via MCP extension |
| `goosetown-researcher-jira` | Jira via `acli` |
| `goosetown-researcher-beads` | Local beads issue tracker |

You normally don't invoke these directly — the orchestrator picks the right ones. But you *can* nudge it: "Send a researcher to Stack Overflow specifically before deciding the approach."

## Ending a Session

Just exit goose (Ctrl+D or `/exit`). The wrapper's cleanup trap deletes the per-session telepathy and Town Wall files automatically.

The dashboard, if you launched one, keeps running until you stop it explicitly:

```bash
./dashboard --stop
```

See [DASHBOARD.md](DASHBOARD.md) for more.

## What to Read Next

- [DASHBOARD.md](DASHBOARD.md) — watch the flock work in real time
- [GTWALL.md](GTWALL.md) — read and post to the Town Wall as a human operator
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — fixes for common problems
- [AGENTS.md](../AGENTS.md) — the reference loaded by agents themselves (useful background)
