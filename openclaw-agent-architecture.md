# TEMPLATE — OpenClaw Agent Architecture

> **Purpose:** A shareable, public-safe architecture template for building personal and team OpenClaw agents. Derived from a production multi-agent deployment and hardened engineering practices.
>
> **Status:** Public architecture pattern template, derived from a production multi-agent deployment. Public-safe: no personal names, phone numbers, ports tied to live systems, private project names, internal hostnames, or credentials appear in this file. Role names are generic. OpenClaw (an open-source agent runtime) is referenced for concreteness; verify its current links and CLI against the live project before relying on them.
>
> ### How to Treat This Document
>
> **Version note and scope:** this is an architecture template, not an installation manual. CLI command forms and configuration field names change between releases, so the JSON shapes here are illustrative of structure and scale — do not copy them verbatim; pair this document with the official docs at https://docs.openclaw.ai for the current schema and commands. What this template supplies that the docs do not: the layered architecture, the security baseline, the operational discipline, and the failure modes learned in production. What it deliberately does NOT supply: a runnable reference implementation. The pseudocode, schemas, and skeletons here are specifications, not a starter repo — expect to write roughly 900–1300 lines of careful glue (locking, ledger helpers, two-phase gate with crash reconciliation, approval handler, privileged viewer, daily digest with all its checks, promotion/aggregation, anomaly detection, backup smoke test, supervisor integration) yourself, using the shapes and patterns in the appendices as specifications. That work is the point: an agent you built to this spec is an agent you understand well enough to trust.

---

## Contents

0. Guiding principles
1. Layer 1 — Core single agent: workspace files, memory, models, security baseline, scheduled work, observability, human gates, minimal config, durable state files
2. Layer 2 — One gateway, multiple agents
3. Layer 3 — Fleet: machines and hosted deployments
4. Engineering process for changes
5. Setup checklist
6. Common failure modes
7. References
0a. Threat model · 8. Appendices: A workspace skeletons · B artifact templates · C directory layout · D helper function signatures

---

## 0. Guiding Principles

1. **Ship first, iterate after.** A working agent with plain instructions beats a perfect plan.
2. **Text is memory.** Agents forget between sessions. Everything important lives in files.
3. **Least privilege by default.** External actions require human approval. Secrets never enter agent context.
4. **Isolation is a spectrum.** Match isolation to risk:

   | Risk level | Example | Isolation technique |
   |---|---|---|
   | Low — trusted local work | File organization, research | Same process/workspace, read rules only |
   | Medium — third-party code | Skills, plugins, scripts | Separate OS user or container, no network by default, read-only filesystem |
   | High — different trust domain | Client work (non-hosted) | Separate agent + workspace + credentials; same machine acceptable only with strict OS-level separation |
   | Critical — personal/family vs. business, or serving untrusted parties | Family agent vs. work agent; hosted agents for external users | Separate physical machine or separately hosted instance with isolated kernel, zero shared credentials |

5. **Human gate for anything that leaves the machine.** Sending messages, spending money, publishing content. See §1.8 for implementation patterns.
6. **Auditable procedures.** Recurring work becomes written procedures (SOPs), not tribal knowledge.

---

## 0a. Threat Model (who we are defending against)

Every defense in this template maps to a concrete adversary. Know them:

| Adversary | Vector | Primary defenses |
|---|---|---|
| Malicious web content / documents / emails | Prompt injection via fetched or read content | Tool-output treated as untrusted; schema validation; read/write approval split (§1.5) |
| Compromised or malicious third-party skill | Credential theft, exfiltration, code execution | Pre-install scanning; sandboxed execution; no keychain/host access (§1.5) |
| The model provider itself | Data retention, training on prompts, breach | Provider trust boundary; local models for sensitive work (§1.5) |
| Another process on the machine | Secret theft via argv/env/transcripts | Keychain-only secrets; no-argv rules; transcript redaction (§1.5) |
| The agent itself (misuse or compromise) | Self-modification, unauthorized outbound actions | Write-allowlist on brain files; tool-enforced human gate; two-phase ledger (§1.5, §1.8) |
| Physical theft / backup leak | Unencrypted data at rest or in transit | Full-disk encryption; encrypted off-machine backups; restore drills (§1.5) |
| A poisoned memory file | Persistent injection across sessions | Search results as data-only; deterministic promotion; periodic audit (§1.3) |

If a recommended control feels heavy, check which row it defends against before dropping it.

## 1. Layer 1 — Core: Single Agent

> **The Layer 1 baseline, in two stages.** **Day 1 agent:** gateway + one agent + the five workspace files (Appendix A skeletons) + primary model + at least one fallback + one web-chat channel + daily memory files + secrets in the keychain. That is a working agent — genuinely Day 1, with one caveat: the exact CLI commands for agent creation come from the current docs, and expect small friction there on first setup. Budget a few hours for the full first setup (agent creation, instruction files, gate, ledger wiring) — the template is honest that the hardened state takes weeks, but even Day 1 is more than an hour. **Hardened agent (incremental, over the first 1–2 months of part-time work for a solo builder — each item ships value on its own, and the agent does real work the whole time):** add the outbound human gate (§1.8), git-versioned workspace with encrypted backup and one restore test (§1.5), simple cron with timeout, and the run ledger with a consecutive-error alert (§1.7). Everything past that (semantic search, advanced approval workflows, the full daily-digest stack) is progressive enhancement — add each piece when the work it protects actually exists.

### 1.1 Components

| Component | Purpose |
|---|---|
| Gateway | One long-running process. Routes channels, tools, sessions. |
| Agent | One identity: name, persona, model, workspace. |
| Workspace | Directory of instruction files + memory. Per-agent, file-based. |
| Model provider(s) | Primary model + fallback chain. Never a single point of failure. |
| Channels | How humans reach the agent: web chat, messaging apps, voice. |
| Scheduler | Cron jobs and heartbeats for periodic work. |
| Skills/Tools | Extend what the agent can do: file ops, shell, web, custom tools. **Tools** are executable functions the runtime exposes to the agent; **skills** are instruction/workflow packages that teach the agent to use tools well. Third-party ones of both kinds are untrusted code (§1.5). See the capability-boundaries split in §1.5 before deciding what lives where. |

**Component relationships (Layer 1):**

```text
 Human ──▶ Channels (web chat now; messaging later)
              │
              ▼
          Gateway ──── model providers (primary + fallback chain)
              │
              ▼
           Agent ────▶ Tools/Skills (built-in core + custom, registered)
              │                 │
              ▼                 ▼
        Workspace files     data/ (run ledger, recall store,
        (SOUL/AGENTS/MEMORY  pending-actions, index — outside
         + memory/)          version control)
              ▲
              └── Cron/heartbeat jobs write outcomes to run ledger
```

The gateway should not be network-facing: the built-in web chat must be bound exclusively to 127.0.0.1 (never 0.0.0.0) and must never be exposed to any network interface. Any remote access goes through a separate authenticated fronting proxy with its own TLS termination — never the gateway itself. The agent reads its instructions from the workspace at session start and persists memory to files. Custom tools are the only things that touch the outside world on the agent's behalf — which is why enforcement lives there.

### 1.2 Workspace Files (the instruction layer)

| File | Contents |
|---|---|
| `SOUL.md` | Persona, values, boundaries, tone. The "who." |
| `AGENTS.md` | Operating rules: session startup steps, memory rules, safety rules, tool notes. |
| `USER.md` | Durable user profile and preferences the agent serves. |
| `IDENTITY.md` | Name, avatar, one-line self-description. |
| `MEMORY.md` | Curated long-term memory. Loaded only in private, direct conversations. |

**Writing rules for these files:**
- Short, declarative, imperative sentences. One idea per line where possible.
- Say what TO do and what NOT to do. Vague guidance produces vague behavior.
- Never put secrets, phone numbers, addresses, or account identifiers in any workspace file. Reference them by location (e.g., "credentials are in the system keychain, service `X`").

### 1.3 Memory System

Four subsections: what is required at Layer 1 (baseline), the optional semantic upgrade, how promotion works, and the defenses/context rules. The baseline is short — everything after the first block is optional.

#### Baseline memory (required at Layer 1)

- **Daily files:** `memory/YYYY-MM-DD.md` — raw session logs, append-only. **How they get written:** an operating rule in `AGENTS.md` ("log decisions and outcomes to today's file in the same session") makes the agent itself the writer, via its normal file-write tool, at the moment something worth logging happens — not in a batch at session end, which is where logs get lost. A heartbeat can additionally flush anything unsaved. Verify on Day 1: after the first real session, `ls memory/` shows today's file with real content.
- **Long-term file:** `MEMORY.md` — curated index of decisions, lessons, active work. Keep it small (an index, not the territory). Maintain it periodically; prune stale entries. Loaded only in private, direct conversations — never in groups.
- One rule goes into the agent's instructions: "search memory before answering questions about past work." That completes baseline memory. Everything below is optional and can wait.

#### Enhanced memory: semantic search (add when memory grows past a few dozen entries)

Start simple: full-text search first (SQLite FTS over your memory directory, or plain `grep`/`ripgrep` from a small search tool) already gives real search behavior with near-zero engineering. Graduate to embedding-based semantic search only when plain text search misses paraphrased content.

**Recommended components:** a small local embedding model (300M–1B parameters is enough; runs on CPU or a single GPU), a local vector store (SQLite with a vector extension, LanceDB, or similar — no cloud dependency needed), and chunking per memory file section rather than per line.

**Indexing strategy:** re-embed changed files on write or on a 5–15 minute schedule; keep the index next to the memory directory; version the index schema so a format change can rebuild from source files.
  - **Supervise the embedding service (once semantic search is on):** the embedding model and vector store are long-running processes — run them under the same supervisor as the gateway (restart-on-failure), verify after reboot, bind any service port to localhost only, and handle OOM by capping context/threads at startup rather than discovering it mid-run. A dead embedding service means search silently falls back or fails — the digest's known-good-query check (below) is what catches it.
  - **Rebuild observability:** the index rebuild job reports to the run ledger like any other scheduled job. Known failure classes and responses: embedding model unavailable → retry with backoff, then alert; index corruption → delete and rebuild from source files (always safe); schema drift → scripted migration, never a silent dual-format read. A failed rebuild degrades search silently unless the ledger catches it — treat index health as part of the daily digest. When the memory directory grows into thousands of daily files, move the *index* into a dedicated data directory outside the always-synced workspace (the recall store already lives at `data/.recall.jsonl` per the Layer 1 baseline). The index stays fully rebuildable from source files either way, so its location is a performance choice, not a durability one.

