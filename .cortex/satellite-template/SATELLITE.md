# CORTEX Satellite Protocol

## What this is

This is a lightweight CORTEX node. It connects this project to a CORTEX hub where agents coordinate. Through it, a sandboxed agent can communicate with a persistent agent, request work, share knowledge, and reach the full multi-agent system.

The hub has the full protocol, agent logs, task history, and views. This satellite has just enough to send and receive.

## Architecture

```
┌──────────────────────────┐      ┌──────────────────────────────────┐
│  THIS PROJECT (satellite) │      │  HUB PROJECT                     │
│                           │      │                                  │
│  .cortex/                 │      │  .cortex/                        │
│    outbox/  ───────────────────→ │    agents/                       │
│    inbox/   ←──────────────────    tasks/                          │
│    manifest.json          │      │    views/                        │
│    SATELLITE.md           │      │    satellites.json                │
└──────────────────────────┘      └──────────────────────────────────┘

Persistent agent sweeps all satellite outboxes on heartbeat.
Responses come back to the satellite's inbox.
```

## Directory structure

```
.cortex/
├── SATELLITE.md        ← This file
├── manifest.json       ← This project's identity
├── outbox/             ← Write signals here; persistent agent picks them up
├── inbox/              ← Persistent agent writes responses here; read and delete
└── log/                ← Local event log (optional)
```

## Deployment

If this satellite hasn't been set up yet:

1. Create the directories this protocol needs:
   ```
   .cortex/outbox/
   .cortex/inbox/
   .cortex/log/
   ```

2. Fill in `manifest.json`:
   - `project`: a short name for this project
   - `hub`: the filesystem path to the hub's `.cortex/` directory (ask the human operator)
   - `hub_agent`: the agent ID of the persistent agent at the hub (ask the human operator, or check the hub's `agents/` directory if you have access)
   - `registered_at`: current ISO-8601 timestamp

3. The persistent agent at the hub must register this satellite in `.cortex/satellites.json`. Send your first signal asking it to do so, or ask the human operator to relay.

## Sending a signal

Write a JSON file to `outbox/`. Set the `"to"` field to the `hub_agent` from your manifest unless you're targeting a different agent.

```json
{
  "id": "{project}-{YYYYMMDD}-{HHmmss}-{4hex}",
  "from": "your-agent-id",
  "from_project": "project-name",
  "to": "hub-agent-id",
  "ts": "ISO-8601",
  "type": "request|info|alert",
  "content": "What you need",
  "refs": [],
  "ttl_hours": 24
}
```

## Receiving responses

Check `inbox/` for JSON files. Read them, act on them, delete after processing.

## What you can do from a satellite

- Request work from agents at the hub
- Share information across the system
- Ask the persistent agent to create hub-level tasks on your behalf
- Query system state (task status, shared memory, other project state)
- Alert on issues

## What you cannot do

- Write directly to the hub's agent namespaces, tasks, or views
- Read the hub's files (different sandbox)
- Create hub-level tasks directly

Everything routes through the persistent agent as relay.
