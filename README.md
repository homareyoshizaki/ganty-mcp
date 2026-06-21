# Ganty MCP

[Ganty](https://ganty.app) is a Gantt chart SaaS with a built-in **Model Context Protocol (MCP)** server that lets AI clients like Claude Desktop and Claude Code drive your projects in natural language — and computes critical paths and shift-impact server-side rather than asking the LLM to do math.

> **Status: v1.0.0** — production endpoint at `https://ganty.app/api/mcp` (Streamable HTTP)
> Free on every Ganty plan including Free.

## What makes this different

Most MCP servers expose CRUD over an existing API. Ganty's also exposes **calculation tools that compute on the server and return JSON** — so the LLM never has to reason about dates, dependencies, or critical paths.

- `get_critical_path` — forward/backward-pass CPM with progress-aware remaining duration. Returns the critical path tasks in order, per-task earliest/latest start/finish, slack, projected end date. Detects cycles before computing and returns `error: 'cyclic_dependency'` instead of hanging or guessing.
- `reschedule_and_propagate` — simulate or commit a task shift. Default mode is `dry_run` (no DB writes). Push-only cascade respects pinned tasks (anything at `progress=100` or in `pinned_task_ids`) and reports `conflicts` instead of overwriting. `commit` mode wraps everything in an all-or-nothing transaction.

## Quick start

1. **Issue an MCP token** in Ganty: dashboard → Settings → MCP Integration → "Issue new token".
2. **Configure Claude Desktop** (`claude_desktop_config.json`) or **Claude Code** (`.mcp.json` at project root):
   ```json
   {
     "mcpServers": {
       "ganty": {
         "url": "https://ganty.app/api/mcp",
         "headers": { "Authorization": "Bearer YOUR_TOKEN" }
       }
     }
   }
   ```
3. **Restart Claude** and try a prompt:
   - `"List my Ganty projects"`
   - `"What's the critical path for the e-commerce rebuild project?"`
   - `"If I push the payments task by +3 days, when does release ship?"`

Full setup walk-through: <https://ganty.app/guide/mcp-integration>

## Tools

### Read tools (visible to read-only and write tokens)

| Tool | Description |
|---|---|
| `list_workspaces` | Workspaces the token's user belongs to |
| `list_projects` | Projects in a workspace |
| `list_tasks` | Tasks in a project (filter by name/status/assignee) |
| `get_task` | Single task with assignees + dependencies |
| `list_milestones` | Milestones in a project |
| `get_critical_path` | **Server-side CPM with slack and total duration** |

### Write tools (write-scope tokens only)

| Tool | Description |
|---|---|
| `create_project`, `delete_project` | Project lifecycle |
| `create_task`, `update_task`, `delete_task` | Task lifecycle |
| `set_task_progress` | Update task progress (auto-sets status) |
| `add_dependency` | Finish-to-Start dependency between two tasks |
| `create_milestone` | Add a milestone |
| `reschedule_and_propagate` | **Shift a task, cascade through dependencies, dry-run or commit** |

`reschedule_and_propagate` is classified as a write tool even in `dry_run` mode (it shares the same compute path as commit and we don't want read-only tokens running unlimited simulations).

## v1 scope and limitations

We prefer explicit non-support over silent wrong answers. Every response from the calculation tools includes a `limitations` array.

- **Dependency types**: Finish-to-Start only (Ganty's schema doesn't model SS/FF/SF)
- **Calendar**: calendar days by default; `business_days: true` skips Sat/Sun only (no holiday table)
- **No resource calendars** in v1
- **Multi-period tasks (extraSegments)** are ignored in v1; calculations use only the main start/end of each task
- **Pinned tasks**: `progress=100` automatically pinned; supply `pinned_task_ids` for additional pins
- **Push-only cascade**: successors are pushed back when required, never pulled forward

## Architecture

- **Transport**: Streamable HTTP (JSON-RPC 2.0 over HTTP POST) — `https://ganty.app/api/mcp`
- **Auth**: Bearer tokens (`gnty_...`) issued from the Ganty dashboard. Tokens are scoped `read` or `write`.
- **Rate limits**: 120 req/min for read, 30 req/min for write (per token)
- **Audit logging**: every write call is logged with action, actor, IP, and arg summary
- **Atomicity**: `reschedule_and_propagate` `commit` mode is wrapped in a Prisma transaction; conflicts abort without writes

The calculation logic lives in pure TypeScript functions backed by 28 golden tests asserting concrete dates and day counts (linear chain shifts, diamond merges, cycle detection within 2 seconds, progress-aware remaining duration, hand-calculated CPM with slack, business-day mode, lag-day handling, push-only cascade).

## Links

- **Marketing page**: <https://ganty.app/integrations/mcp>
- **Setup guide**: <https://ganty.app/guide/mcp-integration>
- **Claude × Gantt guide**: <https://ganty.app/blog/claude-gantt-chart>
- **Design article**: <https://ganty.app/blog/mcp-server-side-calculation-engine>
- **Tools by example**: <https://ganty.app/blog/mcp-gantt-chart-5-examples>
- **Troubleshooting**: <https://ganty.app/blog/mcp-setup-troubleshooting>
- **Sign up (free 5-member plan)**: <https://ganty.app/signup>

## License

MIT for this documentation repository.
The Ganty server itself is a hosted SaaS; the MCP endpoint is a feature of every Ganty workspace.