#### Recall store + promotion pass

The agent should search memory before answering questions about past work. Searches feed a recall store; a periodic "promotion" pass moves frequently-recalled notes into long-term memory (additive — the daily file keeps the original).

- **Integration shape:** two pieces connect this to the runtime. (1) A search tool registered with the gateway — input: query string; output: the top-k matching snippets with their file paths and line ranges, which the agent can then read in full with its normal file tool. (2) The promotion job — selection logic stays deterministic (a script reads the recall statistics file and picks candidates above a hit-count/score threshold); if you want distilled summaries rather than raw excerpts, the script may then make a controlled, schema-constrained LLM call to write the distilled line before appending it to the long-term file. Either way, the script executes the write; the LLM never decides what gets promoted.

- **Recall store schema — two views of the same data, do not conflate them:**
  - **Raw event format (what is persisted):** a single JSONL file at `data/.recall.jsonl`, one line per search event:
    ```json
    {"ts": "<iso-8601>", "query": "<query text>", "hits": [{"file": "memory/YYYY-MM-DD.md", "line_range": "40-52", "score": 0.83}], "max_score": 0.83}
    ```
    Note: in the raw event, `hits` is an ARRAY of hit objects. `promote()` never sees this raw format — it operates on the aggregated view produced by `aggregate(...)`, which converts hits into integer `item.hits` and `item.avg_score` (next bullet).
  - **Aggregated statistics (what `promote()` receives):** the promotion script aggregates raw events by `(file, line_range)` over a rolling window (e.g., 30 days) BEFORE `promote()` runs, producing per-group records where `item.hits` is an INTEGER event count and `item.avg_score` is the mean of the per-hit `score` values. The pseudocode below operates on this aggregated view only.
  - **`aggregate()` starter (the error-prone part, made concrete):**
    ```text
    def aggregate(events, key_fn, within_days, now_ts):
        cutoff = now_ts - within_days * 86400
        groups = {}                     # key -> {"hits": int, "scores": []}
        for e in events:                # events already sorted or sort here by ts
            if parse_ts(e.ts) < cutoff: continue
            for h in e.hits:
                g = groups.setdefault(key_fn(h), {"hits": 0, "scores": []})
                g["hits"] += 1
                g["scores"].append(h.score)
        return [ {"file": k.file, "line_range": k.line_range,
                  "hits": g["hits"],
                  "avg_score": sum(g["scores"]) / len(g["scores"])}
                 for k, g in groups.items() ]
    ```
    Read the raw file ONCE per run, aggregate in memory, then promote from the aggregated list. For very large stores, maintain a running per-day aggregate file and merge only the new day's events at each run.



