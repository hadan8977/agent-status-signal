# Agent Status Signal (draft 0.1)

A small format for telling *ambient* surfaces — a notch app, a menu-bar pet, a phone lock screen, a watch, a car display, a desk device — what an agent is doing right now, and whether it needs a human.

Status: early draft for discussion. Nothing here is implemented yet.

## 1. Why

More and more people run several coding agents at once, and there is a growing set of small apps whose whole job is to show their state: [coucou](https://github.com/Louis-CFM/coucou), [Codogotchi](https://codogotchi.app/), various notch and desk-pet projects. Each of them reverse-engineers the same thing from different sources:

- Claude Code-style lifecycle hooks (`UserPromptSubmit`, `PreToolUse`, `Notification`, `Stop`, ...).
- Tool-specific formats, e.g. coucou's hook payload with a `coucou_agent` field sent over a Unix socket ([docs/AGENTS.md](https://github.com/Louis-CFM/coucou/blob/main/docs/AGENTS.md)).
- Orchestrator APIs, e.g. Paseo's daemon WebSocket (`agent_update`, `turn_completed`, ...) and its plugin `PluginAgentSnapshot`.

The existing protocols each cover part of this, but none is aimed at "many agents → one glanceable surface":

- **ACP** is one client talking to one agent session. Its tracker has run into the same gaps from the client side: a turn-complete signal ([#554](https://github.com/agentclientprotocol/agent-client-protocol/issues/554)), how to report a turn that fails after it was accepted ([#2285](https://github.com/agentclientprotocol/agent-client-protocol/issues/2285); both closed now, but worth reading), permission prompts losing tool-call context ([#1979](https://github.com/agentclientprotocol/agent-client-protocol/issues/1979)), and agent-initiated sub-sessions ([#2444](https://github.com/agentclientprotocol/agent-client-protocol/issues/2444)).
- **A2A** has task states (`working`, `input-required`, `auth-required`, `completed`, `failed`, `rejected`, ...) but they describe agent-to-agent tasks, not what a person should be told.
- **OpenTelemetry GenAI** conventions are for after-the-fact observability. Even there, "the agent went silent" has no good representation ([semantic-conventions#4187](https://github.com/open-telemetry/semantic-conventions/issues/4187)).

The result: every display app writes N adapters, every agent host is asked for a different hook, and none of them can tell "stuck" from "thinking hard".

The goal here is deliberately narrow: a status vocabulary plus one event shape, cheap enough that a host can emit it in an afternoon and a display can consume it without knowing which agent it is.

## 2. States

| State | Meaning | Needs a human? |
|---|---|---|
| `idle` | Session exists, nothing running. | no |
| `working` | Actively doing something (thinking, reading, editing, running tools). | no |
| `needs-approval` | Blocked on a yes/no permission for a specific action. | yes |
| `needs-input` | Blocked on a free-form answer or choice. | yes |
| `stuck` | Supposed to be working, but no meaningful progress (see §6.2). | maybe |
| `done` | The last turn finished successfully. | optional (review) |
| `failed` | The last turn ended in an error, was cancelled, or was refused. | usually |

`working` MAY carry an `activity` hint for richer visuals: `thinking`, `reading`, `editing`, `running`, `testing`, `browsing`, `waiting-on-subagent`. Displays MUST treat unknown activities as plain `working`.

### Transitions

```
idle ──prompt──▶ working
working ──permission request──▶ needs-approval ──answered──▶ working
working ──question──▶ needs-input ──answered──▶ working
working ──no progress for T──▶ stuck ──progress──▶ working
working ──turn ends ok──▶ done
working | stuck ──error / cancel / refusal──▶ failed
done | failed ──acknowledged or timeout──▶ idle
any ──session closed──▶ (signal with "closed": true, then forget)
```

Rules:

1. A producer MUST send the full current state in every event (no deltas). A consumer that missed messages can always recover from the next one.
2. `stuck` MAY be computed by either the producer or the consumer. If the producer sends it, it SHOULD include `stuckReason`.
3. `done` and `failed` are sticky until acknowledged or until the next prompt, so a person who looks away does not miss them.
4. A `needs-approval` event MUST include the action being approved (see §3), so an approval surface never shows a bare "Allow?" — the problem described in ACP #1979.

## 3. Message

```json
{
  "v": "0.1",
  "type": "agent.status",
  "source": "paseo",
  "agentId": "a1b2",
  "sessionId": "s-42",
  "parentId": null,
  "name": "Clover",
  "workspace": "~/code/shop-api",
  "state": "needs-approval",
  "activity": null,
  "summary": "Wants to run the migration",
  "since": "2026-10-27T08:01:12Z",
  "lastProgressAt": "2026-10-27T08:01:10Z",
  "progress": { "done": 3, "total": 5 },
  "request": {
    "id": "perm-9",
    "kind": "approval",
    "title": "Run `pnpm db:migrate`",
    "detail": "Touches the local dev database only",
    "options": ["allow", "deny"]
  },
  "result": null,
  "urgency": "high",
  "closed": false
}
```

- `parentId` links sub-agents, so a display can roll up "Clover · 3 agents".
- `summary` is one human sentence, ≤ 80 chars. Producers SHOULD write it for a person, not a log.
- `result` is set on `done`/`failed`: `{ "outcome": "ok" | "error" | "cancelled" | "refused", "text": "Opened PR #12, tests pass" }`.
- `urgency` is a producer hint (`low` | `normal` | `high`). Consumers decide what to do with it (§6).

### JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/hadan8977/agent-status-signal/blob/main/schema/status-0.1.json",
  "type": "object",
  "required": ["v", "type", "source", "agentId", "state", "since"],
  "properties": {
    "v": { "const": "0.1" },
    "type": { "const": "agent.status" },
    "source": { "type": "string", "pattern": "^[a-z0-9-]{1,32}$" },
    "agentId": { "type": "string", "minLength": 1 },
    "sessionId": { "type": ["string", "null"] },
    "parentId": { "type": ["string", "null"] },
    "name": { "type": ["string", "null"], "maxLength": 64 },
    "workspace": { "type": ["string", "null"] },
    "state": { "enum": ["idle", "working", "needs-approval", "needs-input", "stuck", "done", "failed"] },
    "activity": { "type": ["string", "null"] },
    "summary": { "type": ["string", "null"], "maxLength": 80 },
    "since": { "type": "string", "format": "date-time" },
    "lastProgressAt": { "type": ["string", "null"], "format": "date-time" },
    "stuckReason": { "enum": ["no-output", "repeating", "retrying", "waiting-external", null] },
    "progress": {
      "type": ["object", "null"],
      "properties": { "done": { "type": "integer", "minimum": 0 }, "total": { "type": "integer", "minimum": 1 } }
    },
    "request": {
      "type": ["object", "null"],
      "required": ["id", "kind", "title"],
      "properties": {
        "id": { "type": "string" },
        "kind": { "enum": ["approval", "question"] },
        "title": { "type": "string" },
        "detail": { "type": "string" },
        "options": { "type": "array", "items": { "type": "string" } }
      }
    },
    "result": {
      "type": ["object", "null"],
      "required": ["outcome"],
      "properties": {
        "outcome": { "enum": ["ok", "error", "cancelled", "refused"] },
        "text": { "type": "string" }
      }
    },
    "urgency": { "enum": ["low", "normal", "high", null] },
    "closed": { "type": "boolean" }
  },
  "allOf": [
    { "if": { "properties": { "state": { "enum": ["needs-approval", "needs-input"] } } },
      "then": { "required": ["request"], "properties": { "request": { "type": "object" } } } }
  ],
  "additionalProperties": true
}
```

Unknown fields are allowed; consumers MUST ignore what they don't understand.

### Replying (optional)

A display that can act sends back:

```json
{ "v": "0.1", "type": "agent.respond", "agentId": "a1b2", "requestId": "perm-9", "choice": "allow" }
```

Producers MUST check `requestId` against the request that is actually pending and drop anything else. Approving the wrong request is the worst failure this format can cause.

## 4. Transport

Keep it boring; pick whichever the host already has.

- **Local socket** (default): newline-delimited JSON over a Unix domain socket at `$XDG_RUNTIME_DIR/agent-status.sock` (macOS: `~/Library/Application Support/agent-status/agent-status.sock`), or a named pipe `\\.\pipe\agent-status` on Windows. The display listens; producers connect, write, and never block if no one is listening — same rule coucou uses for hooks.
- **WebSocket**: for hosts that already run a daemon (Paseo, OpenMausBot). A consumer connects to `ws://127.0.0.1:<port>/agent-status`, receives a snapshot (one `agent.status` per live agent), then live events.
- **Remote** (phone, watch, car, desk device): out of scope for 0.1. A local bridge relays events over whatever pairing the device already has. Never expose the socket on a network interface by default.

Auth: local socket relies on file permissions; WebSocket SHOULD require a token in a header or the first message.

## 5. Mappings

These were checked against each project's own schema or docs on 2026-10-27. Links point to the source used. If something here drifts, the upstream source wins.

### ACP

Source: [`schema/v1/schema.json`](https://github.com/agentclientprotocol/agent-client-protocol/blob/main/schema/v1/schema.json).

| ACP | → Signal |
|---|---|
| client sends `session/prompt` | `working` |
| `session/update` with `agent_message_chunk`, `agent_thought_chunk`, `tool_call`, `tool_call_update`, `plan` | `working`; `activity` from the tool call's `kind` (`read`/`search` → reading, `edit`/`delete`/`move` → editing, `execute` → running, `fetch` → browsing, `think` → thinking); `progress` from `plan` entries |
| `session/request_permission` (agent → client request) | `needs-approval`; `request` built from the attached `toolCall` and `options` |
| `session/prompt` result `stopReason: end_turn` | `done` |
| `stopReason: max_tokens` or `max_turn_requests` | `failed`, `outcome: error` (the turn stopped early, not by choice) |
| `stopReason: refusal` | `failed`, `outcome: refused` |
| `stopReason: cancelled` (after `session/cancel`) | `failed`, `outcome: cancelled` |
| JSON-RPC error response to `session/prompt` | `failed`, `outcome: error` |
| `session/close` | `closed: true` |

ACP has no stop reason for "failed after the prompt was accepted"; that is what [#2285](https://github.com/agentclientprotocol/agent-client-protocol/issues/2285) asked about, so for now a JSON-RPC error is the only signal.

### A2A

Source: [`specification/a2a.proto`](https://github.com/a2aproject/A2A/blob/main/specification/a2a.proto), `TaskState`. The proto uses `TASK_STATE_*` names; older JSON versions of the spec use kebab-case (`input-required`, ...). Both are listed.

| A2A | → Signal |
|---|---|
| `TASK_STATE_SUBMITTED` / `submitted`, `TASK_STATE_WORKING` / `working` | `working` |
| `TASK_STATE_INPUT_REQUIRED` / `input-required` | `needs-input` |
| `TASK_STATE_AUTH_REQUIRED` / `auth-required` | `needs-input`, `request.kind: question`. This is about credentials (sign in, grant access), not approving one action, so it is not `needs-approval`. |
| `TASK_STATE_COMPLETED` / `completed` | `done` |
| `TASK_STATE_FAILED` / `failed` | `failed`, `outcome: error` |
| `TASK_STATE_CANCELED` / `canceled` | `failed`, `outcome: cancelled` |
| `TASK_STATE_REJECTED` / `rejected` | `failed`, `outcome: refused` |
| `TASK_STATE_UNSPECIFIED` / `unknown` | keep the last known state |

### Claude Code hooks

Source: [Hooks reference](https://code.claude.com/docs/en/hooks).

| Hook | → Signal |
|---|---|
| `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `SubagentStart` | `working` (`activity` from `tool_name`) |
| `PermissionRequest` | `needs-approval`, `request` from `tool_name` + `tool_input`. Fires right away. |
| `Notification` with `notification_type: permission_prompt` | `needs-approval`. Only fires after the prompt has waited about six seconds and the user seems away, so prefer `PermissionRequest`. |
| `Elicitation`, or `Notification` with `elicitation_dialog` / `elicitation_url_dialog` | `needs-input` |
| `Stop` | `done` |
| `StopFailure` (matcher = error type, e.g. `rate_limit`, `server_error`) | `failed`, `outcome: error` |
| `Notification` with `idle_prompt` | no change: Claude finished about 60 s ago and nobody typed. A display can use it to move `done` → `idle` or to nudge. It is **not** `needs-input`. |
| `SubagentStop` | child `done`, parent unchanged |
| `SessionEnd` | `closed: true` |

### Paseo

Source: [`packages/plugin/src/contracts.ts`](https://github.com/getpaseo/paseo/blob/main/packages/plugin/src/contracts.ts) (`PluginAgentSnapshot`) and [`public-docs/sdk/events.md`](https://github.com/getpaseo/paseo/blob/main/public-docs/sdk/events.md).

| Paseo | → Signal |
|---|---|
| `status: initializing` or `running` | `working` |
| `requiresAttention: true`, `attentionReason: "permission"` | `needs-approval` |
| `turn_completed` event, or `attentionReason: "finished"` | `done` |
| `turn_failed` event, or `status: error` / `attentionReason: "error"` | `failed`, `outcome: error` |
| `turn_canceled` event | `failed`, `outcome: cancelled` |
| `status: idle` with no attention flag | `idle` |
| `status: closed`, or `agent_update` with `kind: "remove"` | `closed: true` |
| `parentAgentId`, `cwd`, `title`, `lastActivityAt` | `parentId`, `workspace`, `name`, `lastProgressAt` |

Paseo's docs say not to infer turn completion from an `agent_update` that goes to `idle`; use the `turn_*` events. An earlier draft of this table did exactly that, which was wrong.

## 6. Interruption guidance (non-normative)

The point of a status surface is to interrupt *less*, not more. Suggestions from in-car UI, where a bad interruption costs a lot:

### 6.1 Roll-up
Show one aggregate by default: "3 working · 1 needs you". Only the states that need a human get their own line. Group by `parentId`, then `workspace`.

### 6.2 Stuck detection
Consider an agent `stuck` when it has been `working` and:
- no progress event for T (start with 3 minutes, tune per activity; `running`/`testing` can legitimately be quiet longer), or
- the same tool call with the same input repeated 3+ times, or
- 3+ consecutive retries of a failing step.
Escalate `stuck` gently: change the pet's look first, notify only if it lasts 2×T.

### 6.3 Interruption budget
- `needs-approval` / `needs-input`: notify once, then stay visible; don't re-notify more than every 10 minutes.
- `failed`: notify once.
- `done`: notify only if the person asked to be told, or the run was long (> 10 min); otherwise just show it.
- Batch everything else and deliver when the person comes back (unlock, returns to the desk, car parked).
- Respect context: focus mode, a meeting in the calendar, or driving should push everything except `high` urgency into the batch.

### 6.4 Say what happened
`done` and `failed` should carry `result.text`. "Finished" is not useful; "Opened PR #12, tests pass" or "Gave up: can't reach the staging DB" is.

## 7. Open questions

1. Should this live inside ACP (e.g. a client-side subscription capability) or as a separate tiny spec that maps onto ACP?
2. Who computes `stuck` — producer, consumer, or both with a precedence rule?
3. Is `done → idle` on acknowledgement a producer concern or purely a display concern?
4. Do we need a per-agent identity/avatar field, or leave that to displays?
5. How should multiple displays coordinate, so an approval answered on the phone clears the notch too?
6. Minimum security for `agent.respond`: is a local token enough, or should approvals require a device-level confirmation (Face ID, etc.)?
7. Naming: `needs-approval` vs ACP's "permission" vs A2A's `auth-required`.

Feedback welcome. This is written to be thrown away if an existing protocol is the better home.

*Drafted with help from an AI assistant; reviewed and edited by hand.*
