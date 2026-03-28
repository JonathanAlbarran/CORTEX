# CORTEX

Multi-agent coordination via filesystem. Drop a `.cortex/` directory into your project and agents can find each other, claim tasks, and exchange signals by reading and writing JSON files.

## What is this?

CORTEX is a protocol for making AI agents work together through a shared filesystem. The full spec is one file (`PROTOCOL.md`). There's nothing to install.

It came out of a specific setup: a sandboxed, ephemeral agent (Claude in Cowork) collaborating with a persistent local agent (built on OpenClaw), with a human operator who has final say. The agents couldn't share memory or call each other's APIs, but they could both see the same project folder. CORTEX is the set of rules that makes that work.

## How it works

Each agent gets its own directory under `.cortex/agents/`. An agent only writes to its own directory. That's the core rule, and it's what makes everything else possible: if no two agents ever write to the same file, you don't need locks.

There is no "current state" file. You figure out the current state by reading all agents' event logs and merging them (same idea as a CRDT). Events are append-only JSON files. Tasks are write-once specs. Ownership, status, shared memory — all derived at read time from the logs.

The protocol doesn't care what's behind an agent. OpenAI, Anthropic, a local model, a bash script. If it can read and write JSON files, it can participate.

A human operator always has the final word. The compromise containment protocol (§18 in the spec) describes how agents monitor each other and how the human intervenes when something goes wrong.

## Quick start

### 1. Initialize the hub

Copy `.cortex/` into your project root.

```bash
cp -r .cortex/ /path/to/your/project/.cortex/
```

### 2. Bootstrap an agent

Give your agent [BOOTSTRAP.md](BOOTSTRAP.md) and point it at the `.cortex/` directory. The agent reads the protocol and registers itself.

### 3. Deploy a satellite to another project

Copy the satellite template into the other project, then let the agent there read `SATELLITE.md`. It covers setup, including how to find the hub agent and send the first signal.

```bash
cp -r .cortex/satellite-template/ /path/to/other/project/.cortex/
```

### Manual setup

If you'd rather configure things by hand, see the [protocol spec](.cortex/PROTOCOL.md) §12 ("Adding a New Agent") for the full procedure.

## Architecture

```
.cortex/
├── PROTOCOL.md              ← The full specification
├── satellites.json          ← Registry of satellite projects
├── agents/
│   ├── .template/           ← Manifest template for new agents
│   ├── {agent-id}/
│   │   ├── manifest.json    ← Agent self-description
│   │   ├── log/             ← Append-only event stream
│   │   └── inbox/           ← Signals from other agents
│   └── ...
├── tasks/                   ← Write-once task specifications
├── views/                   ← Cached derived state
├── staging/                 ← Shared config staging area
└── satellite-template/      ← Bootstrap template for new projects
```

### Hub and satellites

One project folder runs the full protocol (the "hub"). Other projects can't write to the hub directly — different sandboxes — so they get **satellites**: a minimal `.cortex/` with just an outbox and inbox. A persistent agent at the hub checks satellite outboxes on a timer and writes responses back.

Concrete example: an OpenClaw agent on a home server owns the hub. Claude sessions in Cowork are sandboxed per project. Each project gets a satellite. The OpenClaw agent sweeps them all every few minutes.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Project A      │     │  Hub            │     │  Project B      │
│  Claude session │     │  OpenClaw agent │     │  Claude session │
│  (satellite)    │     │  (full CORTEX)  │     │  (satellite)    │
│  outbox/ ────────────→│  agents/        │←─────── outbox/      │
│  inbox/  ←────────────│  tasks/         │────────→ inbox/       │
└─────────────────┘     │  views/         │     └─────────────────┘
                        └─────────────────┘
                     OpenClaw sweeps satellites
                           on heartbeat
```

## Glossary

If you want the full details, read `PROTOCOL.md`. This is just enough to orient you.

**Iron rule** — an agent only writes to `.cortex/agents/{its-own-id}/`. Never to another agent's namespace. This is the entire concurrency model.

**Events** — immutable JSON files in `agents/{id}/log/`, numbered sequentially (zero-padded to 8 digits). Once written, never modified.

**Tasks** — write-once JSON specs in `tasks/`. Who owns a task is determined by scanning all agent logs for claim/release events and applying a deterministic tiebreak. The task file itself never changes.

**Signals** — messages dropped into `agents/{recipient}/inbox/`. Filenames include sender ID, timestamp, and a random suffix, so two agents writing to the same inbox at the same time produce different files.

**Views** — cached derived state in `views/`. Any agent can regenerate them. They exist to save time on full log scans; they are not authoritative.

**Satellites** — lightweight `.cortex/` directories in other projects (just an outbox and inbox). Described above.

## Scope

CORTEX is just the coordination layer. You provide the agents. The protocol tells them how to share a filesystem without stepping on each other. There is nothing to deploy beyond copying the `.cortex/` directory and pointing your agents at `PROTOCOL.md`.

## Security

§18 of the spec covers compromise detection. The short version: agents watch each other. If one violates the iron rule or acts outside its declared capabilities, the others quarantine it and alert the human. Only the human can lift the quarantine. See the spec for the full procedure.

## Repository contents

```
README.md                            ← This file
BOOTSTRAP.md                         ← Give this to an agent to self-register
LICENSE                              ← MIT license
.gitignore                           ← Prevents committing live state
.cortex/                             ← Drop-in protocol directory
  PROTOCOL.md                        ← Full specification
  satellites.json                    ← Satellite registry
  agents/.template/manifest.json     ← Agent manifest template
  satellite-template/                ← Bootstrap for satellite nodes
  staging/                           ← Config staging area
  tasks/                             ← Task storage
  views/                             ← Derived state cache
examples/
  manifests/
    agent-anthropic.json             ← Example: Anthropic agent manifest
    agent-openai.json                ← Example: OpenAI agent manifest
    agent-openclaw.json              ← Example: OpenClaw agent manifest
    agent-human.json                 ← Example: Human operator manifest
  task.json                          ← Example: Task specification
  signal.json                        ← Example: Signal between agents
  events/
    00000001.json                    ← Example: agent.online event
    00000002.json                    ← Example: task.claim event
```

## License

MIT. See [LICENSE](LICENSE).
