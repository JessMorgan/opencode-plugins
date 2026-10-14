# @madsoftwaredev/opencode-goal

A real `/goal` for OpenCode V2: a durable objective that survives compaction and
restarts, stays in front of the model, and re-prompts the session until the goal
is verified done or the loop is stopped.

This package includes a **server plugin**, a **TUI plugin**, and an exported
**RPC contract**. Unlike a Markdown command (which injects one prompt and
disappears), it owns state and mechanics:

| Piece | What it does |
| --- | --- |
| `/goal` command | Set and manage the goal lifecycle |
| `context` hook | Re-injects the goal into every model call (compaction-proof) |
| `goal_*` tools | The agent records progress and ends the loop honestly |
| `session.idle` loop | Re-prompts the session while the goal is open |
| Goal island (TUI) | Fixed status row above the message composer |
| Plugin storage | Goal state persists across restarts and reloads |

## Install

```sh
opencode plugin add @madsoftwaredev/opencode-goal
```

Use OpenCode V2. The supported dependency baseline is `@opencode/plugin`
2.0.27, OpenTUI core/solid 0.5.17, and Solid 1.9.12. The package ships TypeScript
and TSX for OpenCode's plugin loader; it is not a standalone Node CLI.

The server entry belongs in `opencode.json(c)`. The install command adds the
package to your global configuration; for project-local use, add it to your
project's `opencode.jsonc` instead:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": ["@madsoftwaredev/opencode-goal"]
}
```

Preserve your other plugin entries. The connected server advertises the package's
`./tui` entrypoint, so the CLI loads the goal island automatically, including
when connected to a remote server. **Do not configure it only in `cli.json`:**
the command, tools, storage, loop, and RPC require the server plugin. There is
no need to add a second entry in `cli.json`.

Restart the TUI after installation or updates. Server config changes reload
automatically; use `opencode reload` if an explicit reload is needed. If you
already use a local copy, replace its configuration/discovery entry rather than
loading both copies. The runtime IDs remain `goal` (server/RPC) and `goal.tui`
(TUI).

## Commands

| Command | Effect |
| --- | --- |
| `/goal <text>` or `/goal set <text>` | Set (replace) the goal and start working |
| `/goal status` | Show the goal, progress notes, and loop state |
| `/goal update <text>` | Reword the goal; reopens a done goal |
| `/goal continue` or `/goal resume` | Keep going: re-arm the loop and nudge now |
| `/goal pause [reason]` | Pause the loop; the goal stays in context |
| `/goal done [summary]` | Finished: mark done and stop the loop |
| `/goal stop` or `/goal clear` | Abandon: remove the goal entirely |
| `/goal budget <n>` | Set the loop budget for this goal |

Bare lifecycle words (`status`, `continue`, `stop`, …) only act when used alone,
so goal text like `/goal stop the bleeding` is still a goal. Commands such as
`pause`, `done`, and `update` accept arguments as shown above.

## Agent tools

- `goal_status` — read the goal and loop state.
- `goal_progress(note)` — record a checkpoint, decision, or blocker.
- `goal_done(summary)` — end the loop; call only once the goal is verified.

Subtask sessions inherit their parent's goal as context only; the parent session
owns the goal and its progress/completion tools. A goal is scoped to its session,
so set it in the session that should work on it.

## Loop mechanics and safety

1. `/goal ...` stores the goal and kicks off a turn immediately.
2. Every model call in that session gets the goal block injected (goal text,
   progress notes, remaining budget, protocol), so compaction cannot drop it.
3. A nudge fires only when the session is **fully idle**: its turn is over and
   no work is still in flight beneath it — no executing descendant sessions
   (subtasks) and no running background shells. Otherwise the nudge is held and
   fires when the tree settles. This uses OpenCode's session and shell lifecycle
   events and works with or without the background-subagent plugin. Interactive
   PTY terminals are intentionally not treated as work in flight.
4. Each continuation nudge counts against the budget (default **12**). The loop
   pauses with a notice when the budget is spent, on `/goal pause`, or when the
   goal is marked done. `/goal resume` resets the counter and re-arms the loop;
   raising a spent budget past the nudge count also re-arms it. A re-armed nudge
   is held while work is in flight.

**The budget limits continuation nudges, not tokens, cost, time, tool calls, or
work within a turn.** The initial kickoff is not counted. Repeated resumes reset
the counter, and every nudge can start more model/tool work. Pausing or clearing
a goal prevents future loop nudges; it does not interrupt work already running.
Loop messages (`[goal loop] ...`, or `[goal resumed] ...` after startup) are
synthetic messages visible in the transcript and queued behind existing input.

### Reload survival and startup

Goal text, notes, nudges, budget, and status live in durable plugin storage and
survive `opencode reload`, plugin reloads, and service restarts. The in-flight
inventory is persisted too, so held nudges can survive a reload and release
when pre-reload work settles.

**`resumeOnStart` defaults to `true`: loading the server plugin can immediately
resume active goals without waiting for you to open their sessions.** Paused and
done goals are not restarted. Set it to `false` to skip this startup resumption;
it does not disable the idle-event loop for active goals. Pause a goal with
`/goal pause` when you want it to stay paused.

The startup scan currently reads at most 100 stored `goal/` entries. Persisted
in-flight work is reconciled by live lifecycle events, not a fresh inventory
query; if work finishes while the plugin is offline, a stale entry can leave a
nudge held. Persistence is not a guarantee of unattended recovery in that case.

## Goal island (TUI)

A fixed island above the message composer shows the active goal and loop state:

```text
⚑ GOAL · land the migration · loop 3/12
⏸ GOAL · land the migration · paused (budget)
⚑ GOAL (parent) · land the migration · loop 0/12
```

It hides when there is no goal or the goal is done. Subtask sessions show the
parent goal marked `(parent)`. The island reads the server's state over RPC and
refreshes on goal updates and session-status events. If the server plugin is
missing or the RPC request fails, it hides instead of showing stale state.

## Options

Replace the string entry with this object in `opencode.json(c)`, preserving your
other plugins. These are **server** options; the values shown are the defaults:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    {
      "package": "@madsoftwaredev/opencode-goal",
      "options": {
        "budget": 12,          // positive integer: loop nudges per goal
        "resumeOnStart": true, // allow startup/reload resumption of active goals
        "inherit": true        // parent goal context in subtask sessions
      }
    }
  ]
}
```

