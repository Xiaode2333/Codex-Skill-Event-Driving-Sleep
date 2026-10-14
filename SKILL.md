---
name: sleep-and-resume
description: Wait in a live Codex or Claude Code session using event-driven blocking waits when available, with timed fallback, then check status or resume work. Use for session-bound waiting, not durable scheduling.
---

# Sleep and Resume

Apply the project's active constraints and monitoring workflow before choosing
an interval. Preserve the user's objective, keep-alive instruction, completion
condition and wake target in existing task state for long or recurring waits.
Perform any requested pre-check once and state the intended wake time.

## Backend lifetime and computer-off use

Determine where the agent runner, watcher and network connection live before
claiming that work continues while the user's computer is off. Inspect process
ancestry and session ownership without printing credentials or environment
variables. A remote command host alone does not prove the agent runs remotely.

- **Always-on remote host:** the agent itself must run on that host in a
  persistent session or service. For a CLI inside `tmux`, confirm the agent is
  a descendant of the tmux server and `destroy-unattached` is off. Losing the
  local SSH client then leaves the runner and its wait alive; resuming the same
  tmux session reconnects to the running agent. Network/authentication and the
  remote host must remain available. `tmux` does not survive host reboot.
- **Cloud task:** use the product's supported cloud task execution. Confirm
  the required project tools, cluster access and credentials exist there;
  moving the conversation alone does not transfer local access.
- **Local runner:** sleeping or turning off its host suspends or stops work,
  even when the watcher targets a remote cluster. Place the runner on an
  always-on host or use supported cloud execution for computer-off operation.

Keep the difference between client disconnection and runner termination
explicit. Test a short wait and wake on the actual backend; report that evidence
without claiming an unperformed power-off, restart or recovery test. A detached
watcher, `nohup`, submitted Slurm job, or saved task note can preserve a process
or evidence but does not independently wake a terminated agent. Preserve task
state for manual resume after interruption; durable unattended restart requires
a supported supervisor or scheduler that actually launches the agent.

## Event-driven waits

For process-, log-, or condition-based waits, prefer one event watcher over a
timed sleep when the runtime supports PTY/session reads. Start one watcher
session that exits on completion, failure, fatal output, or timeout, then use a
single blocking read on that same session. Treat the watcher timeout as the
fallback for a stuck task. If the blocking read yields before the event, resume
the same watcher session; do not create duplicate watchers or poll the target.
On wake, inspect the event once, diagnose failure or continue the workflow. A
watcher only improves responsiveness; it does not make waiting durable.

## Timed waits

For timed waits, use the runtime's available interruptible sleep tool (for
example `clock.sleep`) first. Inspect its duration limits and interruption
behavior; obey higher-priority wait and communication limits even when the tool
accepts a longer duration. Prefer a single sleep for the chosen interval when
permitted. Otherwise retain one wake target across bounded chunks, track
elapsed time from tool results and avoid clock/job polling on intermediate
wakes. Keep required idle updates brief.

When no dedicated sleep tool exists, use a supported asynchronous timer with
the runtime's actual yield limits. Resume the same timer cell after an early
yield; do not create duplicate timers. If neither mechanism exists, preserve
task state and report that in-session waiting is technically unavailable.

## Concrete mechanics in Codex exec sessions

Inspect the exposed tools rather than assuming every Codex backend has the
same capabilities. If `clock.sleep` is available, use it for timed waits within
the active runtime's limits. For an event watcher, or a timed fallback when no
sleep tool exists, run a child process through `exec_command` and read the same
session with `write_stdin`. Use the host's actual shell: Bash/POSIX `sleep` on
Linux/macOS, or PowerShell `Start-Sleep` on Windows. Print explicit start and
outcome lines. Use a PTY when cancellation needs Ctrl-C.

- Start the wait with a command call that yields early and returns a session id
  while the work is still running. Its own yield caps at 30s, so a long wait
  necessarily spans several reads. Never promise that a long wait costs zero
  turns.
- Block on that same session id with an empty write: one empty read waits 5s to
  300s. A read returns the instant the child exits, so a 20s timer is reported
  at 20s, not at the read ceiling.
- Resume the same session id after an early yield. Reissuing the wait command
  creates a second timer or watcher on the same target.
- To cancel a live wait, send a break to the session. It kills the child
  immediately, the session id becomes invalid, and no completion line is
  printed. Verify cancellation while the wait is genuinely live; a break sent
  after the child already exited proves nothing.
- Run concurrent producers, observers, or helpers as separate sessions so the
  waiter and the actor progress independently.

