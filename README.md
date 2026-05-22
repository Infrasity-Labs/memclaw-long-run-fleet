
![Hero Image](/docs/images/memclaw_longrun_fleet_hero.svg "a title")

# MemClaw Long-Run Research Fleet

MemClaw gives long-running agent fleets governed, shared, self-improving memory.

This repo shows MemClaw managing shared context across **three OpenClaw agents** operating continuously over a simulated 14-day window, with automatic contradiction detection, status-governed recall, and zero manual memory cleanup.

---

[OpenClaw](#what-is-openclaw) · [MemClaw](#what-is-memclaw) · [The Problem](#the-problem) · [How It Works](#how-memclaw-resolves-contradictions) · [Architecture](#architecture) · [Quickstart](#quickstart) · [Screenshot Moments](#the-three-screenshot-moments) · [Docs](#related)

---

## Three agents. One memory pool. Zero stale context.

The Sourcing Agent scrapes. The Verification Agent confirms. The Synthesis Agent briefs.

On Day 9, the competitor raises their price from **$299 to $349**. By Day 10, the Synthesis Agent's brief surfaces only `$349`, because MemClaw automatically marked every `$299` memory as `outdated` the moment the new fact arrived.

---

## What is OpenClaw

OpenClaw is an open-source agent orchestration gateway. It runs locally as a daemon, registers named agents from workspace directories, and exposes them through a unified chat interface and API. Each agent has its own workspace, a directory containing identity files (`SOUL.md`, `AGENTS.md`) injected as system context at session start, along with its own plugin bindings (MCP servers, memory backends, tools).

In this repo, OpenClaw is doing three things:

- **Routing**: `/agent sourcing-agent` targets a specific registered agent
- **Context injection**: loads each agent's `SOUL.md` and `AGENTS.md` before the first message
- **Plugin wiring**: registers the MemClaw MCP server so agents call `memclaw_*` tools natively as tool calls

```bash
npm install -g openclaw@latest
```

---

## What is MemClaw

MemClaw is open-source multi-agent memory for AI agent fleets: governed, shared, and self-improving. Agents write plain text. MemClaw turns it into searchable, governed, structured memory with automatic enrichment, lifecycle management, and contradiction detection.

The core loop: **write → enrich → govern → recall → compound.** Every interaction makes the next one smarter.

What makes MemClaw different from a vector database:

| Capability | What it means |
|---|---|
| **LLM enrichment on write** | Every `memclaw_write` auto-classifies type, generates title/summary/tags, extracts entities, detects contradictions from a single `content` field |
| **8-status lifecycle** | Memories move through `active`, `pending`, `confirmed`, `outdated`, `conflicted`, `archived`, `deleted` statuses automatically |
| **Contradiction detection** | New facts that contradict existing memories trigger automatic supersession, old memories marked `outdated`, new memory promoted |
| **Crystallizer** | LLM batch process that merges near-duplicate memories into canonical atomic facts with full provenance |
| **Hybrid recall** | `memclaw_brief` combines vector similarity, keyword search, and status filters in one call |
| **Audit trail** | Every read and write logged "which agent wrote this and when" is always answerable |
| **Fleet isolation** | Memory partitioned by `fleet_id`. Every recall passes a `WHERE fleet_id IN (...)` predicate before search runs |

This repo is a use-case implementation: three OpenClaw agents operating against a single MemClaw fleet (`fleet-longrun-research`), showing what MemClaw's contradiction resolution and status governance look like in a continuous long-running deployment.

→ [MemClaw source (Apache 2.0)](https://github.com/caura-ai/caura-memclaw) · → [Documentation](https://memclaw.net/docs) · → [Managed cloud (free tier)](https://memclaw.net/pricing)

---

## The Problem

**Why not just use a standard vector database for a long-running agent fleet?**

In a standard vector store, if a fact changes in the real world, the new fact gets added *alongside* the old one. If your fleet runs every day for two weeks, by Day 9 the database is filled with contradictory entries. Without an active system to resolve contradictions, agents start pulling stale context and daily outputs become inconsistent.

| Approach | Problem |
|---|---|
| Raw vector store | New facts pile up next to old ones no resolution, no governance |
| Prompt-level filtering | "Ignore old pricing data" the stale vectors still pass through recall |
| Manual cleanup | Doesn't scale for fleets running continuously over weeks |

MemClaw resolves this at the **write layer**. When the Sourcing Agent writes `$349/month` on Day 9, MemClaw's contradiction detection pipeline compares the new fact against existing memories in the fleet, identifies the conflict, marks the eight `$299` memories as `outdated`, and promotes the new fact before any agent ever queries the pool.

---

## How MemClaw Resolves Contradictions

```
Sourcing Agent writes "$349/month" on Day 9
                        │
                        ▼
        MemClaw contradiction detection fires
        (new fact compared against fleet-longrun-research)
                        │
            ┌───────────┴──────────────┐
            │   8 × $299 memories      │
            │   status: active         │
            │   → auto-marked outdated │
            └───────────┬──────────────┘
                        │
                        ▼
        $349 memory: status confirmed
        Synthesis Agent Day 10 recall:
        → 1 result returned ($349)
        → 8 results suppressed (outdated)
```

This is not a prompt instruction. It is a write-time pipeline inside MemClaw's enrichment layer the contradiction check runs before the memory is committed, the supersession is stored as a database relationship, and the `status` field governs every future recall.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  OpenClaw Gateway (aisa/deepseek-v3)                                │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  sourcing-agent  │  │verification-agent│  │ synthesis-agent  │  │
│  │                  │  │                  │  │                  │  │
│  │  SOUL.md         │  │  SOUL.md         │  │  SOUL.md         │  │
│  │  AGENTS.md       │  │  AGENTS.md       │  │  AGENTS.md       │  │
│  │  governance skill│  │  governance skill│  │  governance skill│  │
│  │                  │  │                  │  │                  │  │
│  │  fleet:          │  │  fleet:          │  │  fleet:          │  │
│  │  fleet-longrun   │  │  fleet-longrun   │  │  fleet-longrun   │  │
│  │  -research       │  │  -research       │  │  -research       │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                     │            │
│           └─────────────────────┴─────────────────────┘            │
│                                 │  memclaw_* MCP tools             │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MemClaw (memclaw.net managed / self-hosted Docker)                 │
│                                                                     │
│  LLM enrichment on every write:                                     │
│  type · title · tags · entities · contradiction check               │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  fleet-longrun-research                                       │  │
│  │                                                               │  │
│  │  Days 1-8: 8 × $299/month   →  status: outdated (Day 9)      │  │
│  │  Day 9:    $349/month       →  status: confirmed             │  │
│  │  Day 10:   governed recall  →  $349 surfaces, $299 suppressed │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  pgvector store · 8-status lifecycle · crystallizer · audit trail   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Agent Roles

| Agent | Role | What it writes | What it reads |
|---|---|---|---|
| `sourcing-agent` | Raw data collection | Daily competitor pricing facts | Current fleet brief (status: active) |
| `verification-agent` | Cross-checking | Confirmation notes, conflict flags | All sourcing-agent memories |
| `synthesis-agent` | Intelligence briefs | Brief outcome metadata | Active/confirmed memories only |

All three agents share one fleet (`fleet-longrun-research`) and one governance skill (`memclaw-research-fleet.md`). The governance skill defines the `tenant_id`, `fleet_id`, and `agent_id` contract every agent must pass in every MemClaw tool call.

---

## MemClaw MCP Tools Used

| Tool | What it does in this repo |
|---|---|
| `memclaw_write` | Store pricing fact with auto-enrichment type, title, tags, contradiction check |
| `memclaw_search` | Search fleet pool by query, filter by status or agent_id |
| `memclaw_brief` | Governed recall returns only active/confirmed memories for a query |
| `memclaw_transition` | Manually move a memory's status (e.g. `active → confirmed`) |

---

## Repository Structure

```
memclaw-longrun-fleet/
├── .env.example
├── .gitignore
├── openclaw.json               ← Gateway config: model, agents, MCP server
├── README.md
├── agents/
│   ├── sourcing-agent/
│   │   ├── SOUL.md             ← Personality and hard limits
│   │   └── AGENTS.md           ← Fleet scope, daily workflow, tool call rules
│   ├── verification-agent/
│   │   ├── SOUL.md
│   │   └── AGENTS.md
│   └── synthesis-agent/
│       ├── SOUL.md
│       └── AGENTS.md
└── skills/
    └── memclaw-research-fleet.md   ← Shared skill: tenant_id, fleet_id, recall + write protocol
```

`SOUL.md` is injected first on every session and defines who the agent is.  
`AGENTS.md` is injected second and defines what the agent does, which fleet it writes to, and how it uses MemClaw tools.  
`memclaw-research-fleet.md` is a shared skill copied into every agent workspace. Update once, redeploy to all agents.

---

## Prerequisites

- Node.js 18+
- OpenClaw CLI: `npm install -g openclaw@latest`
- A MemClaw account: [memclaw.net](https://memclaw.net) (free tier available) or self-hosted via Docker
- An AISA API key from [aisa.one](https://aisa.one) (used to access DeepSeek-v3)

---

## Quickstart

### 1. Clone

```bash
git clone https://github.com/Shushant-Priyadarshi/memclaw-longrun-fleet.git
cd memclaw-longrun-fleet
```

### 2. Configure environment

```bash
cp .env.example .env
```

Fill in your keys:

```bash
# ── LLM_GATEWAY_API_KEY───────────────────────────────────────────────
LLM_GATEWAY_AP-_KEY=you-llm-gateway-key

# ── MemClaw (managed) ────────────────────────────────────────────────
MEMCLAW_API_KEY=mc_your-memclaw-key-here
MEMCLAW_TENANT_ID=your-tenant-id
MEMCLAW_FLEET_ID=fleet-longrun-research
```

For self-hosted MemClaw via Docker:

```bash
docker run -d --name memclaw -p 8000:8000 ghcr.io/caura-ai/caura-memclaw:latest
# Then set MEMCLAW_API_URL=http://localhost:8000 and leave MEMCLAW_API_KEY blank
```

### 3. Create the MemClaw fleet

```bash
curl -X POST "https://memclaw.net/api/v1/fleet" \
  -H "X-API-Key: $MEMCLAW_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_id": "your-tenant-id",
    "fleet_id": "fleet-longrun-research",
    "name": "Long-Run Research Fleet"
  }'
```

### 4. Install the MemClaw plugin

```bash
export MEMCLAW_API_KEY="mc_your-key"

curl -sf "https://memclaw.net/api/install-plugin?fleet_id=fleet-longrun-research" \
  -H "X-API-Key: $MEMCLAW_API_KEY" | bash
```

### 5. Deploy agent workspaces

```bash
cp -r agents/sourcing-agent ~/.openclaw/workspace-sourcing-agent
cp -r agents/verification-agent ~/.openclaw/workspace-verification-agent
cp -r agents/synthesis-agent ~/.openclaw/workspace-synthesis-agent

for agent in sourcing-agent verification-agent synthesis-agent; do
  mkdir -p ~/.openclaw/workspace-$agent/skills
  cp skills/memclaw-research-fleet.md ~/.openclaw/workspace-$agent/skills/
done
```

### 6. Start the gateway

```bash
openclaw gateway restart
openclaw agents list        # verify all three agents are registered
openclaw dashboard          # http://127.0.0.1:18789
```

### 7. Confirm MemClaw is connected

```bash
openclaw doctor
# Expected: [memclaw] ContextEngine 'memclaw' registered
```

---

## Testing

### 📸 1: Parallel enrichment (Day 1)



Send to **sourcing-agent** in the dashboard:

```
Run your Day 1 data collection. Call memclaw_write with:
- tenant_id: "your-tenant-id"
- fleet_id: "fleet-longrun-research"
- agent_id: "sourcing-agent"
- content: "Competitor pricing page shows $299/month for the Pro plan as of Day 1."
- memory_type: "fact"
- visibility: "scope_team"

Then call memclaw_search with the same fleet_id and query: "competitor pricing".
Show the raw tool responses including the memory ID and auto-enriched metadata.
```


Immediately send to **verification-agent**:

```
Run your Day 1 verification pass. Call memclaw_search for what sourcing-agent wrote about
competitor pricing in fleet "fleet-longrun-research". Report memory IDs and statuses,
then call memclaw_transition on the most recent memory to set status: "confirmed".
```

![Day 1 Source Agent](/docs/images/day1_verify_agent_ask.png "a title")

![Day 1 Source Agent](/docs/images/day1_verify_agent_response.png "a title")

**What you'll see:** Both agents writing to and reading from the same pool simultaneously. MemClaw auto-generates a title, tags, and similarity score from the raw `content` string — the agent only sent text.

---

### 📸 2: The contradiction (Day 9)

After running Days 1–8 (eight writes of `$299/month`), send to **verification-agent**:

```
CRITICAL VERIFICATION TASK — Day 9.

A new pricing memory was just written by sourcing-agent claiming the competitor price
is now $349/month. This directly contradicts the confirmed $299/month memories from Days 1-8.

Call memclaw_search with fleet_id: "fleet-longrun-research", query: "competitor pricing", top_k: 10.
List every memory with its ID, status, and content.
Identify which memories are now outdated.
Call memclaw_transition to mark the old confirmed memory as "outdated".
Write a conflict resolution note to the fleet.
Show every tool call response.
```

![Day 1 Source Agent](/docs/images/contradiction_ask.png "a title")

![Day 1 Source Agent](/docs/images/contradiction_response.png "a title")

**What you'll see:** The verification agent finding both price points in the pool, explicitly transitioning the old memory to `outdated`, and writing a conflict resolution record. MemClaw's contradiction detection may have already auto-superseded the old memories on write — the agent's search will confirm it.

---

### 📸 3: Governed recall (Day 10)

Send to **synthesis-agent**:

```
Run your Day 10 daily intelligence brief.

Call memclaw_brief with:
- fleet_ids: ["fleet-longrun-research"]
- query: "competitor pricing current"
- agent_id: "synthesis-agent"
(no status_filter — MemClaw returns only active/confirmed by default)

Also call memclaw_search with status_filter: "outdated" to count suppressed memories.

Produce the brief in this format:
---
DAILY INTELLIGENCE BRIEF — Day 10
Generated by: synthesis-agent | Model: aisa/deepseek-v3

## Competitor Pricing (Governed Recall)
Current price: $[X]/month
Source memory IDs: [list]
Suppressed: [N] outdated memories from Days 1-8

## Summary
[2-3 sentences based only on active/confirmed memories]
---
```

![Day 1 Source Agent](/docs/images/day10_ask.png "a title")

![Day 1 Source Agent](/docs/images/day_10_verification.png "a title")


**What you'll see:** The brief surfaces `$349` as the current value and explicitly reports how many `$299` memories were suppressed proof that MemClaw's status governance, not prompt engineering, is filtering the stale context.

---

## Running the Full 14-Day Simulation

To fast-forward Days 2–8 without sending eight individual agent prompts, write directly via the MemClaw API:

```bash
for day in 2 3 4 5 6 7 8; do
  curl -s -X POST "https://memclaw.net/api/v1/memories" \
    -H "X-API-Key: $MEMCLAW_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{
      \"tenant_id\": \"$MEMCLAW_TENANT_ID\",
      \"fleet_id\": \"fleet-longrun-research\",
      \"agent_id\": \"sourcing-agent\",
      \"content\": \"Competitor pricing page shows \$299/month for the Pro plan as of Day $day.\",
      \"memory_type\": \"fact\",
      \"visibility\": \"scope_team\"
    }" | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'Day $day → {d.get(\"id\",\"err\")} | {d.get(\"status\")}')"
  sleep 1
done
```

Then trigger the Day 9 drift injection through the `sourcing-agent` in the dashboard to get the full agent-visible contradiction moment.

---

## Why This Matters for Production Fleets

A standard vector store accumulates contradictions silently. By week two of a daily intelligence fleet, the pool contains:

- 8 memories saying the price is `$299`
- 1 memory saying the price is `$349`
- No mechanism to tell the synthesis agent which one is true

Without status governance, the synthesis agent either hallucinates a consensus, gets confused by conflicting context, or outputs inconsistent briefs depending on which memories happen to rank highest on a given day.

MemClaw solves this structurally. The contradiction detection is a write-time pipeline, not a prompt instruction. The `status` field is a database column, not a hint. The Synthesis Agent's brief is clean because the data it received was clean — not because the prompt told it to ignore old memories.

---

## Security Notes

- Never commit `.env` it is in `.gitignore` by default
- Never commit `openclaw.json` with live API keys use `${ENV_VAR}` references as shown
- Rotate any key that appears in terminal logs, git history, or chat exports
- Use HTTPS for all `MEMCLAW_API_URL` values in hosted deployments
- If you need to purge secrets from git history: `git filter-repo --invert-paths --path .env` then force-push

---

## Related

- [MemClaw documentation](https://memclaw.net/docs)
- [MemClaw open source (Apache 2.0)](https://github.com/caura-ai/caura-memclaw)
- [OpenClaw agent workspace guide](https://docs.openclaw.ai)
- [MemClaw cross-fleet governance reference repo](https://github.com/Infrasity-Labs/memclaw-cross-fleet-gov)
- [AISA — multi-model gateway](https://aisa.one)

---

Built on [MemClaw](https://memclaw.net) open-source multi-agent memory for AI agent fleets. Governed, shared, self-improving.  
[Source (Apache 2.0)](https://github.com/caura-ai/caura-memclaw) · [Documentation](https://memclaw.net/docs) · [Managed cloud](https://memclaw.net/pricing)