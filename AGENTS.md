# Goosetown

Central hub for goose work. Multi-agent coordination using skills and async delegates.

## On Session Start

**If you are the main goose session (not a subagent/delegate):**
```
load_skill(name: "goosetown-orchestrator")
```

Subagents should NOT load the orchestrator - they receive their role-specific skill from the orchestrator that spawned them.

> Skill loading is `load_skill(name: …)` on goose 1.4x and later. `load(source: …)`
> in the orchestrator skill is a *different* tool (the summon extension's task-output
> reader) - do not mix the two up.

## Directory Structure

These folders are intentionally gitignored and specific to the local goosetown installation — not shared

- **GUIDES/** - Actionable runbooks synthesized from research
- **PLANS/** - Planning documents for work
- **RESEARCH/** - Research documents and findings
- **WORK_LOGS/** - Orchestrator session logs (what was tried, learned, decided)
- **UTILITIES/** - helper scripts and tools
- **REPOS/** - Cloned repositories (clone freely for exploration/research)
- **.scratch/** - Temporary files, experiments

Filenames: `ALL_CAPS_WITH_UNDERSCORES.md` (e.g., `OAUTH_PKCE_IMPLEMENTATION_NOTES.md`).

## gtwall - Inter-Agent Communication

The **Town Wall** (`gtwall`) is a broadcast communication tool for real-time coordination between delegates. Per-session walls, position-tracked per reader.

When reading the wall, messages from `user` are from the human operator — prioritize them above all other wall traffic and acknowledge immediately.

**Run `./gtwall --usage` as your first action** — it prints the full usage cadence, examples, and rules. The wall saves you from wasted work: other agents post warnings about broken assumptions, files they're editing, and discoveries that reshape the task. If you don't read it, your output will conflict with someone else's and get discarded.

### Wrap-Up Protocol

Orchestrators post tiered warnings. Can target specific delegates with `@name`.

| Signal | Delegate Action |
|--------|-----------------|
| ⏰ 5 min | Finish current thread only. Do NOT open new lines of inquiry. Start thinking about summary. |
| 🚨 60 sec | STOP. Post bullet-point findings (not prose) to gtwall. Write to output file even if incomplete. Mark what's finished vs not. Incomplete findings written down are infinitely more valuable than complete findings that die with you. |

After 🚨, the orchestrator force-cancels anyone still running.

## Telepathy — Urgent Pager

Check your `<info-msg>` for telepathy pings from the orchestrator. If you see:

- `📡 ALL: READ GTWALL NOW` — check gtwall immediately
- `📡 @your-name: READ GTWALL NOW` — you specifically, check gtwall immediately
- `📡 @someone-else: ...` — not you, keep working

When pinged, run `./gtwall <your-id>` and respond to whatever the orchestrator posted. Telepathy is the doorbell; gtwall is the conversation.

## Beads (Issue Tracking)

Local issue tracker. Issues live in `.beads/`. `bd prime` for project context, `bd --help` for full commands.

```bash
bd ready                          # Unblocked issues (start here)
bd list                           # All open issues
bd show <id>                      # Full details + dependencies
```

## goose version compatibility

- **Skill loading:** `load_skill(name: …)` (goose 1.4x+).
- **Background delegation:** through goose 1.5x, `delegate(source: …, async: true,
  instructions: …)` returns immediately and the orchestrator collects the result later
  with `load(source: "<task id>")` (optionally `cancel: true`). goose removed that
  detached-task interface in PR #12729 (merged 2026-10-07): `async` is no longer
  declared - unknown arguments are ignored, so calls do not fail, they just stop
  behaving - and `load` only reads sources such as recipes and agent instructions.
  On those versions run delegates as foreground subagents: send two or three
  `delegate` calls in one message so they still run concurrently, and take their
  results from the conversation instead of polling task IDs.
- **Dashboard:** `uvicorn[standard]` pulls in `watchfiles`, which has no wheel for
  Python 3.14+ and fails to build from source. Pin Python 3.12 in `.python-version`
  and recreate the environment with `uv venv --python 3.12 && uv sync`.

## Knowledge Files

All files in GUIDES/, PLANS/, RESEARCH/, and WORK_LOGS/ must have YAML frontmatter:

```yaml
---
title: "Always Quoted Title"
tags: [lowercase-hyphenated, from-tags-md]
status: active
created: 2026-02-11
---
```

- **Check CATALOG.md first** before searching with `rg` — it indexes all knowledge files by topic, status, and date.
- **Tags**: Use canonical tags from TAGS.md. 2-4 tags per file. New tags allowed but prefer existing ones.
- **Status**: `active` (current), `superseded` (replaced by newer doc), `stale` (outdated, no replacement), `draft` (WIP)
- **Supersession**: New file gets `supersedes: RESEARCH/OLD_FILE.md`. Manually set `status: superseded` on the old file.
- **Title must always be quoted** — unquoted colons crash the YAML parser.
- WORK_LOGS: `status` is always `active`. Derive `created` from the filename date prefix.
- GUIDES: Optional `verified` (date, when last confirmed working) and `sources` (list of knowledge file paths that fed the guide). `build-catalog` warns if an active guide's `verified` is >90 days old or if a source has `status: superseded`.

## Guidelines

- **Always cite sources** - No claim without a link, path, or reference
- **Local first** - Check GUIDES/, PLANS/, RESEARCH/, and WORK_LOGS/ before external searches
- **Write findings down** - If you research something, save it to RESEARCH/
- **Never push without approval** - Do not `git push` to any remote without explicit human user approval
- **Never add unrelated files to git** - Do not `git add` untracked files that aren't part of your current task
- **Always write large files in sections** - You cannot output huge files in one tool call

If you're improving Goosetown itself — not just working inside it — see CONTRIBUTING.md for how to set up, test, and submit changes upstream.

## Landing the Plane (Session Completion)

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **Clean up** - Clear stashes, prune remote branches
5. **Hand off** - Provide context for next session