### Keep long waits dormant

When the orchestration runtime can keep an exec cell alive and later wait on
that cell, keep bounded session reads inside one cell instead of returning to
the model after every 300-second read:

1. In one exec cell, loop empty reads on the same child-session id, using at
   most the supported per-read ceiling. Accumulate output and stop only when
   the child exits or the cell's own bounded deadline is reached.
2. If the outer exec call yields a cell id while that loop is still running,
   use one interruptible cell-level wait for the remaining wake horizon. This
   preserves user steering while the model remains dormant between reads.
3. Do not emit heartbeat notifications, repeated commentary, or unchanged
   status messages during a healthy silent wait. Return control only for the
   watcher's explicit event/timeout/fatal status, user interruption, or a real
   runtime limit that requires continuation.

If no steerable cell-level wait exists, fall back to resuming the same child
session directly and keep any runtime-required idle updates minimal. Never
replace a silent watcher with repeated scheduler or target polling.

## Concrete mechanics in Claude Code sessions

Claude Code re-invokes the model when tracked background work finishes, so do
not poll with foreground sleeps (they are blocked) or short wake loops.

- Event watcher: start the watcher with `Bash` and `run_in_background: true`;
  the model is woken when it exits. Alternatively use the `Monitor` tool with
  an until-loop on the condition. Do not read its output file while it runs.
- External state the harness cannot track (Slurm queue, remote job): run one
  background watcher script that exits on the event or its own timeout, so its
  exit is the wake signal.
- Self-paced `/loop` mode: use `ScheduleWakeup` with a delay matched to how
  fast the target changes; as the fallback heartbeat behind a watcher, use
  1200s or more. Pass the same loop prompt back on each wake.
- `CronCreate` recurs only while the session is alive; it is not durable.
- Cancel a live background watcher with `TaskStop`.

Measured (2026-10-14, Claude Code 2.1.300, MD_LAMMPS Slurm-hosted backend):

- No model requests occur between ending the turn and the watcher's exit; each
  wake costs one request that re-reads the cached context.
- In `--input-format stream-json` mode, closing stdin (what the backend does
  when the VS Code client disconnects with no turn pending) does not stop a
  running background watcher. The CLI wakes on its exit, runs the follow-up
  turn, may start the next watcher and wake again, and exits only once no
  background task remains. Chained waits therefore continue with the client
  gone. Always keep one background watcher running across a wait; a
  `ScheduleWakeup`/`CronCreate`-only wait has nothing holding a detached
  process open and is not verified.
- Detached turns write output to the backend journal and the session
  transcript; reopen the conversation in the extension to see it. The runner
  stops with its Slurm allocation (check `squeue -o %L` for time left).

## Detecting the watcher's outcome

Have the watcher print one explicit status line at exit, for example
`status=EVENT|TIMEOUT|FATAL elapsed=...`, and judge the outcome by that line or
the captured child exit code (`$?` immediately after the child in Bash,
`$LASTEXITCODE` in PowerShell). Propagate that code explicitly when wrapping it.

Prefer the explicit outcome over a shell wrapper's session-level exit code.
In PowerShell, a shell host can report 1 whenever its last command's `$?` was
false, so a watcher that deliberately exits 3 on timeout or 4 on fatal output
can still surface as session code 1. Measured case: the script's own
`$LASTEXITCODE` was 3 while the session reported 1. This behavior is not a
universal rule for Bash or every backend.

Read the three outcomes differently:

- `EVENT` / `COMPLETE`: the condition fired; proceed with the workflow.
- `TIMEOUT`: the fallback fired as designed. Perform one status check, then
  choose the next interval; do not assume the target failed.
- `FATAL`: stop waiting and diagnose before scheduling another wait.

## Resuming and interruption

On interruption, apply user steering before resuming or canceling the remaining
wait. On expiry, perform one requested status check, do newly available useful
work, and select the next interval using the project's workflow. After context
restoration, resume the remaining wait or check once if overdue.

For explicit open-ended keep-alive requests, continue until the objective is
complete, the user stops it, or continuation is technically impossible. Do not
ask for an artificial deadline or end merely because there is no useful work.
Honor a supplied deadline or bounded monitoring horizon when one exists.

Do not promise zero model turns when runtime limits force continuations. Do not
claim waiting survives runner termination or host shutdown. Client
disconnection can leave a verified remote persistent runner or supported cloud
task active, as described above. A durable scheduling request needs a supported
scheduler and runner; a skill cannot independently reactivate a closed chat.
