# OpenClaw Templates

Architecture templates for building trustworthy OpenClaw AI agents. Derived from a production multi-agent deployment; audited to zero blocking findings by an independent reasoning model.

## Current templates

### [openclaw-agent-architecture.md](./openclaw-agent-architecture.md)
The complete blueprint for personal and team OpenClaw agents, in three layers:

- **Layer 1 — Core single agent**: workspace instruction files, memory system, model fallback chains, security baseline, observability, human gates
- **Layer 2 — One gateway, multiple agents**: coordinator + sub-agent pattern, channel bindings, group communication protocol
- **Layer 3 — Fleet**: multi-machine trust-domain isolation, hosted agents for other people, warm pools, metering

Includes a threat model, setup checklist, failure-mode catalog, and four appendices (workspace skeletons, artifact templates, directory layout, helper function signatures).

**Scope note:** this is an architecture template, not an installation manual. It pairs with the [OpenClaw docs](https://docs.openclaw.ai) — the docs give the current commands; this gives the architecture and discipline.

## Design principles

1. Ship first, iterate after
2. Text is memory — agents forget between sessions; files do not
3. Least privilege by default, isolation scaled to risk
4. Human gate for anything that leaves the machine
5. Enforcement lives in tools, knowledge lives in skills, determinism lives in scripts
6. Auditable procedures beat tribal knowledge
