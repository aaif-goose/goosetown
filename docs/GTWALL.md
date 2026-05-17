# The Town Wall (`gtwall`)

`gtwall` is a per-session broadcast channel that delegates use to coordinate. It's a flat append-only log on disk with per-reader position tracking — the simplest possible group chat.

This page is the **human operator's** guide. For the agent-side protocol, see `./gtwall --usage` or [AGENTS.md](../AGENTS.md#gtwall---inter-agent-communication).

## Watching the Wall

Open a second terminal alongside your goose session. To see anything posted to the wall, you need the same `GOOSE_GTWALL_FILE` your goose session is using:

```bash
# In the terminal where ./goose is running, find the wall file:
echo $GOOSE_GTWALL_FILE
# /Users/you/.goosetown/walls/wall-12345-6789.log

# In the second terminal, export it:
export GOOSE_GTWALL_FILE=/Users/you/.goosetown/walls/wall-12345-6789.log
```

Then tail it however you like. The wall is plain text — one message per line, fields separated by `|`:

```bash
tail -f "$GOOSE_GTWALL_FILE"
```

Or use `./gtwall` itself, which formats messages prettily and tracks your read position:

```bash
./gtwall human                 # show only messages new since last read
./gtwall --reset human         # forget read position; next call shows everything
```

The dashboard (see [DASHBOARD.md](DASHBOARD.md)) renders the wall too, with timestamps and color.

## Posting as a Human

You can post to the wall and the orchestrator will see it. Messages from `user` are treated as high-priority (the orchestrator's skill tells it to acknowledge user messages immediately):

```bash
./gtwall user "Pivot: the customer wants Postgres, not SQLite. Adjust."
./gtwall user "@worker-auth stop, that file is being rewritten in PR #42"
```

Use `@<delegate-id>` to address a specific delegate. The orchestrator will see your post on its next wall read and route accordingly.

## Commands

```bash
./gtwall <id> "message"   # post + show new messages
./gtwall <id>             # show new messages only
./gtwall --reset <id>     # reset read position for <id>
./gtwall --clear          # clear this session's wall
./gtwall --status         # message count + per-reader unread counts
./gtwall --list           # all session walls on this machine
./gtwall --help           # help
./gtwall --usage          # the agent-side protocol
```

`<id>` rules:
- Letters, numbers, dash (`-`), underscore (`_`) only
- Non-empty
- Pick something stable so your read position persists across calls. `human` or `operator` works fine.

## Telepathy — the Pager

The Town Wall is the conversation; **telepathy** is the doorbell. The orchestrator uses telepathy to ping delegates that should drop what they're doing and read the wall *now*:

- `📡 ALL: READ GTWALL NOW` — everyone, check the wall
- `📡 @worker-auth: READ GTWALL NOW` — that specific delegate

Telepathy is a single file (`GOOSE_MOIM_MESSAGE_FILE`, set by the wrapper) that delegates poll via their `tom` extension. You don't need to interact with it directly — it's an implementation detail. But it explains why the wrapper sets that env var.

## Where Things Live

| Path | What |
| --- | --- |
| `~/.goosetown/walls/wall-<pid>-<rand>.log` | The wall file for one goose session |
| `~/.goosetown/walls/wall-<pid>-<rand>.positions/<id>.pos` | Per-reader read position |
| `~/.goosetown/walls/dashboard-<wall-id>.port` | Portfile written by the dashboard |
| `/tmp/goose-telepathy-<pid>.txt` | Per-session telepathy file |

The wrappers create these on launch and delete them on clean exit. If something crashes, you may have orphaned files — they're safe to delete by hand:

```bash
rm -rf ~/.goosetown/walls
rm -f /tmp/goose-telepathy-*.txt
```

## Wrap-Up Signals

The orchestrator posts time warnings as it approaches deadlines:

| Signal | Meaning |
| --- | --- |
| ⏰ 5 min | Finish current thread, no new lines of inquiry, start summarizing |
| 🚨 60 sec | STOP. Post bullet-point findings and write to your output file even if incomplete |

After 🚨, anything still running gets force-cancelled. If you see these in your wall view, the flock is wrapping up.

## When the Wall is Useful to You

- **Catching the flock pivoting** — you can see "💡 found that crate is deprecated" before the orchestrator surfaces it
- **Stopping work early** — `./gtwall user "stop, requirements changed"` is faster than waiting for goose's next turn
- **Auditing what happened** — the wall is an immutable per-session log
- **Debugging a stuck flock** — `./gtwall --status` shows who has read what; a delegate that never reads is probably hung
