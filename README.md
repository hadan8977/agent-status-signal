# agent-status-signal

A small, draft format for telling glanceable surfaces (a notch app, a menu-bar pet, a lock screen, a watch, a car display) what a coding agent is doing and whether it needs you.

Seven states, one JSON event, a local socket by default, and mapping tables from ACP, A2A, Claude Code hooks and Paseo.

- Spec: [SPEC.md](SPEC.md)
- JSON Schema: [schema/status-0.1.json](schema/status-0.1.json)
- 中文摘要: [SUMMARY.zh.md](SUMMARY.zh.md)

Status: 0.1 draft, nothing implemented yet. Issues and objections are welcome, including "this belongs in protocol X".

MIT licensed.
