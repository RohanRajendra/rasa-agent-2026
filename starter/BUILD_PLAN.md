# Build Plan — "Always-On AI Coworker" with Long-Term Memory

> A team coworker that **remembers 90 days of decisions** and **catches contradictions**
> before they ship. Built on the Rasa CALM starter; storage is JSON + Nebius (no SQLite,
> no vector DB). Four people, three checkpoints.

---

## 1. The one-paragraph pitch

Feed the agent your team's history (decisions, bugs, tasks). When someone proposes
something new, the agent semantically searches that memory and **flags conflicts with
past decisions** — e.g. *"Hold on, in March the team decided to deprecate legacy auth by
Q3; this contradicts that."* New meetings get ingested back into memory, so the loop
closes: **decisions in → contradictions caught → new decisions in.**

Maps to all four prize criteria: 🧠 Persistence, 🛡️ Resilience (LLM constrained to
retrieval), 🏢 Workflow fit, 🎙️ Voice (the starter already has the voice loop).

---

## 2. How it maps onto the starter

The starter's golden rule (from [actions/actions.py](actions/actions.py)):
**flows own the conversation logic; actions do the raw work and return slots for the flow
to branch on.** We follow it exactly.

| Our piece | Starter pattern we copy | Storage |
|---|---|---|
| Memory layer | [actions/tickets.py](actions/tickets.py) (JSON store) | `.data/memory.json` |
| Semantic search | — (new) | Nebius embeddings + cosine in `numpy` |
| `contradiction_check` flow | [data/flows/ticket_status.yml](data/flows/ticket_status.yml) (`if slots.X` branch) | — |
| `meeting_ingestion` flow | [data/flows/support_triage.yml](data/flows/support_triage.yml) (collect → action → utter) | — |
| LLM judgment / extraction | model groups in [endpoints.yml](endpoints.yml) | Nebius chat (Qwen, temp 0) |

**Why JSON + Nebius instead of SQLite + ChromaDB:** drops two new infra pieces (a SQL
schema and a vector DB + 80MB model download) and leans on the JSON store the starter
already ships plus the Nebius connection we're already wired into. De-risks the demo.

---

## 3. The shared contract (everyone codes against this)

This is the **single most important artifact** — Person A publishes it first so the
other three can work in parallel against stable signatures.

### Memory item schema (`.data/memory.json`, dict keyed by id)
```jsonc
{
  "MEM-0002": {
    "id": "MEM-0002",
    "kind": "decision",              // decision | bug | task
    "summary": "Deprecate legacy auth by Q3",
    "detail": "RFC-012 approved ...",
    "owner": "Marcus Chen",
    "status": "active",
    "decided_at": "2026-03-04",
    "deadline": "2026-09-30",
    "source": "seed",                // seed | meeting | manual
    "embedding": [0.01, -0.2, ...]   // added by memory.py at write time
  }
}
```

### `actions/memory.py` API
```python
def embed(text: str) -> list[float]              # one Nebius embeddings call
def add_item(kind, summary, **fields) -> str     # embeds + writes, returns new id
def search(query, k=5, kind=None) -> list[dict]  # cosine over stored vectors, top-k
def all_items(kind=None) -> list[dict]           # for the memory-graph demo
def get(item_id) -> dict | None
```

### Slot contract (so the demo runner can capture the hit)
`action_check_contradiction` must `SlotSet`:
- `contradiction_found: bool`
- `conflicting_decision_id: str | None`  (e.g. `"MEM-0002"`)
- `conflicting_decision_date: str | None`
- `conflicting_decision_summary: str | None`

These are already documented in [scripts/seed_data/demo_cases.json](scripts/seed_data/demo_cases.json) `_meta.slot_contract`.

---

## 4. Assets already in the repo

- [scripts/seed_data/team_history.json](scripts/seed_data/team_history.json) — **60 items**
  (26 decisions, 14 bugs, 20 tasks) of one team's 90-day history (Meridian / Helios
  Platform team). Includes planted contradiction targets `MEM-0002`…`MEM-0007` **and
  decoys** `MEM-0031`…`MEM-0060` so retrieval has to discriminate.
