# Codex Skill: Event-Driving Sleep

A lightweight Codex skill for session-bound waiting.

- Prefers one event-driven blocking wait when a process, log, or condition can be watched.
- Falls back to bounded timed sleep when no watcher is available.
- Keeps resume, interruption, and durable-scheduling limitations explicit.
- Records the concrete Codex exec-session mechanics: the absent `clock.sleep`, the 30s yield
  and 5-300s blocking-read limits, how to cancel a live wait, and why a watcher's outcome must
  be read from its own status line rather than the session exit code.

## Install

Copy this folder into `~/.codex/skills/sleep-and-resume/`.