- **Starter implementation (pseudocode in a Python-like style; the promotion script's aggregation and file handling can be a small Python script using only the standard library — the optional `distill()` step may use an LLM client — and v0 of the search tool depends on the `ripgrep` binary or equivalent; for very large recall stores, aggregate incrementally rather than reading the whole file):**
  ```text
  # search tool (registered with the gateway; called by the agent)
  def memory_search(query, k=5):
      # v0: ripgrep over memory dir (no embeddings needed)
      results = ripgrep(query, path="memory/", max_results=k)
      # v1: rank with embeddings over pre-chunked sections
      #   query_vec = embed(query)
      #   results   = vector_store.top_k(query_vec, k=k)
      log_recall_event(ts=now(), query=query, hits=results)   # append to recall.jsonl
      return format(results)   # file paths + line ranges + snippet text

  # promotion script (cron, weekly; NOT an agent decision)
  # input: the aggregated statistics returned by
  #   aggregate(events, by=(file, line_range), within=rolling_window).
  # promote() must never operate directly on raw recall events.
  def promote(rolling_window=30_days, min_hits=3, min_avg_score=0.7):
      events = read_jsonl("data/.recall.jsonl")
      stats  = aggregate(events, by=(file, line_range), within=rolling_window)
      for item in stats:
          if item.hits >= min_hits and item.avg_score >= min_avg_score:
              line = distill(item)                     # optional: schema-constrained LLM call, or plain first-line excerpt
                                               # distill() governance: pinned model, its own rate-limit
                                               # budget, reports to the run ledger like any job, and is
                                               # SKIPPED (falls back to plain excerpt) whenever the digest
                                               # is already failing — never let polish outrun health
              if append_once("MEMORY.md", line):       # returns true only if actually appended
                  mark_promoted(item.file, item.line_range)   # keyed on the aggregation key, not the whole record
      rotate("data/.recall.jsonl", keep_days=rolling_window)  # trim recall store to the rolling window
      log_job_outcome("promote-memory", "ok" if ok else "error",
                      detail=error_msg or "")   # promotion is a scheduled job: report to the run ledger (§1.7)
  ```
  Swap the ripgrep line for the embedding/vector-store path when text search stops finding paraphrased content — the recall logging, promotion logic, and schemas stay identical.

  **Treating memory search as an untrusted-input surface:** memory files can absorb injected instructions (a poisoned web page the agent once summarized, a malicious document). Defenses: the search tool returns file path + line range + snippet only — the agent must re-read the cited region with its normal file tool before acting on it; snippets are rendered as quoted data in the agent's context, never executed; and the weekly promotion pass (deterministic script) is the only *automated* writer to the long-term `MEMORY.md` index file (the agent itself writes daily session files automatically, and humans may edit MEMORY.md directly), so a single poisoned search hit cannot silently rewrite durable memory.

  **Where these live in an OpenClaw workspace:** the search tool under `tools/`, the promotion script under `scripts/` — both inside the version-controlled workspace, part of the agent's brain, included in the backup/restore drill and §4 changes. The recall store is the exception: it belongs at `data/.recall.jsonl`, OUTSIDE the version-controlled workspace and added to `.gitignore` — raw queries may contain PII, so it is backed up under the same encrypted-at-rest rules as the workspace but is never committed with the instruction files. They are also part of the trusted computing base, so they get the same observability as everything else: the search tool logs each call (query, hit count, latency) to the structured log directory (§1.7) — because queries may contain PII, the structured log directory and `data/` live outside version control, are never synced unencrypted, and are protected by the host's full-disk encryption (FileVault on macOS, LUKS/dm-crypt on Linux). For additional defense-in-depth on high-sensitivity deployments, use per-directory encryption (e.g., `fscrypt` on Linux or an encrypted APFS volume on macOS). All backups that leave the machine are independently encrypted with a separate key. Both locations are included in the restore drill. Scheduled memory jobs report to the run ledger, and the promotion script writes an explicit success/failure line per run. A promotion script that fails silently is how memory silently stops improving — route its failures into the §1.7 alert path. The search tool is a custom tool registered with the gateway: a small module that exports the function above plus a machine-readable declaration (name, input schema, output schema). The search tool must be implemented as a module the *current* OpenClaw release can load. The exact loading mechanism (a `register()` function, a decorator, a manifest, or a specific interface) is version-specific and **must** be taken from https://docs.openclaw.ai under the current Tools/Plugins section. The pseudocode and JSON declaration in this document describe only the required semantics and data flow (search → log recall event to `.recall.jsonl` → return structured hits with file paths and line ranges). Do not treat any function signature or registration code shown here as canonical.

**1.3.1 Minimal Custom Tool Skeleton (illustrative but realistic):** the most common OpenClaw pattern is a single Python module whose exports the gateway discovers. Marked illustrative — verify the loader signature against the current docs before relying on it:

```python
# tools/memory_search.py — memory search tool, v0 (ripgrep-based)
import subprocess, json, time

DECLARATION = {
    "name": "memory_search",
    "description": "Search the agent's own memory files. Returns file paths, line ranges, and snippet text.",
    "input":  {"type": "object",
               "properties": {"query": {"type": "string"}, "k": {"type": "integer", "default": 5}},
               "required": ["query"]},
    "output": {"type": "object",
               "properties": {"hits": {"type": "array"}}},
}

def memory_search(query: str, k: int = 5) -> dict:
    out = subprocess.run(
        ["rg", "--max-count", str(k), "--json", query, "memory/"],
        capture_output=True, text=True, timeout=30,
    )
    if out.returncode not in (0, 1):
        raise RuntimeError(f"ripgrep failed: {out.stderr[:200]}")
    hits = []
    for line in out.stdout.splitlines():
        if not line.strip():
            continue
        try:
            entry = json.loads(line)
            if entry.get("type") == "match":
                data = entry["data"]
                hits.append({
                    "file": data["path"]["text"],
                    "line_range": str(data["line_number"]),
                    "snippet": data["lines"]["text"].strip(),
                })
        except Exception:
            continue          # skip malformed lines; the telemetry row still logs the query
        if len(hits) >= k:
            break
    # recall telemetry (PII: this file lives in data/, outside version control)
    with locked("data/.recall.jsonl"):
        append_line("data/.recall.jsonl", {
            "ts": iso_now(), "query": query, "hits": hits,
            "max_score": max((h.get("score", 1.0) for h in hits), default=0.0),
        })
    return {"hits": hits}

def register(registry):          # loader entry point — VERIFY signature vs current docs
    registry.add(memory_search, DECLARATION)
```

The loadable entry point (`register`, decorator, or manifest) is the one line that changes between releases; the function body and declaration transfer as-is.

> ⚠️ **Version-skew note (relevant to the memory search tool in §1.3):** when you implement the tool-registration step, capture in your own notes which OpenClaw release you verified it against (a dated line in your `AGENTS.md` tooling section is enough). Re-verify the registration after every gateway upgrade — loader API changes are exactly the class of change that turns a working memory system silent.
>
> **How to use:** Read Layer 1 first. Add Layer 2 when you run more than one agent. Add Layer 3 when you run more than one machine or serve other people. Each layer builds on the one before it.
>
> **What is OpenClaw?** OpenClaw is an open-source personal AI agent runtime (gateway + agents + skills + channels). Docs: https://docs.openclaw.ai — Source: https://github.com/openclaw/openclaw. This template is written using OpenClaw conventions (file names, component model, workspace layout). The architectural patterns and isolation strategies are broadly applicable to other agent runtimes.

Registration mechanics are version-specific and must be taken from https://docs.openclaw.ai under the current Tools/Plugins section. The pseudocode and JSON declaration in this document describe only the required semantics and data flow — do not treat any function signature, export pattern, or registration code shown here as canonical. **Do not skip this step:** the memory system does not work until the tool is registered through the current plugin/tool mechanism and you have verified the agent can call it (ask the agent to search memory and confirm it answers from real files). Illustrative declaration shape:

```json
{
  "name": "memory_search",
  "description": "Search the agent's own memory files. Returns file paths, line ranges, and snippet text.",
  "input":  { "type": "object", "properties": { "query": { "type": "string" }, "k": { "type": "integer", "default": 5 } }, "required": ["query"] },
  "output": { "type": "object", "properties": { "hits": { "type": "array", "items": { "type": "object", "properties": { "file": {"type":"string"}, "line_range": {"type":"string"}, "snippet": {"type":"string"} } } } } }
}
``` The promotion script is a plain cron-invoked script under the workspace (e.g., `scripts/promote-memory`); it needs no gateway registration. The recall store lives at `data/.recall.jsonl` (outside the version-controlled workspace, added to `.gitignore` — see the PII note above).
- **Rule:** if it matters tomorrow, write it to a file today. Agents must never rely on "mental notes."
- **Context-window management (add before memory outgrows the window):** total injected memory (long-term file + today's file + search hits) must stay inside the practical context budget. A strategy that holds in practice:
  - **Tiered recall:** long-term file stays small (an index, not the territory — one line per item with a pointer); details live in daily/archived files; semantic search pulls only the needed lines into context at answer time.
  - **Eviction/archival:** on a fixed schedule (e.g., monthly), move stale detail from the long-term file into dated archive files. The index line stays; the bulk leaves the always-loaded set.
  - **Summarization on promote:** when the promotion pass moves a daily note into long-term memory, store the distilled line, not the raw text. Raw stays in the daily file.
  - **Budget check (concrete):** at session start, the agent's startup rule includes one action: sum the byte size of `MEMORY.md` + today's daily file + the last search results (bytes ≈ tokens/4 for English text). If the sum crosses half the practical context window, the agent archives before adding anything new. The tool layer can also enforce this — a wrapper that measures what gets injected and logs the number to the run ledger gives you the trend line for free.

### 1.4 Model Configuration

- Define an explicit **primary model** and an ordered **fallback chain** (2–3 providers).
- Pin the model for every scheduled job. Do not let jobs inherit a mutable default.
- **Context windows differ across the fallback chain.** A prompt sized for the primary may overflow the fallback. Set the working context budget to the SMALLEST window in the chain, and keep injected memory (§1.3 budget check) inside that number. When the budget is approached, the agent must either trigger archival/summarization before adding content or refuse the action with a clear error — never rely on the provider to truncate gracefully.
- If your primary provider rejects certain sampling parameters, disable those parameters for that provider (a common silent-failure class: jobs failing for weeks because one provider rejects a `reasoning` flag).
- Keep at least one local or self-hosted model as a resilience option where hardware allows.

### 1.5 Security Baseline (non-negotiable, applies to every layer)

- **Secrets:** store in the OS keychain or a secrets manager. Never in config files, never in workspace text, never pasted into chat. Retrieve via a subshell (`$(...)`) so the value never prints.
- **Secret injection pattern (concrete):** tools and scripts must read secrets at execution time without materializing them in command-line arguments, in environment variables visible to other processes, or in tool output. Preferred approaches, in order:
  1. **Wrapper binary or language-specific client** that reads directly from the keychain/secrets store and makes the call itself. The secret never crosses a shell at all.
     Starter shape (Python, macOS keychain): a ~20-line script that calls `subprocess.run(["security","find-generic-password","-s",svc,"-w"], capture_output=True)` and passes the value straight into the request header — the secret exists only inside the script's memory, never in argv, env, output, or logs. Linux: `secret-tool` via the same pattern, or the `keyring` library.
  2. **Config file at a `0600` path** owned by the tool's OS user, read by the tool at startup.
  3. **Shell fallback is strongly discouraged.** Even on single-user machines, environment variables are visible to other processes under the same UID via `/proc/<pid>/environ` on Linux, and argv is visible via process listings. For anything touching money, deletion, or irreversible actions, use a compiled wrapper binary or native client that reads directly from the keychain. One-shot command substitution passed via stdin or a file descriptor is acceptable only for low-risk reads when a native client is not feasible. The secret value must never become a tool result, appear in logs, or persist in a shell variable beyond the single command.
  Hard rules: (a) never let the secret value become a tool *result* that can be persisted into the session transcript — never assign it to visible output, never echo it, never log it; (b) redact credential fields from any command output before storage (many API/CLI receipts echo credentials back); (c) for anything security-critical, use a compiled helper or the runtime's native credential API rather than shell indirection.
- **Capability boundaries (what lives where):** a common architectural failure is putting every capability in the wrong tier. The split that holds up in practice:
  - **Built-in core tools** (file read/write, shell, web fetch): generic, carefully permissioned, shared across everything the agent does. Treat as the trusted computing base — changes to these go through the full §4 process.
  - **Custom tools** (registered with the gateway): narrow, single-purpose functions with schema-validated inputs/outputs and the least privilege that works. Anything that touches money, deletion, or external sends is a custom tool with its own approval path — never a prompt rule on a general tool.
  - **Skills** (instruction packages): knowledge and workflow, not authority. A skill can teach the agent *how* to do something; it can never grant *ability* the tools do not already have. If a workflow needs new power, that is a new custom tool with review — not a skill edit.
  - **External scripts** (cron-invoked, non-agent): deterministic operations on destructive/high-stakes paths. The agent proposes, the script executes, a human approves. See §3.4.
  - Rule of thumb: **authority lives in tools, knowledge lives in skills, determinism lives in scripts, and judgment lives in the agent** — never the reverse.
- **Session transcripts are a leak vector.** Any tool result can be persisted and forwarded to a model provider. Mask or redact secret-bearing tool output before it is stored. Never run commands that print raw secret values.
- **Outbound gate:** email, posts, payments, and messages to third parties require explicit human approval. Reads are free; writes are gated.
- **PII rule:** never put real names, addresses, phone numbers, or other personal data in any workspace file under any circumstances — local-only included, because local repos get pushed or leaked later. Use pseudonyms, initials, or record identifiers exclusively. Unencrypted files that leave the machine (synced, shared, emailed) are a PII leak by definition.
- **Credential rotation:** expired or revoked model-provider keys are a top silent-failure source — every job through that provider fails until someone notices. Add each key's expiry to the daily digest balance checks (or a calendar/cron reminder ahead of expiry); on rotation, update the keychain entry and re-run one verification call per provider before declaring the rotation done.
- **Model provider trust boundary (the biggest one people skip):** everything the agent sends to a cloud model provider — its instruction files, memory, search results, conversation content — becomes visible to that provider. Treat every model provider as an untrusted third party. Pick one, explicitly: (a) sensitive workspaces use only local/self-hosted models; or (b) you consciously accept that your agent's brain and memory are processed by the provider (check their data-retention/training terms). Document the choice in `AGENTS.md`. The same logic applies to the memory search tool: snippets sent to the provider carry whatever the memory files contain.
- **Pre-push secret scan:** run a pattern-based scan on every push to any repo. Block, classify, fix, then push. Prefer fixing test data over bypassing the scan.
- **Backups and recovery (Layer 1 required, not optional):** the workspace is the agent's brain — protect it like source code.
  - **Version it:** keep the workspace (instruction files, memory, procedures) in a local git repo. Commit after significant sessions. This is the cheapest possible backup and the only one with real recovery semantics.
  - **Encrypt off-machine copies:** any backup that leaves the machine (external drive, second machine, cloud) is encrypted at rest (full-disk encryption on the target, or an encrypted archive). Plain-text memory files can contain sensitive personal information and deserve strong confidentiality controls (encrypted backups, careful sync choices).
  - **Test the restore:** a backup is real only after you have restored from it once. Backups get two verification tiers: a **smoke test** (automatic: the daily digest includes a check that yesterday's backup exists, decrypts, and contains today's memory file) and a **full restore drill** (manual: restore to a scratch directory and run the four checks above) run after major workspace-format or tool changes and at least once before you rely on backups for critical work. The smoke test catches the common failures cheaply; the full drill catches the rest. The drill verifies the full surface, not just files: restore to a scratch directory, point a test gateway at it, then confirm (a) memory files and search work, (b) custom tools load and register, (c) the pending-actions ledger is intact, (d) cron jobs fire with the right model and delivery target.
  - **Vector-index corruption:** the semantic index is disposable by design — it must always be rebuildable from the source markdown files. If it corrupts or drifts, delete and rebuild; never hand-edit index files.
  - **Layer 3 addition:** one-directional backup across machines (§3.2), with the same encrypted-at-rest rule and restore drill.
- **Updates:** dry run first, review the diff, then apply. Verify plugins and skills still load after every upgrade.
- **Skill/plugin installs:** scan third-party skills for credential theft, code injection, prompt manipulation, and exfiltration before installing (use pattern scanners such as `gitleaks`, `semgrep` with agent-specific rules, plus manual review of any file that touches credentials or the network). Treat them like you treat dependencies. Run all third-party skills in a sandboxed environment (separate container or OS user with minimal privileges — e.g., `docker run --rm --read-only --network=none --cap-drop=ALL` — no network access by default, read-only root filesystem where possible). Never grant third-party skills direct keychain, credential-store, or host filesystem access.
- **Prompt injection and tool-output defense (your own tools too):** third-party skills are not the only injection vector — any web page, document, email, or API response the agent reads can carry instructions. Defenses that hold in practice:
  - Treat all tool output and fetched content as untrusted data, never as instructions. Enforce this at the runtime/tool layer (schema validation, output sandboxing, allowlists, separate read/write approval paths). Supplement with the rule in instruction files — a prompt rule alone is insufficient.
  - Validate tool inputs and outputs against a schema before acting on them; reject or quarantine mismatches.
  - Constrain destructive tool capabilities at the tool layer (allowlists, dry-run flags, read-only defaults), not just in prompts. A prompt rule is a control, not a guarantee.
  - **Protect the agent against self-modification:** the file-write and shell tools must maintain an allowlist that explicitly denies writes to `SOUL.md`, `AGENTS.md`, `tools/`, `scripts/`, and any gateway configuration files. Self-modification of the agent's brain or tools is a critical action requiring human approval at the tool enforcement layer — the same `gate_check` path as money or external sends.
  - Keep write-capable tools on a separate approval path from read-only tools (see §1.8).

### 1.6 Scheduled Work (cron/heartbeat)

- Batch similar periodic checks into one heartbeat rather than many jobs.
- Use cron for exact times, standalone tasks, and isolated sessions.
- Every job: pinned model, isolated session, sensible timeout, defined delivery destination, and an alert rule (e.g., page the human after N consecutive errors — silent cron failure is the most common operational surprise).

### 1.7 Observability (add before the first real task, not after the first outage)

- **Structured logs:** every gateway, scheduler, and job run writes a timestamped line with job id, outcome, and duration to a local log directory. Rotate them.
- **Run ledger (scheduled-job outcomes):** cron and heartbeat outcomes append to a JSONL (JSON Lines) file. Alert rules read from this file — this is what makes "job silently failing for weeks" detectable in minutes instead. Starter schema:
  ```json
  {"job_id": "daily-health", "ts": "2026-11-19T10:00:00Z", "outcome": "ok", "duration_ms": 8421, "detail": "example detail"}
  ```
  (Example values are valid JSON; in real records `outcome` is one of `"ok" | "error" | "timeout"` and `ts` is an ISO-8601 timestamp.)
  The alert rule reads the last N records per `job_id` and counts consecutive errors; the daily digest job summarizes the same file. Starter alert logic (stdlib-only, cron every 15 min):
  ```text
  def check_alerts(ledger_path, threshold=3, alert_channel="..."):
      recent = last_records_per_job(ledger_path, n=threshold)
      for job_id, records in recent:
          errors = count_trailing(records, outcome="error|timeout")
          if errors >= threshold:
              notify(alert_channel,
                     f"job {job_id}: {errors} consecutive errors")   # destructive/billing jobs: page the owner directly
          elif all_success(records) and was_alerting(job_id):
              notify(alert_channel, f"job {job_id}: recovered")        # recovery notice, so alerts self-resolve
  ```
  Wire the "page the owner directly" branch to your highest-attention channel; everything else goes to the digest.
- **Daily digest (starter shape):** one scheduled job sends a short health summary to a low-priority channel. Starter skeleton (stdlib-only):
  ```text
  def daily_digest():
      lines = ["=== agent daily digest ==="]
      # 1. scheduled jobs: read run ledger, report last 24h ok/error counts per job
      for job_id, counts in summarize_last_24h("data/run-ledger.jsonl"):
          lines.append(f"{job_id}: {counts.ok} ok / {counts.errors} err")
      # 2. balance checks: every metered resource, value vs threshold
      for name, value, warn_at in balances():          # provider APIs or usage endpoints
          if value < warn_at: lines.append(f"LOW {name}: {value} (warn at {warn_at})")
      lines.append(f"disk: {disk_free_pct()}% free | mem pressure: {mem_status()}")
      # 3. index health: one known-good memory search must return results (§1.3)
      lines.append("memory search: " + ("ok" if search_returns_hits() else "FAILING"))
      # 3b. human gates: pending approvals aging toward expiry
      stale = [r for r in pending_rows() if r.status == "pending" and hours_left(r) < 12]
      if stale: lines.append(f"APPROVALS WAITING: {len(stale)} (oldest {oldest_age(stale)}h)")
  # 4. supervision: gateway unit healthy? digest written in last 24h?
      lines.append(f"gateway: {supervisor_status()} | last digest ok: {last_digest_age_h()}h ago")
      post("
".join(lines))   # web chat at Layer 1; external channel at Layer 2
  ```
  Every check is one provider call or one local command. Anything below threshold gets a line; healthy things get one summary line. Humans read digests; they do not read logs. The digest job itself is a critical scheduled job: it writes to the run ledger like everything else, gets its own consecutive-error alert (escalating after 2 failures), and the gateway supervision check verifies a digest succeeded within the last 24h — if the digest stops, that is itself an alert, not a silent gap.
  **Digest delivery in a web-chat-only setup:** with no external channel, the digest is a message the agent posts into its own web chat on a schedule — acceptable at Layer 1 because the owner opens the chat daily. Its alert path, however, must not depend on the chat being read: the escalation branch writes a `digest-stalled` flag file that any external integration (an email CLI, a messaging webhook — added at Layer 2) picks up. At Layer 1, the flag file IS the mechanism: any cron job (even a plain shell `test -f` check) can watch for `digest-stalled` and act — e.g., append to a system log, trigger a local notification. Wire a real external delivery (email CLI, messaging webhook) before Layer 2 makes the agent business-critical.
  What "balance checks" means in practice: query every metered resource the agent depends on and compare against its threshold — API credit/DIEM balances (provider dashboard APIs or usage endpoints), token-budget remaining per the spend governance numbers, disk free space (`df -h %` style local check), memory pressure (host metric), and any prepaid quota (e.g., search API credits). Each check is one provider call or one local command; the digest reports value vs. threshold for each. Thresholds: warn at 20–25% remaining, alert at 10%.
- **Escalation split:** routine failures go to the digest; destructive or money-relevant jobs escalate to the owner directly after a small threshold (2–3 consecutive errors).
- **Provider rate limits and budget enforcement:** configure per-provider rate limits and request budgets at the gateway/proxy layer, with exponential backoff plus a circuit breaker on repeated rejections (a provider returning 429s should shed load, not burn credits or stall jobs). Rate-limit errors count toward the consecutive-error alert — a provider silently degrading is the same operational failure as a job dying.
- **Gateway supervision:** the gateway is a long-running process — treat it like production infrastructure. Run it under a process supervisor (systemd unit with `Restart=on-failure`, launchd `KeepAlive`, or the equivalent), verify it comes back after a crash and after a host reboot, and rotate the gateway's own logs. Minimal systemd shape:
  ```ini
  [Service]
  ExecStart=<gateway start command>
  Restart=on-failure
  RestartSec=5
  [Install]
  WantedBy=default.target
  ```
  First failure modes to expect and check: permission errors on `data/` (the supervisor's user must own it — test with a manual run as that user), lockfile leftovers blocking startup (clear `.lock` sidecars after a crash), and the unit not starting after reboot (verify with an actual reboot test once).
  Monitor the supervisor itself in the daily digest: on systemd hosts, check `systemctl --failed` (or launchd equivalent via `launchctl list`) and alert on the gateway unit appearing there — a dead supervisor restarting nothing is the same outage class as a dead gateway. A monitored agent behind an unmonitored gateway is still an outage.
- **Local inference governance:** if using local or rented GPUs, enforce a hard wall-clock timeout and/or token-per-minute limit at the inference wrapper or model-server layer. Include GPU temperature, power draw, and hourly spend (if rented) in the daily digest — a runaway local agent can produce surprise four-figure electricity or rented-GPU bills just as easily as a cloud model loop.
- **Anomalous-tool-usage detection:** log (or sample) which tools the agent calls and flag patterns worth a human look: repeated self-modification attempts (writes near protected files), unusual spikes in memory_search volume (possible injection probing), a tool called with payloads unlike its history, or any write tool hitting paths outside the workspace. This does not need to be clever — a daily-digest line "tools called: file-write 12, web-fetch 40, memory-search 3 (normal)" plus an alert on threshold breach covers most real cases.
- **Spend governance (every layer, not just hosted fleets):** track model token/cost usage per agent and per scheduled job from day one. Set a monthly budget with a soft warning (e.g., at 80%) and a hard cutoff (at 100%) enforced at the provider or proxy layer — not by prompt rules. Review the top consumers weekly; runaway loops and recursive agent calls are the most common source of surprise bills.

### 1.8 Human-Gate Implementation Patterns

"Human approval" needs a concrete mechanism. Pick one of these patterns, in rising order of effort:

Rules that hold for all three:
- The human sees the **exact final payload** before approving — no summaries of what "will" be sent — via the privileged viewer (pattern 1's decryption flow); the payload itself never enters agent context or any model call.
- Approval is per-action and single-use. Bundled approvals get split.
- The pending state is durable (a file or database row), so a restart cannot silently drop or auto-approve an action.
- The agent never interprets a vague "sure" as approval for a different action.
- **Enforcement lives in the execution tool, not the agent.** Any tool that can send, spend, or publish checks the pending ledger itself before acting: it verifies that an `approved` row exists for this exact action, that the payload hash matches, and that the row has not expired. A missing or mismatched row is a hard refusal at the tool layer. This is what makes the gate survive prompt injection — the agent cannot talk a tool into skipping a check the tool performs on its own inputs. Starter check (stdlib-only, runs inside the tool before it acts):
  ```text
  def gate_check(action_type, payload_hash):
      rows    = read_jsonl("data/pending-actions.jsonl")
      row_id  = pending_id_for(action_type, payload_hash)
      approved = latest_row(rows, id=row_id, status="approved")
      if not approved:                          refuse("no approved row")
      if approved.payload_hash != payload_hash: refuse("payload mismatch")
      if now() > approved.expires:              refuse("approval expired")
      if exists_row(rows, id=row_id, status="executing"): refuse("concurrent execution")
      if exists_row(rows, id=row_id, status="executed"): refuse("already executed")
      append_row(rows, {"id": row_id, "status": "executing", "executed_at": now()})
      return PROCEED   # tool performs the side-effect, then appends status "executed"
  ```
  **Crash safety (required for money/sending tools):** the two-phase status above closes the crash window. If the tool dies after the side-effect but before writing `executed`, the row stays `executing` — recovery on startup reconciles: check the real-world result (did the send/payment actually happen?) and write the truthful terminal status (`executed` or `failed-retry-allowed`). Never re-execute on an ambiguous row without that reconciliation; for irreversible actions, ambiguous means human investigation, not automatic retry.

---

---

The three patterns, in rising order of effort:

1. **Ask-in-channel (simplest):** the agent posts only a neutral reference — the pending action id (e.g., "Action act-20261119-001 requires approval; reply approve or deny") — to the approval channel and waits for the owner's reply. Timeout = no action. The human sees the exact payload through a privileged viewer (approval UI, or decrypting the payload file with the keychain-held key) that renders it outside the agent's context and session transcript. Works well when volume is low.
   - **Web-chat-only setups:** if the web chat is your only channel, the approval request is a normal agent message, and the owner's approval reply is matched by the pending-row id included in the agent's request ("reply with: approve act-20261119-001 / deny"). The loop-prevention rule still applies: an inbound message that is an approval id match executes nothing except the already-pending row.
   Implementation notes that prevent the common failures:
   - **Key pending state durably** before posting: a row in `data/pending-actions.jsonl` (outside version control — action metadata can contain sensitive identifiers; same `.gitignore` + encrypted-backup rules as the recall store) with an id, the exact payload hash, status `pending`, and a timestamp. The approval reply references the id — the agent matches on id, not on conversational context, so a restart cannot orphan an approval.
   - **Post to a channel the agent does not auto-reply in.** If the approval channel is one of the agent's own inbound channels, mark those inbound messages as approvals-only (no tool execution triggered by them) so the loop cannot act on its own request.
   - **Timeout = deny.** No reply within the window (e.g., 24h) marks the row `expired`; the action never executes.
   - **One action per id.** An approval of id 7 never authorizes id 8, even if payloads look similar.
- **Recurring/class approvals (Layer 2+):** when volume grows, issue a scoped standing grant instead of per-action approvals: `grant(action-class, constraints, expires)`. Example: "approve all transfers under $50 to vendor X for 30 days." The grant is itself a row in the pending ledger; the `gate_check` verifies the action against the grant's class and constraints. Every grant auto-expires. Per-action approval remains the default for anything new or unusual.
- **Human unreachability:** `timeout = deny` means an absent owner blocks outbound actions — that is the correct default. For Layer 2/3 business-critical agents, plan for it explicitly: define which action classes may batch and wait (non-urgent sends), which need a designated secondary approver, and which simply wait no matter how long. Document the plan in AGENTS.md; never widen the gate to compensate for an absent human.
- **Approval audit trail:** the pending-actions ledger doubles as the approval record — who approved, when, and the exact payload hash that executed. Rotate/archive it on a fixed schedule (e.g., yearly) into an encrypted archive; keep it tamper-evident by including the previous row's hash in each new row (a cheap hash chain). Anything touching money, legal commitments, or client work should assume the ledger will be read months later by someone who was not there.
2. **Draft-then-confirm:** the agent writes the full action to a pending-queue file (e.g., `data/pending-actions.jsonl`) with an id and status. A scheduled job or manual check picks up approved ids. Good for async approval.
3. **Approval command/UI:** the runtime or a small companion app exposes an allow/deny control with the exact payload preview shown. Best for high-volume or multi-agent setups.
   Implementation notes: the UI reads the same `data/pending-actions.jsonl`; allow/deny writes the same `approved`/`denied` status rows; the executing tool's `gate_check` is identical across all three patterns. This is the property that makes the patterns interchangeable — the enforcement never moves.

### 1.9 Minimal Gateway Config (conceptual Layer 1 baseline)

> ⚠️ Illustrative only — see "How to Treat This Document" above.

The configuration must connect one agent, its workspace, model providers, and channels.

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "workspace": "~/.openclaw/workspace",
        "model": { "primary": "provider-a/model-x", "fallbacks": ["provider-b/model-y"] }
      }
    ]
  },
  "channels": {
    "webchat": { "enabled": true }
  }
}
```

**Model provider auth:** the provider entry in the gateway config references the credential by keychain/secrets-store reference (an id or keychain service name), never by literal value. The gateway resolves the secret at request time using the same injection rules as §1.5. Shape:

```json
{
  "providers": {
    "provider-a": {
      "base-url": "https://api.provider-a.example/v1",
      "api-key": { "keychain": { "service": "provider-a-key", "account": "gateway" } }
    }
  }
}
```

If your release only supports a literal key in config, the config file must live at a `0600` path owned by the gateway's OS user and OUTSIDE the version-controlled `workspace/` directory. Never commit a file containing literal API keys to the workspace git repository, even temporarily — the pre-push secret scan must catch this class. The gateway configuration file itself must also be `.gitignore`d and kept under the agent root (outside `workspace/`), never inside the version-controlled workspace. Treat migrating to keychain-backed references as an early enhancement.

That is a complete Layer 1. Everything else (bindings, more agents, cron, extra providers) grows from here. After writing it: start the gateway, open the web chat, send a message, confirm the agent replies using its workspace files. That round-trip is the Layer 1 test.

The web-chat channel is built into the gateway — it serves a local web UI over HTTP. It must be configured with authentication (a strong random secret from the keychain used as a bearer token or basic auth) required on every request. It is the recommended first channel provided it remains bound to localhost only (127.0.0.1) AND requires authentication — even a localhost UI is an attack surface on a multi-user machine or one with compromised local software. Never expose this interface to any network without authentication and TLS termination at the gateway or a fronting proxy. See the current webchat auth options in the docs. Messaging channels (Signal, WhatsApp, Discord, and similar) are added at Layer 2 when multiple humans or agents need access.


### 1.10 Durable State Files (quick reference)

Four append-only or curated files carry the agent's durable state. They look similar; they are not interchangeable:

| File | Location | Written by | Purpose | Version-controlled? |
|---|---|---|---|---|
| Run ledger | `data/run-ledger.jsonl` (or equivalent) | Cron jobs, scheduled memory jobs | Outcome of every scheduled run; feeds alert rules + daily digest (§1.7) | No |
| Recall store | `data/.recall.jsonl` | The memory search tool | Search telemetry; feeds the weekly promotion job (§1.3) | No |
| Pending-actions ledger | `data/pending-actions.jsonl` | Agent (proposals) + executing tools (executed rows) | Human-gate state + approval audit trail (§1.8) | No |
| Structured logs | `<log-dir>/` (alongside `data/`) | Gateway, tools, scheduled jobs | Event-level logs; queries may contain PII — same rules as `data/` (§1.5, §1.7) | No |
| Shipped-work ledger | `workspace/SHIPPED.md` | You/agent after each shipped change | Provenance: what shipped, when, evidence (§4) | Yes — it is part of the brain |

All four: append-only or idempotent-row discipline, never hand-edit in place, all included in the encrypted backup except where noted.

**Retention and encryption for `data/`:** the `data/` directory gets the same encryption-at-rest rule as backups — its contents (queries, action metadata) are personal-data-bearing. Set explicit retention: rotate or archive the run ledger on a schedule (e.g., yearly), prune or anonymize recall-store events beyond the promotion window, and treat the pending-actions ledger per the audit-trail rule in §1.8. If the agent ever processes third-party personal data, deletion requests must be honor-able across ALL of these stores — know where every copy lives before you need to delete one.

**Portable locking wrapper (reference — stdlib-only, ~15 lines):**
```python
# lockfile.py — context manager for data/ ledger access
import fcntl
from contextlib import contextmanager

