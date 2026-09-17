# Codex Skill: Event-Driving Sleep

A lightweight Codex skill for session-bound waiting.

- Prefers one event-driven blocking wait when a process, log, or condition can be watched.
- Falls back to bounded timed sleep when no watcher is available.
- Keeps resume, interruption, and durable-scheduling limitations explicit.

## Install

Copy this folder into `~/.codex/skills/sleep-and-resume/`.