Invalid budgets or non-boolean flags fail plugin setup. The configured budget
is used for new goals; replacing an existing goal retains its current budget.
Use `/goal budget <n>` to change that session's budget.

## RPC integration

The package exports:

| Import | Purpose |
| --- | --- |
| `@madsoftwaredev/opencode-goal` | Server plugin (default export) |
| `@madsoftwaredev/opencode-goal/tui` | TUI plugin (default export) |
| `@madsoftwaredev/opencode-goal/rpc` | `GoalRpc`, `GoalView`, and display helpers |

For example, inside another TUI plugin's setup function, using its `context`:

```ts
import { GoalRpc } from "@madsoftwaredev/opencode-goal/rpc";

const goal = context.client.rpc(GoalRpc);
const view = await goal.state({ sessionID });
const unsubscribe = goal.events.on("updated", (event) => {
  console.log(event.location, event.data);
});
// Call unsubscribe() during cleanup.
```

`state` returns the session's own goal view or an inherited ancestor goal, or
`{ active: false }` when none exists. `updated` emits the owning session ID and
its goal view; its wire event type is `rpc.goal.updated`. Events are live-only
and include location metadata; external clients should filter by location.
Refresh with `state` after reconnecting and when an ancestor's goal changes.
JSON Schema RPC values are typed as `unknown`; validate them before relying on
the exported `GoalView` shape. This RPC is read-only;
commands and agent tools manage the lifecycle. The server plugin must be running
at the relevant location.

## Develop

From the monorepo root:

```sh
bun test --preload @opentui/solid/preload ./packages/goal
```

Or run `bun run test` from `packages/goal`. The OpenTUI Solid preload is required
to compile the JSX in the TUI plugin and its renderer tests. The existing tests
cover the state machine, server command/tools and idle loop, RPC views/events,
and the rendered TUI island without calling a model or starting a live server.

See the OpenCode V2 guides for [plugins](https://opencode.ai/v2/docs/build/plugins),
[CLI plugins](https://opencode.ai/v2/docs/build/plugins/cli),
[CLI loading](https://opencode.ai/v2/docs/cli/plugins), and
[RPC](https://opencode.ai/v2/docs/build/plugins/rpc).