@contextmanager
def locked(path):
    with open(path + ".lock", "a+") as lockfh:
        fcntl.flock(lockfh, fcntl.LOCK_EX)   # blocks until acquired
        try:
            yield
        finally:
            fcntl.flock(lockfh, fcntl.LOCK_UN)
```
Usage: `with locked("data/pending-actions.jsonl"): ...` around every append and every check-then-act sequence (the `gate_check` above, the promotion aggregation). The `.lock` sidecar file holds no data. All platforms with a POSIX layer (macOS, Linux) support `fcntl`; Windows users substitute `msvcrt.locking`. Wrap this once and import it everywhere — never hand-roll locking per call site.

**Concurrency control (required, not optional):** several processes write these files concurrently (gateway, cron jobs, digest job, executing tools). Every append and every read-modify sequence (the gate check above, the promotion aggregation) must be serialized: use file locking around writes and check-then-act sequences — language-level locks (`fcntl.lockf` in Python, Go's `flock` package) preferred; the `flock(1)` binary is not installed by default on macOS. File locking is mandatory — a TOCTOU race on the pending-actions ledger is a real security boundary, and append with single atomic write calls (open in append mode, one `write()`, flush). The pending-actions ledger is part of the security boundary — a TOCTOU race between "check approved" and "execute" is a real attack surface, so the check and the status update must happen under the same lock.

## 2. Layer 2 — Team: One Gateway, Multiple Agents

### 2.1 Decision: One Gateway vs. Many

**One gateway, multiple agents** (operational-simplicity default — for agents in the SAME trust domain):
- Simpler to manage: one process, one config.
- Agents share the gateway port; requests route by agent binding.
- Isolation is per-workspace + per-agent directory + per-session store.

**Separate gateways/machines:**
- Use when isolation must be physical (see Layer 3) or when resource contention demands it.

### 2.2 Roles: Coordinator + Sub-Agents

A common, effective pattern:

- **Coordinator agent:** one per project/domain. Owns the workspace, the channel identity, and the group conversation. Speaks for the team.
- **Sub-agents:** spawned internally by the coordinator for functions such as:
  - **Marketing** — content, social, community.
  - **Sales/Outreach** — partnerships, pipelines, follow-ups.
  - **Engineering** — development, PRs, CI/CD, docs.

Sub-agents coordinate with the coordinator through internal session messaging. They do not need their own external channel identities.

### 2.3 Per-Agent Configuration

Each agent gets:
- Its own workspace directory (e.g., `~/.openclaw/workspace-<role>/`).
- Its own agent directory and session store.
- Its own instruction files (Section 1.2), tailored to the role.
- A **binding**: an entry in the gateway configuration that maps an inbound channel/target (for example, a specific chat, group, or phone identity) to a named agent. Bindings are how one gateway serves several agents without mixing their conversations.

**Binding example** (illustrative only — see "How to Treat This Document" above):

```json
{
  "agents": {
    "list": [
      { "id": "work-coordinator", "workspace": "~/.openclaw/workspace-work" },
      { "id": "personal",        "workspace": "~/.openclaw/workspace-personal" }
    ]
  },
  "bindings": [
    { "channel": "messaging-platform", "target": "<work-team-group-id>", "agent": "work-coordinator" },
    { "channel": "messaging-platform", "target": "<personal-dm-id>",      "agent": "personal" }
  ]
}
```

The same shape applies per agent: dedicated workspace, dedicated agent directory, its own instruction files, and explicit routing so that two agents never share one conversation.

### 2.4 Channel Identity Strategy

- **One identity per coordinator** is the clean default. Each coordinator has its own account/number/handle on the messaging platform.
- Sub-agents speak through the coordinator. Do not give every sub-agent an external identity.
- Human-facing groups: one team group per project (human owner + that coordinator), plus a coordinators group for cross-project coordination.

### 2.5 Group Communication Protocol

**Within a team:**
- Coordinator posts summaries and completed work to the group.
- Sub-agents post only deliverables or requests for decisions.
- Rule: the coordinator is the voice. Sub-agents speak when they have results.

**Cross-team (coordinators group):**
- Coordinators flag dependencies and conflicts in a structured format: `[From → To] Need/Issue + detail`.
- This gives the owner one place to see all cross-project coordination.

**Escalation path:** sub-agent → coordinator (internal) → human (via group). Sub-agents never contact the owner directly.

**Message formats (keep them structured):**

```
STATUS: [Team/Role] one-line result
- Key facts (2-4 bullets)
- Blockers: none | <blocker>
```

```
HANDOFF: [From → To] task
- Context
- Action + deadline
```

```
ESCALATION: [Coordinator] decision needed
- Situation (1-2 lines)
- Options: A / B / C with tradeoffs
- Recommendation + deadline
```

**Volume control:**
- Coordinators: a small daily cap of group messages (summaries, not play-by-play).
- Internal chatter unlimited — it never touches the messaging platform.
- Urgent items carry a consistent marker (e.g., an emoji or `URGENT` tag) so the owner can filter.

### 2.6 Business vs. Personal Separation

- Separate workspaces, agents, and machines for business and personal life. They must never share a gateway or a physical host — use separate machines or separately hosted instances with zero shared credentials, per the Critical row of the isolation table in §0. Separate gateway processes on the same host are NOT sufficient for this trust boundary.
- An agent serving personal/family matters must not have access to business repos, keys, or wallets, and vice versa.
- Put this scope boundary in the agent's instruction files explicitly ("this agent handles X only").

---

## 3. Layer 3 — Fleet: Multiple Machines and Hosted Deployments

### 3.1 When to Add a Second Machine

Add hardware separation when any of these is true:
- Different trust domains (business vs. family vs. client work).
- You serve other people (hosted agents for users) — that requires the Critical row of the isolation table (§0): a separate machine or separately hosted instance with zero shared credentials. Strict container isolation on a shared machine is acceptable only for Medium-risk work, not for untrusted hosted users.
- Workload contention (heavy local inference vs. always-on coordination).

### 3.2 Trust-Domain Isolation Rules

- One deployment per trust domain. No shared credentials, no shared file sync between domains.
- Backups flow one direction (e.g., business machine → backup target on the personal machine), never both. Mechanism: push-only access (the source holds credentials to write; the target holds none to read back), encrypted at rest per §1.5, verified by restoring once on the target side.
- Verify isolation with a checklist after setup. Minimum checklist:
  - [ ] No business credentials present on the personal machine (search keychain, config dirs, shell histories).
  - [ ] No personal credentials present on the business machine.
  - [ ] No file-sync job crosses the boundary in either direction.
  - [ ] Network exposure: neither agent can reach the other's gateway/admin ports (test from both sides).
  - [ ] Backup restore drill completed once per direction.

### 3.3 Hosted Agents for Other People (if applicable)

- **Warm buffer pattern:** pre-provision containers in a warm pool for instant assignment — each in its own user namespace with seccomp/AppArmor (or equivalent), read-only root filesystem where possible, no shared host mounts beyond explicit per-user volumes, and strict network egress controls; cold-deploy as fallback.
- **Ownership:** assign containers to users at claim time; use opaque, non-identifying internal identifiers (never raw user identifiers in infrastructure labels).
- **Budget metering:** meter model usage server-side, per user, with warning and cutoff thresholds. Tier differences should be budget differences, not model rosters, wherever possible.
- **Deletion and recovery:** soft-delete with a defined grace window and a documented restore path.
- **Payment lifecycle:** subscription status drives access. Failed payment → immediate, reversible revocation.
- **Multi-tenant gateway concerns:** each user's agent gets its own config root, workspace, and session store — never a shared session space. Per-tenant audit logging (which user triggered which action). Revocation must propagate to the gateway layer immediately (access checks at request time, not cached indefinitely). The human-gate pending ledger is per-tenant — one user's approvals can never touch another user's actions. Treat cross-tenant leakage as a P0 incident class. Concretely, per tenant: the pending-actions ledger, run ledger, and recall store are namespaced under that tenant's data directory (never one global file); approval UIs/channels are bound to the tenant's identity so an approval can only match a row in that tenant's ledger; and revocation propagates to in-flight gates — a revoked tenant's pending rows are expired at revocation time, and any tool gate re-checks tenant status on each execution attempt, not just at provisioning. For the gateway itself: one config root and plugin set per tenant (no shared plugin directories), session stores namespaced per tenant, log streams tagged with the tenant id and shipped separately, and per-tenant rate-limit buckets at the gateway/proxy layer (one tenant's runaway loop must not consume another's quota). **Model-level isolation:** tenants sharing one model provider or local inference backend share a context-leak surface — provider-side retention applies to everyone's content, and a shared self-hosted backend can leak context across requests if misconfigured. For untrusted tenants, prefer per-tenant model deployments or providers with contractual isolation; at minimum, document the shared-backend trust assumption to every tenant.

### 3.4 Operational Discipline at Fleet Scale

- Golden-version tracking: record the known-good combination of image + services + configuration for every deployed component, with a tested rollback path.
- Deterministic scripts over agent judgment for destructive or high-stakes operations (deletions, closings, migrations). The LLM proposes; a script executes; a human approves.
- Health checks across every external dependency on a fixed schedule, with alerting on failure.
- Incident playbook: classification tiers, containment steps per incident class, postmortem template, and retroactive procedure updates.

---

## 4. Engineering Process (applies at every layer)

When you change the agent's own code, configuration, or procedures:

**Scope note — minimum viable review (solo users, read this first):** for a personal agent, the review loop is: (1) state the problem and rollback point, (2) implement, (3) one self-review pass by any strong reasoning model with the blocking/non-blocking classification below, (4) run the baseline test, (5) secrets/PII scan, (6) document in the same change. That is the whole loop. The cross-model audit and bounded iteration machinery below earn their cost only when mistakes cost real money, data, or reputation — team agents, hosted users, anything touching funds.

**Full process (team/high-stakes):**

1. **Triage:** trivial change (config value, version pin) → fast path. Feature or structural change → full path.
2. **Plan:** state the problem, the deployment surface affected, and regression risk before coding.
3. **Implement + review loop:** use a strong reviewer model with a bounded loop (max 3 iterations, time-boxed). Classify findings into blocking vs. non-blocking for YOUR project (a common split: Correctness and Security block; Style and preference do not). Stop at zero blocking findings.
4. **Cross-model audit:** a second, different model reviews the work. Independence matters: the reviewer must not be the same agent/model that produced the change — during early hardening, prefer a different provider or a local model you trust. Different models catch different classes of error.
5. **Test:** no new failures versus a recorded baseline.
6. **Secrets/PII scan:** before every deploy and push.
7. **Document:** update the relevant files in the same change, not later.
8. **Release + verify:** tagged release, then verification against a checklist in the target environment.
9. **Record:** append to a shipped-work ledger (what shipped, when, evidence). Provenance prevents "did we ship this?" confusion later.

**Instruction-file changes (SOUL/AGENTS/USER and memory format):** treat these like schema migrations. Version the change (a dated entry in the file's own header or a git commit), state what behavior should change, and verify with a test conversation after the change ("what are you, what won't you do?"). When the memory format itself changes, migrate forward with a script that converts old daily files — never leave the agent to interpret two formats at once. Roll back by restoring the prior version from the workspace git history.

**Bounded review loop rules (the anti-stuck protocol):**
- Cap iterations (3) and time-box each round (e.g., 20 minutes).
- Blocking = Correctness or Security. Style is logged, never chased to zero.
- If not clean by the cap, escalate to the human with the classified findings and a recommendation.
- Log per-round tallies (C/S/Y counts) as provenance.

---

## 5. Setup Checklist

**Implementation order (dependency-correct):** the glue components have real dependencies — build in this sequence to avoid circular debugging debt: (1) locking wrapper (everything else needs it) → (2) ledger helpers (`read_jsonl`/`append_row`, §Appendix D) → (3) run ledger + digest job (observability first, so later steps are visible) → (4) human gate: pending ledger → `gate_check` → approval handler → privileged viewer → (5) memory: search tool → verification → promotion job → (6) drills: restore smoke test → full drill. Each step ends with a working, verifiable state.

**Day-1 quick start:** this template pairs with the official docs by design — the docs give the commands, this template gives the architecture and discipline. The fastest first session: follow the docs' "getting started" flow to a running agent with web chat (Step 0 and the first four checklist items below), then return here for the security baseline and hardening path. That single docs session is the only point where you need live CLI syntax before the agent exists.

**Host tooling prerequisites:** `git`, a secret store (OS keychain or equivalent), `cron` (or the gateway scheduler), `flock` or equivalent file locking, and optionally `ripgrep` for the v0 memory search. systemd (Linux) or launchd (macOS) for gateway supervision. All are either preinstalled or single-command installs on the common platforms.

The checklist names what to do; the section references tell you how. For gateway commands, consult the docs site (linked in the header) for the current CLI syntax — command forms change between releases, and this template stays version-free by design.

### Layer 1 (core) — Day 1 items unless marked Week-2 hardening
- [ ] **Step 0 — choose your isolation level** from the table in §0 before creating anything. This decision (what trust domains will this agent touch?) drives every later choice: machine layout, credential separation, gateway boundaries. Write the decision down in AGENTS.md.
- [ ] Install the gateway (package install per docs; verify with its version/health command).
- [ ] Create one agent with its own workspace (gateway agent-add command).
- [ ] Write SOUL/AGENTS/USER/IDENTITY files. Keep them terse and imperative. Starter skeletons: Appendix A.
- [ ] Configure primary model + fallback chain in the gateway provider/model config (docs: model providers).
- [ ] Enable memory files (daily files + curated MEMORY.md). Semantic search is an optional enhancement once memory grows (§1.3).
- [ ] Secrets into keychain/secrets manager using the injection pattern in §1.5. Verify none appear in workspace files or command output.
- [ ] Configure one channel (web chat first — it needs no external accounts); test round-trip.
- [ ] Day 1: minimal outbound gate — ask-in-channel with a simple pending-actions file and the rule "timeout = deny." Test: agent requests an action, you deny it, nothing executes.
- [ ] Week-2 hardening: upgrade to the durable pending-actions ledger with payload hashing, two-phase status, and `gate_check` in the executing tools (§1.8 full pattern). Test both approve and deny paths end to end.
- [ ] Add one heartbeat with 2–3 batched checks; wire the run ledger + daily digest + consecutive-error alert from §1.7.
- [ ] Week-2 hardening (supervision): run the gateway under a process supervisor (systemd `Restart=on-failure` / launchd `KeepAlive`, shape in §1.7) and verify it comes back after a kill and after reboot; wire the "digest succeeded within 24h" check from §1.7 into the alert path.
- [ ] Create the `data/` directory outside version control (recall store, pending-actions ledger, run ledger, vector index live here — see Appendix C); add `.gitignore` coverage for it in the same step.
- [ ] Baseline memory verification (Day 1, even before semantic search): after wiring the v0 search tool (plain ripgrep-based is fine), register it, then VERIFY: ask the agent to search memory and confirm it answers from real files. Registration without this verification step does not count (§1.3).
- [ ] (When semantic search is added) create `tools/` and `scripts/` directories for the embedding-based search + promotion script; re-verify registration as above.
- [ ] Week-2 hardening: git-init the workspace, install and configure the pre-push secret scan (`gitleaks` or equivalent — blocks commits containing high-entropy strings or known credential patterns; §1.5), configure the encrypted off-machine backup, and run one restore drill (§1.5).
- [ ] (When memory search is added) wire the weekly promotion job (§1.3).

### Layer 2 (team)
- [ ] Add second agent with its own workspace and role files (gateway multi-agent config; docs: agents).
- [ ] Configure bindings for channel routing (§2.3 example; docs: bindings/routing).
- [ ] Create team groups; write the communication protocol into coordinator instructions (§2.5 formats).
- [ ] Test: coordinator summary post; sub-agent deliverable post; escalation format.

### Layer 3 (fleet)
- [ ] Second machine for the second trust domain; isolation verified by checklist.
- [ ] One-directional backup configured and tested.
- [ ] If hosting for others: warm pool, opaque identifiers, metering, revocation, restore path (§3.3).
- [ ] Golden-version tracking + rollback test performed before first production user (§3.4).

---

## 6. Common Failure Modes (learned the hard way)

| Failure | Symptom | Prevention |
|---|---|---|
| Provider rejects a sampling param | Scheduled jobs fail silently for weeks | Pin models per job; disable unsupported params per provider; alert on consecutive errors |
| Secret in tool output | Credential persisted into session transcript | Mask tool results; never print raw secrets; subshell retrieval only |
| Unqualified references after hardening | RPC/lookup fails on every call after a security change | Ship hardening + body changes together; live-probe behavior, not just config |
| Agent marks work complete without doing it | False completion; drifted tracking | Require evidence artifacts (reports/files) for completion claims; audit both directions |
| Cron output silently dropped | Jobs run but nobody sees results | Explicit delivery destination on every job |
| Stale hard-coded dates in tests | Tests age out and fail | Relative timestamps in time-window tests |
| Bulk provisioning exceeds platform limits | Partial provisioning, stuck states | Batch within platform caps; verify provider acknowledgments between batches |
| Hard-coded port/image/env in scripts | Works on one host, breaks on another | Read environment values from config/secrets, never bake defaults that defeat them |
| Memory poisoning via retrieved content | Agent repeats injected instructions it once summarized into memory | Search tool returns snippets as quoted data only; agent re-reads cited region before acting; deterministic promotion is the only automated writer to MEMORY.md; audit promoted lines periodically |
| Local model OOM/degradation | Fallback chain silently degrades or crashes on GPU/CPU exhaustion | Health-check local models like providers; log OOM/reload events to the run ledger; keep context length within RAM budget; treat model reload latency as expected behavior in timeouts |
| Local embedding/index service down | Memory search returns nothing; promotion starves | Daily digest checks the index is queryable; alert when search returns zero hits for known-good queries |

---

## 7. References

**Adapting to other runtimes:** the concepts, isolation table (§0), human-gate patterns (§1.8), secret-handling rules (§1.5), and review discipline (§4) transfer as-is. The concrete artifacts — file names, directory layout, checklist commands, and the JSON shapes — are OpenClaw-specific and need real translation work on another framework (persisted state stores instead of markdown files, your framework's tool-registration API instead of gateway plugins, its scheduler instead of cron semantics). Budget that translation effort explicitly; do not assume the checklist ports line-for-line.

Mapping guidance if you use a different agent framework: SOUL.md → system persona prompt, IDENTITY.md → name/avatar/self-description, AGENTS.md → operating rules in the system prompt, USER.md → user-profile state, MEMORY.md + daily files → long-term state store, gateway bindings → your framework's routing/session config. The isolation table (§0), human-gate patterns (§1.8), and review discipline (§4) apply unchanged.

**Secret-store portability:** the examples use `secret-tool` (Linux/libsecret). Equivalents: macOS `security` CLI (`security find-generic-password -s <svc> -w`), 1Password CLI (`op read`), HashiCorp Vault, or `pass`. The pattern is the same — read at execution time, never in argv, never in visible output.

- Minimal agent loop pattern: single gateway + workspace instruction files + memory + cron (Layer 1).
- Multi-agent coordination: coordinator/sub-agent pattern with structured group messaging (Layer 2).
- Fleet operations: warm pools, metering, golden versions, deterministic execution for destructive ops (Layer 3).

## Appendix A — Starter Workspace Skeletons

Minimal working examples for the two most important instruction files, plus the supporting three. Copy, then edit for your own role. Keep each file under ~2 KB.

**`SOUL.md`**
```markdown
# SOUL.md - Who You Are