- [scripts/seed_data/demo_cases.json](scripts/seed_data/demo_cases.json) — 6 contradiction
  cases (incl. a no-conflict control) + 3 ingestion cases (incl. the bonus "ingest →
  auto-contradict" loop), each with `input` + `expected_hit`.

---

## 5. The four workstreams

> **Unlock rule:** Person A commits `actions/memory.py` with the schema + **stubbed**
> functions (a fake in-memory dict) in the **first 30 minutes**. B, C, D then build
> against the signatures and swap in the real impl when it lands. Nobody waits.

### 👤 Person A — Memory core *(critical path)*
**Owns:** the storage layer and the contract everyone depends on.

**Tasks:**
1. `actions/memory.py`: JSON load/save (copy the pattern in [actions/tickets.py](actions/tickets.py)), `embed()` via Nebius, `add_item()`, `search()` (cosine in `numpy`), `all_items()`, `get()`.
2. Add a Nebius **embeddings** model group to [endpoints.yml](endpoints.yml) (verify the exact embedding model id in the Token Factory console — same warning as the chat models).
3. Add `numpy` is already a Rasa dep; add `openai` to [pyproject.toml](pyproject.toml) for the embeddings call.

**Where to start:** publish the stub first —
```python
# actions/memory.py  (stub — commit this in the first 30 min)
_FAKE = {}
def embed(text): return [0.0] * 8
def add_item(kind, summary, **f): _FAKE[...] = {...}; return id
def search(query, k=5, kind=None): return list(_FAKE.values())[:k]
def all_items(kind=None): return list(_FAKE.values())
def get(item_id): return _FAKE.get(item_id)
```
Then build the real version: load `.data/memory.json`, embed via Nebius, cache vectors in
the JSON, cosine-rank in `search()`.

**Fallback in your back pocket:** if Nebius embeddings are flaky, `search()` can return
*all* items and let Checkpoint 2's LLM pick — 60 items fit the context window fine.

---

### 👤 Person B — Checkpoint 1: seed + memory graph
**Owns:** loading history into memory and *proving* it's there before the demo starts.

**Tasks:**
1. `scripts/seed_memory.py`: read [team_history.json](scripts/seed_data/team_history.json), call `memory.add_item(...)` for each → writes `.data/memory.json` with embeddings. Idempotent (re-runnable).
2. `scripts/show_memory.py`: a `rich` table (reuse the starter's `rich` dep) grouped by kind, sorted by date — **this is the Checkpoint 1 demo**: "the agent already remembers 90 days."
3. Add a `make seed` and `make show-memory` target to the [Makefile](Makefile).

**Where to start:** write `seed_memory.py` against Person A's **stub** today — it'll
"work" against the fake store immediately, then light up for real once A ships. Verify
with `show_memory.py` that all 60 items load and `MEM-0002` is present.

---

### 👤 Person C — Checkpoint 2: contradiction detection *(the wow)*
**Owns:** the differentiator.

**Tasks:**
1. `data/flows/contradiction_check.yml` — collect `proposal_text` → `action_check_contradiction` → branch on `slots.contradiction_found` (model on [ticket_status.yml](data/flows/ticket_status.yml)).
2. `actions/llm.py` — a thin Nebius chat helper (OpenAI SDK, base_url + `NEBIUS_API_KEY`, Qwen, temp 0, JSON response).
3. `action_check_contradiction` — `memory.search(proposal, k=5)` (deterministic) → pass **only retrieved items** + proposal to the LLM → `SlotSet` the slot contract from §3.
4. Slots + `utter_contradiction_warning` / `utter_no_conflict` in `domain/`.

**Where to start:** hardcode 3–4 fake decisions and nail the **LLM judge prompt** first
(*"Given these past decisions and this proposal, does it contradict any? Return JSON
{contradicts, decision_id, reason}."*). Once it's reliable on the fakes, point `search()`
at B's real seed and test against [demo_cases.json](scripts/seed_data/demo_cases.json)
cases `C-A`…`C-F`. Target: `C-A` hits `MEM-0002`, `C-F` stays clean.

**Resilience talking point for judges:** the LLM only ever sees retrieved memory and
returns JSON at temp 0 — it can't invent a decision. Say this out loud.

---

### 👤 Person D — Checkpoint 3 + integration & demo
**Owns:** closing the loop, and all the cross-cutting glue that otherwise becomes a
last-minute scramble.

**Tasks:**
1. `data/flows/meeting_ingestion.yml` — collect `meeting_transcript` → `action_ingest_meeting` → utter summary (model on [support_triage.yml](data/flows/support_triage.yml)).
2. `action_ingest_meeting` — LLM extracts a JSON list of `{kind, summary, owner, deadline}` → `memory.add_item(...)` each → mock-Jira helper returning fake `PROJ-####` URLs (reuse the id generator in [actions/tickets.py](actions/tickets.py)).
3. **Bonus finale:** after ingesting, run C's contradiction logic on a new item and report — the live loop (case `I-3` → hits `MEM-0002`).
4. **Integration/ops:** [pyproject.toml](pyproject.toml) deps, `.env` keys, `make verify`, and a `scripts/run_demo_cases.py` that feeds [demo_cases.json](scripts/seed_data/demo_cases.json) and prints a ✅/❌ pass table.
5. Own the **demo runbook** — the order of operations on stage.

**Where to start:** get the project running end-to-end *today* — deps, `.env`, `make
verify` green, `make train`, and confirm `make demo-text` talks to Rasa. You're the one
who finds the Nebius-model-id problem before demo morning, not during it.

---

## 6. Sequencing

```
Hour 0   A: commit memory.py STUB + schema  ──► unblocks everyone
         D: deps, .env, `make verify` green, `make train`
Hour 0+  B: seed_memory.py + show_memory.py  (against stub)
         C: contradiction_check flow + LLM judge prompt  (against fakes)
         D: meeting_ingestion flow + mock Jira  (against stub)
Mid      A: real memory.py (JSON + Nebius embeddings) lands
         B: re-seed for real → 60 items in .data/memory.json
         C/D: point search()/add_item() at real data
Late     D: run_demo_cases.py green for C-A..C-F  ──► Checkpoint 2 proven
         D: wire the I-3 bonus loop  ──► Checkpoint 3 finale
         All: demo runbook dry-run
```

**Dependency truth:** everything needs `memory.py`. That's why A stubs first and B
double-staffs Checkpoint 1 (the foundation lands fastest).

---

## 7. Demo runbook (the story on stage)

1. **"It already remembers."** Run `make show-memory` → 90 days, 60 items. (Checkpoint 1)
2. **"It catches contradictions."** Type case `C-A` ("extend legacy auth for mobile") →
   agent surfaces `MEM-0002` from March. Then `C-F` (CSV export) → no false alarm.
   (Checkpoint 2)
3. **"The loop closes."** Feed ingestion case `I-3` → it extracts "launch mobile on
   legacy auth," adds it to memory, then immediately flags it against `MEM-0002`.
   (Checkpoint 3 + bonus)

---

## 8. Gotchas (plan around these now, not at 2am)

- **Python 3.10/3.11 only** — check during `make install`, not demo morning.
- **Nebius model ids drift** — wrong id is the #1 cause of a dead demo. Copy exact chat
  *and* embedding model ids + base URL from the Token Factory console into [endpoints.yml](endpoints.yml). (D verifies via `make verify`.)
- **Keep the LLM on a leash** — in both actions it sees *only retrieved memory*, returns
  *JSON*, temp 0. That's the resilience story.
- **Seed the contradiction target** — Checkpoint 2's demo silently depends on `MEM-0002`
  being in memory. B's seed + C's demo must be tested together.
- **Decoys are intentional** — `MEM-0034` ("deprecate" SMS 2FA) and `MEM-0040` (Redis)
  look similar to real targets; catching the *right* one is the proof the embeddings work.
- **`.data/` is gitignored** — the seed *source* lives in `scripts/seed_data/` (committed);
  the built `.data/memory.json` does not. Always re-seed on a fresh clone.

---

## 9. File checklist

| File | Owner | Status |
|---|---|---|
| `scripts/seed_data/team_history.json` | — | ✅ done |
| `scripts/seed_data/demo_cases.json` | — | ✅ done |
| `actions/memory.py` | A | ☐ |
| `endpoints.yml` (embeddings group) | A | ☐ |
| `scripts/seed_memory.py` | B | ☐ |
| `scripts/show_memory.py` | B | ☐ |
| `data/flows/contradiction_check.yml` | C | ☐ |
| `actions/llm.py` | C | ☐ |
| `action_check_contradiction` (in `actions/actions.py`) | C | ☐ |
| `data/flows/meeting_ingestion.yml` | D | ☐ |
| `action_ingest_meeting` + mock Jira | D | ☐ |
| `scripts/run_demo_cases.py` | D | ☐ |
| domain slots + responses | C/D | ☐ |
