---
name: sleep-and-resume
description: Wait in a live session using event-driven blocking waits when available, with timed fallback, then check status or resume work. Use for session-bound waiting, not durable scheduling.
---

# Sleep and Resume

Apply the project's active constraints and monitoring workflow before choosing
an interval. Preserve the user's objective, keep-alive instruction, completion
condition and wake target in existing task state for long or recurring waits.
Perform any requested pre-check once and state the intended wake time.

## Event-driven waits

For process-, log-, or condition-based waits, prefer one event watcher over a
timed sleep when the runtime supports PTY/session reads. Start one watcher
session that exits on completion, failure, fatal output, or timeout, then use a
single blocking read on that same session. Treat the watcher timeout as the
fallback for a stuck task. If the blocking read yields before the event, resume
the same watcher session; do not create duplicate watchers or poll the target.
On wake, inspect the event once, diagnose failure or continue the workflow. A
watcher only improves responsiveness; it does not make waiting durable.

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

On interruption, apply user steering before resuming or canceling the remaining
wait. On expiry, perform one requested status check, do newly available useful
work, and select the next interval using the project's workflow. After context
restoration, resume the remaining wait or check once if overdue.

For explicit open-ended keep-alive requests, continue until the objective is
complete, the user stops it, or continuation is technically impossible. Do not
ask for an artificial deadline or end merely because there is no useful work.
Honor a supplied deadline or bounded monitoring horizon when one exists.

Do not promise zero model turns when runtime limits force continuations. Do not
claim waiting survives session closure, reload, disconnection or host
termination. A durable scheduling request needs a supported scheduler and
runner; a skill cannot independently reactivate a closed chat.