- **Name:** <agent name>
- **Role:** <one line: what this agent does and for whom>
- **Vibe:** <tone in 3-5 words>

## Core Rules
- Tell the truth. Never fabricate.
- <Domain boundary: what this agent does NOT handle>
- Ask before any action that leaves this machine (sends, posts, payments).
- Keep secrets out of all files and chat. Reference the keychain instead.

## Voice
- Short declarative sentences. No filler.
```

**`AGENTS.md`**
```markdown
# AGENTS.md - Operating Rules

## Every Session
1. Read SOUL.md and USER.md.
2. Search memory for recent context before acting.
3. Read today's memory file (memory/YYYY-MM-DD.md).

## Memory
- Log decisions and outcomes to memory/YYYY-MM-DD.md in the same session.
- Anything needed tomorrow gets written to a file today.

## Safety
- trash over rm. Ask before destructive commands.
- No personal data in any file that syncs or commits.
- Secrets live in the system keychain. Never print them.

## Tools
- <environment-specific notes: hosts, device names, preferred voices>
```

**Verification for a new agent:** start a session, ask it what it is and what it will not do. Correct answers from the files above mean the instruction layer is wired.

**`USER.md`**
```markdown
# USER.md - About Your Human

- **Preferred address:** <e.g., "chief" or "partner"> — relational terms or neutral identifiers only; never real names, even common first names (see §1.5 PII rule)
- **Timezone:** <tz>
- **Working style:** <e.g., direct, expects agent to research before asking>

