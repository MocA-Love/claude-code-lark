---
name: takeover
description: Take over the Lark WebSocket connection from another Claude Code session. Use when Lark messages are going to a different session, or when you want to switch Lark to the current session.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
  - Bash(ps *)
---

# /lark:takeover — Take Over Lark Connection

Takes the Lark WebSocket connection from another session and assigns it to
this one. The other session will detect the takeover and disconnect within
a few seconds.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — take over

1. Read `~/.claude/channels/lark/ws.lock` and show current owner (PID,
   started time) if it exists.

2. Find this session's MCP server PID. Run:
   ```bash
   ps -o pid=,ppid=,command= -ax | grep "bun.*server.ts" | grep -v grep
   ```
   Find the `bun server.ts` process whose parent chain includes this
   Claude Code process. The parent PID (ppid) of `bun server.ts` is a
   `bun run` process, whose parent is the Claude Code process.

3. Write the takeover signal file. Get the PID of the `bun run` process
   that is the direct parent of our `bun server.ts`:
   ```bash
   mkdir -p ~/.claude/channels/lark
   ```
   Write `~/.claude/channels/lark/takeover` with the ppid of our
   `bun server.ts` process (the `bun run` wrapper PID).

   The server.ts poll loop checks if this matches `process.ppid` and
   takes over the lock if it does.

4. Confirm: "Takeover requested. The Lark connection will switch to this
   session within a few seconds."

### `status` — show current state

1. Read `~/.claude/channels/lark/ws.lock`.
2. Show: owner PID, started time, whether the process is alive.
3. List all running `bun server.ts` processes.

---

## Implementation notes

- The takeover signal file is ephemeral — the target server.ts deletes it
  after reading.
- The previous lock holder detects the signal, releases its lock, and
  closes its WSClient.
- The target server.ts acquires the lock and starts its WSClient.
- The whole process takes ~3 seconds (one poll interval).
