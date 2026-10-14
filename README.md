# Codex Skill: Event-Driving Sleep

A lightweight Codex skill for session-bound waiting.

- Prefers one event-driven blocking wait when a process, log, or condition can be watched.
- Falls back to bounded timed sleep when no watcher is available.
- Keeps resume, interruption, and durable-scheduling limitations explicit.
- Selects the available sleep tool and host shell, or uses a child session with bounded reads.
- Checks runner ownership for computer-off use: persistent remote sessions and cloud tasks
  can continue without a local client while their execution host stays available.
- Records how to cancel a live wait and distinguish watcher outcomes from shell wrapper codes.

## Install

Copy this folder into `~/.codex/skills/sleep-and-resume/`.