## Preferences
- <communication channels and when to use each>
- <model or tool preferences, if any>

## Scope Boundary
- <what this agent handles and explicitly does not handle>
```

**`IDENTITY.md`**
```markdown
# IDENTITY.md

- **Name:** <agent name>
- **Creature:** AI agent
- **Vibe:** <tone>
- **Emoji:** <one emoji>
```

**`MEMORY.md`**
```markdown
# MEMORY.md - Long-Term Memory

<!-- Curated index only: decisions, durable facts, lessons. -->
<!-- Load ONLY in private, direct conversations. Never in groups. -->

## Active Work
- <current project + state>

## Decisions
- <dated decision + why>

## Lessons
- <what went wrong and the rule that prevents it>
```

> **Privacy note:** `MEMORY.md` is loaded into every private conversation. Keep only what is needed for the agent to work well. Never store credentials, addresses, or financial details there.

## Appendix B — Starter Artifact Templates

Minimal templates for the three artifacts the engineering process (§4) produces. Copy into your workspace and adapt.

**Plan (step 2 of §4)** — `plans/YYYY-MM-DD-<slug>.md`
```markdown
# Plan: <one-line title>
Date: YYYY-MM-DD   Status: draft | in-progress | done | aborted

## Problem
<2-3 sentences: what is wrong or missing, and why now>

## Surface
<files/services/config affected; deployment surface; blast radius>

## Approach
<numbered steps; rollback point after each irreversible step>

## Regression Risk
<what existing behavior could break; how the baseline will catch it>

## Verification
<concrete checks that prove done — commands and expected output>
```

**Complete Day-1 approval loop (all four pieces together):**

```text
# 1. Agent proposes an action (before posting the request):
payload   = build_exact_final_payload(action)           # recipient, content, amount
payload_hash = sha256(serialize(payload))
store_encrypted("data/payloads/act-001.enc", payload)   # keychain-held key
append_row("data/pending-actions.jsonl",
           {"id": "act-001", "ts": now(), "status": "pending",
            "action": "send-message", "payload_hash": payload_hash,
            "expires": now() + 24h})

# 2. Agent posts ONLY the neutral reference to the approval channel:
"Action act-001 requires approval; reply approve act-001 or deny act-001."

# 3. Privileged viewer (human side; CLI or tiny local HTML page):
payload = decrypt("data/payloads/act-001.enc")          # keychain-held key
display(payload)                                        # human reads the EXACT payload
record_decision("act-001", "approved")                  # append status row

# 4. Executing tool, before acting (inside the tool, not the agent):
gate_check(action_type, payload_hash)                   # pseudocode in §1.8
#   verifies: approved row exists, hash matches, not expired,
#             not already executed/executing — then flips to "executing",
#             performs the side effect, appends "executed".
```

**Pending-actions ledger (§1.8 pattern 2)** — `pending-actions.jsonl`, one record per proposed action:
```json
{"id": "act-20261119-001", "ts": "2026-11-19T10:00:00Z", "status": "pending", "action": "send-message", "payload_hash": "9f86d081884c7d659a2f...", "expires": "2026-11-20T10:00:00Z"}
```
The row stores only the payload HASH, never the payload itself. The full payload lives in a durable encrypted store — `data/payloads/<id>.enc`, encrypted at rest (age/openssl with a keychain-held key, or the OS keychain directly) — NOT in session context alone, because a restart must not break the ability to show the human the exact final payload or to re-verify the hash at execution time. Write the encrypted payload BEFORE posting the approval request. States: `pending → approved | denied | expired`. The agent writes the row (and the encrypted payload) before posting the request; the human's approval references the id; execution happens only against a row whose status is `approved` and whose payload hash matches.

**Decryption flow (approval display):** the approval channel or UI decrypts `data/payloads/<id>.enc` using the same keychain-held key the execution tool uses, shows the plaintext to the human, then immediately discards the decrypted bytes. The decrypted payload never enters the agent's context, the session transcript, or any model call. Append-only — never edit a row, supersede it.

Edge cases to handle on Day 1:
- **Orphaned rows:** a gateway restart between request and approval must not lose the row — that is why state is a durable file, not memory. On startup, sweep rows older than their `expires` and mark them `expired`.
- **Idempotent execution:** the executing tool appends a new JSONL record with the same `id`, `status: "executed"`, an `executed_at` timestamp, and the same payload hash — so a retry can never double-send. Execution tools must query for an existing `executed` record for that id before acting. Duplicate approval replies for the same id are no-ops.
- **Schema evolution:** add new fields, never rename old ones. Readers of old rows are the tools themselves; unknown fields are ignored, so additions are safe.

**Shipped-work ledger entry** — one row per shipped change, append-only:
```markdown
| Date | Item | Evidence | Notes |
|---|---|---|---|
| YYYY-MM-DD | <what shipped> | <commit/tag/report path/test result> | <rollback point or caveat> |
```

**Privileged Payload Viewer (starter — the §1.8 human-side component):**

```python
#!/usr/bin/env python3
# view-payload.py <action-id>  — decrypts and shows the EXACT payload, then discards it.
# Runs OUTSIDE the agent's context. The plaintext is never written to disk or logged.
import subprocess, sys, json, base64
# Linux: swap the `security` call for `secret-tool lookup service approval-payload-key`

ACTION_ID = sys.argv[1]
ENC_PATH  = f"data/payloads/{ACTION_ID}.enc"

# key never enters argv output or logs — captured directly into the decrypt call
key = subprocess.run(
    ["security", "find-generic-password", "-s", "approval-payload-key", "-w"],
    capture_output=True, text=True, check=True
).stdout.strip()

with open(ENC_PATH, "rb") as f:
    blob = f.read()

plaintext = aes256gcm_decrypt(blob, key)   # AES-256-GCM (cryptography lib): random
                                           # 12-byte nonce prefixed to the blob, key from
                                           # keychain; verify auth tag before parsing
payload = json.loads(plaintext)
show_payload_safely(payload)  # MUST NOT write to stdout, logs, files, or any
# persistent storage — the terminal is NOT a safe display channel (stdout can
# land in shell history or logs). Use a GUI dialog (zenity, kdialog, tkinter)
# or a local HTML page opened in a browser that renders and discards the
# payload. The plaintext exists only inside the viewer process's memory.
# plaintext intentionally NOT saved anywhere; process exit discards it
```
Then the human records the decision back through the approval channel (append the `approved`/`denied` row). The viewer never runs inside the agent, never sends the payload to any model, and never persists the decrypted bytes. **Separation of privilege:** the payload decryption key must be accessible only to the human and the execution tools (same UID) — never to the gateway process if it runs under a different user or with different privileges.

**Approval handler (the human-reply side of the gate — parallel to `gate_check`):**

```text
def handle_approval_reply(message_text, author):
    # 1. Parse the reply: must reference an action id and a decision word.
    intent, action_id = parse_reply(message_text)   # "approve act-001" / "deny act-001"
    if not intent or not action_id:
        return refuse_ambiguous()        # vague "sure" never approves anything

    # 2. Verify the author is an authorized approver (allowlist, not identity-guessing).
    if author not in APPROVERS:
        return refuse_unauthorized(action_id)

    with locked("data/pending-actions.jsonl"):          # same lock discipline as gate_check
        rows = read_jsonl("data/pending-actions.jsonl")
        row = latest_row(rows, id=action_id)

        if not row or row.status != "pending":
            return no_op()                              # expired/denied/executed — reply is a no-op
        if now() > row.expires:
            append_row(rows, {"id": action_id, "status": "expired"})
            return notify_expired()

        append_row(rows, {"id": action_id, "status": intent})   # "approved" or "denied"

    notify(f"{action_id} {intent} by {author}")
    # Execution does NOT happen here — the executing tool's gate_check picks it up.
```
Common bugs this closes: vague replies approving the wrong action, unauthorized approvers, double-processing of one reply, and approving an expired row.

**Minimum Viable Restore Test (the §1.5 drill, as a one-page procedure):**
```text
1. cp -r <backup> /tmp/restore-test/            # decrypt if encrypted
2. Point a test gateway at the restored workspace (separate port/config)
3. Run the four checks:
   a. memory:    ask the test agent "what did you work on lately?" — answers from real files
   b. tools:     custom tools appear in the tool list / register without error
   c. gate:      data/pending-actions.jsonl intact (row count > 0, parseable)
   d. cron:      one scheduled job fires and delivers to the right target
   e. human gate: create a test pending row, approve via the privileged viewer,
                 confirm the tool performs the side-effect; then a second row
                 with NO approval — confirm the tool refuses it ("no approved
                 row" / "payload mismatch"). The gate is TCB; a restore that
                 breaks it silently is worse than a restore that fails loudly.
4. Tear down the test gateway. Record the run (date + result) in the shipped ledger.
```
A drill that skips step 3's checks is not a drill — a restored file tree that nothing verified is the exact failure the drill exists to catch. **Lighter variant for personal agents (no second gateway):** restore to a scratch directory, run the search tool and the promotion script against the restored copy directly (both are plain scripts — they do not need a live gateway), and parse the restored pending ledger. That covers memory + ledger integrity without the gateway spin-up; reserve the full drill (with the test gateway and cron check) for business-critical agents and before any restore-actually-needed event.

**Postmortem** — `postmortems/YYYY-MM-DD-<incident>.md`
```markdown
# Postmortem: <one-line incident>
Date: YYYY-MM-DD   Severity: P0|P1|P2|P3|INFO   Duration: <detect -> contain -> fix>

## What Happened
<factual timeline; no blame>

## Impact
<what was affected, quantified where possible>

## Root Cause
<the chain, not the proximate trigger>

## What Worked / What Didn't
<detection, containment, tooling>

## Remediation
- [ ] <fix applied, with evidence>
- [ ] <hardening to prevent the class>

## Procedure Updates
<which SOP/runbook changed as a result>
```

## Appendix C — Workspace Directory Layout

The canonical layout every section above assumes:

```
~/.openclaw/                       # agent root (per agent)
├── workspace/                     # the agent's brain — version-controlled with git
│   ├── SOUL.md                    # persona
│   ├── AGENTS.md                  # operating rules
│   ├── USER.md                    # user profile (pseudonym/initials only)
│   ├── IDENTITY.md                # name/avatar
│   ├── MEMORY.md                  # curated long-term index (privacy: load private-only)
│   ├── memory/                    # daily files: YYYY-MM-DD.md (append-only)
│   ├── tools/                     # custom tools registered with the gateway
│   ├── scripts/                   # cron-invoked deterministic scripts (promotion, etc.)
│   └── plans/ · postmortems/      # §4 artifacts
├── data/                          # OUTSIDE version control — sensitive runtime state
│   ├── .recall.jsonl              # memory search telemetry (may contain PII — .gitignore'd)
│   ├── pending-actions.jsonl      # human-gate pending state (durable, append-only; .gitignore'd — sensitive action metadata)
│   ├── run-ledger.jsonl           # scheduled-job outcomes; feeds alert rules + daily digest (§1.7)
│   ├── payloads/                  # encrypted full payloads for pending actions (§1.8/Appendix B)
│   └── logs/ → ../logs/           # structured logs live beside data/ (same PII rules)
│   └── index/                     # vector index (rebuildable; location is a performance choice)
└── agents/<id>/                   # gateway-managed: sessions, state (never hand-edit)
```

Rules: `workspace/` is git-versioned and backed up encrypted (§1.5). `data/` is excluded from version control, backed up encrypted, never committed. `agents/` is gateway-managed runtime state — restore from backups, do not edit by hand.

**Multi-agent note (Layer 2):** with several agents on one gateway, each agent gets its own root — `~/.openclaw-work/`, `~/.openclaw-personal/`, each with the same `workspace/` + `data/` shape above and zero shared credentials (§2). The gateway configuration itself lives in its own non-version-controlled location (§1.9 rules apply to it).

## Appendix D — Helper Function Signatures

The pseudocode in §1.3, §1.7, and §1.8 relies on small helpers. Signatures (stdlib-only implementations):

| Helper | Signature | Behavior |
|---|---|---|
| `read_jsonl(path)` | `-> list[dict]` | Read all lines; skip blanks and malformed lines (log them). |
| `append_row(path, row)` | `-> None` | Acquire file lock; one atomic append (`open(append)`, single write, flush). |
| `aggregate(events, by, within_days)` | `-> list[dict]` | Group by key fields over the rolling window; emit `{key..., hits: int, avg_score: float}`. |
| `append_once(path, line)` | `-> bool` | Lock, scan for an existing identical pointer line, append only if absent; returns whether it appended. |
| `mark_promoted(file, line_range)` | `-> None` | Append a `promoted` marker row to the recall store so the group is not re-considered. |
| `rotate(path, keep_days)` | `-> None` | Move records older than the window to an archive file; keep the live file small. |
| `distill(item)` | `-> str` | Return the one-line summary: plain first-line excerpt, or a schema-constrained LLM call. |
| `latest_row(rows, id=..., status=...)` | `-> dict or None` | Most recent row matching id + status. |
| `exists_row(rows, id=..., status=...)` | `-> bool` | Any row matching id + status. |
| `last_records_per_job(path, n)` | `-> dict[str, list]` | Map job_id → last n records. |
| `count_trailing(records, outcome)` | `-> int` | Length of the trailing run of matching outcomes. |
| `now()` / `expires` helpers | ISO-8601 strings | Timestamps in UTC; expiry comparison in the same clock. |
| `balances()` | `-> list[(name, value, warn_at)]` | One provider call or local command per metered resource; returns current value + warn threshold. |
| `disk_free_pct()` / `mem_status()` | `-> str` | `df -h` / host memory metrics, formatted. |
| `supervisor_status()` | `-> str` | `systemctl is-active <unit>` / `launchctl list` check; also `systemctl --failed`. |
| `search_returns_hits()` | `-> bool` | Run one known-good query through the memory search tool; false = index/embedding problem. |
| `pending_rows()` / `hours_left(row)` | `-> list[dict]` / `-> float` | Read pending gate rows; compute hours until expiry. |
| `last_digest_age_h()` | `-> float` | Hours since the last successful digest row in the run ledger. |
| `log_job_outcome(job_id, outcome, detail)` | `-> None` | Locked append of one run-ledger record (the promotion script's reporting path). |

Each helper is 5–20 lines against the schemas already given. All writes go through `append_row`-style locking (§1.10 concurrency rules).

## Revision History

| Change | Review |
|---|---|
| Initial release: template extracted from production setup notes; public-safe rewrite. | Reviewed by an independent reasoning model before release; see note below |

*Review provenance:* every release of this template passes an independent model audit with zero Correctness/Security/Completeness findings. Audit transcripts are kept out of this file on purpose — they reference the internal environment.
