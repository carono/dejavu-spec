# Dejavu — Associative (Trigger) Memory for an AI Agent

> The third concept document of dejavu. It covers what `PLAN.md` and `RESEARCH-RLM.md` lack:
> memory **delivery**. PLAN describes the store and **pull-recall** (the agent goes to memory itself),
> RLM is the orchestration mode for heavy queries (also pull). This document describes the **push/trigger**
> layer: memory **surfaces by itself** at the right moment in the agent's reasoning — like human associative
> memory, not like a wiki search.
>
> **Date:** 2026-06-27 (revision 3). **Status:** concept up for discussion. Fields of the Claude Code
> API/hooks are marked ⚠️ — verify before coding (hook mechanics are evolving).
>
> **What revision 2 added** (sections 10–11, plus edits in §2–§3): the trigger fires not only from an *action*, but from
> **any word** in the context; **two-phase** recall (familiarity signal → recollection); the **pain
> axis** of trace strength (threshold lowering + cooling); and an honest section on **weak points and unresolved knots**.
> Sections 1–9 are the base edition; sections 10–11 refine and in places revise them.
>
> **What revision 3 added** (formulas a–k in §3-L1/L2 and §10): the concept stopped being an abstraction —
> spreading activation, lateral inhibition and the sigmoid gate are taken from **SYNAPSE** (arxiv 2601.02744,
> verified against the source); strength/decay — from **MemoryBank** (Ebbinghaus, 2305.10250);
> the two-phase familiarity→recollection — from **RF-Mem** (ICLR 2026, 2603.09250, forms (f)–(h) are
> canonical reconstructions, ⚠️ verify thresholds against the PDF). The pain axis (i–k) is implemented as a modulation of these
> formulas, not as a new number. Prior-art analysis — `docs/RESEARCH-PRIOR-ART.md`.
>
> **Relation to other docs:** `PLAN.md` provides the substrate (tables `facts`/`fact_links`/`fact_embeddings`,
> ingest, REST/MCP). `RESEARCH-RLM.md` provides the heavy-recall mode. Here is the **activation and delivery**
> layer on top of the same store. The documents compose, they do not compete.

---

## 1. Diagnosis: why "memory exists, but the agent doesn't read it"

The user's four pains share **one root**.

| Pain | Surface symptom | Root cause |
|---|---|---|
| Context overflows if everything is kept in instructions | CLAUDE.md/env grows, tokens melt away | **Static loading**: all knowledge is loaded always, regardless of relevance |
| File-based memory exists, but agents don't read it ("didn't look") | MEMORY.md is there, recall is not invoked | **Recall is PULL**: the agent must *decide* to ask itself. No reliable trigger → it doesn't ask |
| Sessions forget the environment (Docker, paths, environment) | "the container is called X" is repeated 5 times | The environment is either static (pain #1) or in pull-memory (pain #2). There is no "command → fact about the environment" binding |
| We want the agent to "recall" as it thinks | — | Human memory is **cue-driven and automatic**; recall in RAG/RLM is **intentional and optional** |

**Key thesis.** The problem is not the store (PLAN solves it) and not the search (RRF/RLM solve it). The problem
is the **delivery model**: pull requires the agent to *remember to remember*. A human does not work this way — memory
is **pushed** into consciousness associatively, by features of the current situation, without a separate act of "let me go
recall". This means dejavu needs a **push layer**, triggered by what the agent is doing right now.

This does not cancel pull (`memory_search` remains for conscious queries). It adds a **second channel** —
unsolicited, but relevant delivery.

---

## 2. What exactly we copy from human memory

Not a metaphor for beauty's sake — each property yields a concrete engineering requirement.

| Property of human memory | Mechanism | Requirement for dejavu |
|---|---|---|
| **Cue-driven retrieval** — we recall by features of the situation, without intent | spontaneous activation on context match | PUSH by observable cues (text, path, command, error), not a tool-call decided by the agent |
| **Spreading activation** — one fact pulls related ones | graph, activation spreads with decay | traversal of `fact_links` for 1–2 hops from direct hits, weight × decay |
| **Encoding specificity** — recall is better in a context similar to the moment of encoding | match between encoding and retrieval context | store the fact's **situational context** (cwd, project, which task, which command) as part of its cue signature |
| **Strength = recency × frequency** | the trace strengthens from use, weakens over time | fields `strength`, `last_activated_at`, `activation_count`; decay + reinforcement |
| **Priming** — recently activated surfaces more easily | temporarily lowered threshold | in-session activation cache: what has already surfaced is closer to the surface until the end of the session |
| **Reconstructive, not verbatim** | we store the gist, not the record | facts are distilled `statement`s, not the raw log (already in PLAN) |
| **Adaptive forgetting** — forgetting is useful | decay + consolidation | decay + prune stale + merge duplicates, otherwise the index gets noisy |
| **Salience-gated encoding** — we encode the new/important/emotional | novelty and significance as a write filter | encode only the new/corrections/repeats/resolved errors, not everything indiscriminately |
| **Hebbian: "fire together, wire together"** | jointly active links strengthen | co-activation counter on `fact_links`; facts that often surface together → stronger link |
| **Familiarity vs recollection** — first a cheap "there is something here" signal, then the expensive retrieval of details | two different memory systems | **two-phase recall** (see §10.2): a broad cheap flag → a narrow selective recollection |
| **Emotional charge** — the charged is remembered more firmly and surfaces more involuntarily, on a weak cue | a separate system (amygdala) modulates the trace | the **pain axis** of trace strength (see §10.3): pain **lowers the firing threshold** and **cools down** over time |

**Honest limitation.** A hook cannot be inserted inside a single forward pass of the model — "in the middle of reasoning" in the
literal sense is unattainable. But in the Claude Code loop, the turn boundary and the tool-call boundary occur **frequently**
(every prompt, every Bash/Read/Edit), and a push at these boundaries feels like ambient memory. This is an
**approximation** of "thinking aloud", and it is sufficient. The document honestly calls this an approximation, not magic.
The exact limits of this approximation are in §11 (strobe vs continuity).

---

## 3. Architecture: 6 activation layers

```
harness event (prompt / pre-tool / post-tool / session-start)
        │
[L0] Cue extraction        cheap, no LLM: tokens, paths, containers, commands, error signatures, project_id
        │
[L1] Activation            symbolic exact-match  +  semantics (local embed)  +  spreading over fact_links
        │                  score = relevance × strength(recency,freq) × link-decay
        │
[L2] Salience-gate         threshold · token budget · habituation (do not repeat what surfaced) · dedup
        │
[L3] Delivery              additionalContext → system-reminder, explicitly low-authority "recalled: …"
        │
   ── the agent works, the loop repeats ──
        │
[L4] Encoding (offline / session-end)   salience-gated: novelty, corrections, user repeats, env discoveries
        │
[L5] Consolidation (periodic)           decay · prune · merge · Hebbian link reinforcement
```

### L0 — Cue extraction (on every event, no LLM)

The hook reads the event and extracts the **cue** — that which the association "hooks onto". No LLM: only parsing,
otherwise we kill the latency of every turn.

> ⚠️ **Revised by revision 2 (see §10.1).** Below, cues are tied to *action types* (Bash command,
> path). This is a reliable but **narrow** case. The full model: the trigger can be **any word or phrase** in the
> text of the conversation, not only a structural action. An action-cue is a strong special case (environment);
> a text-cue is general and weak, and it is exactly what requires the two-phase approach (§10.2) and the pain axis (§10.3) as a filter.

- **From the user prompt:** key tokens, entity names, mentions of projects/domains/containers.
- **From the Bash command (before running):** verb (`docker`, `psql`, `mysql`, `./yii`, `composer`, `nginx`),
  container names (`mysql`, `php83`, `nginx`…), paths (`/mnt/p/projects/...`, `/var/www/...`), domains.
- **From Read/Edit:** file path, project (by prefix), extension/stack.
- **From the post-tool result:** error signature (code, exception class, characteristic string).
- **From the environment:** `cwd`, `gitBranch`, project_id.

Cues are normalized into typed keys: `{type: container, value: mysql}`, `{type: path_prefix,
value: /mnt/p/projects/carono/dejavu}`, `{type: command, value: docker}`, `{type: error, value:
PDOException:SQLSTATE[42S02]}`.

### L1 — Activation (fast retrieval, target < 100–150 ms)

Three candidate sources, combined:

1. **Symbolic exact/prefix match over `fact_cues`** — deterministic, instant. **This is the core of solving
   the environment problem.** The command `docker exec mysql …` → exact match `cue(container=mysql)` → the fact
   "mysql = mysql:8.0, port 3306, root password in the container env". Not semantics, not probability — an exact
   hit. The environment is recalled **reliably**, because the key is discrete.
2. **Semantic match** — a local embed of the cue text (Ollama `nomic-embed-text`, already in the environment)
   against `fact_embeddings`, top-k above the similarity threshold. For fuzzy associations ("this is like what
   happened with the finance planner").
3. **Spreading activation** — from the direct hits we traverse `fact_links` for 1 hop (optionally 2),
   relation-weighted (`relates` > `derived_from` > …), weight × decay coefficient. Thus surfaces
   the related that was not directly in the cue — the essence of associativity.

#### Concrete formulas (not an abstraction): taken from SYNAPSE and MemoryBank

> These formulas are not invented for this document, but proven implementations. Spreading is from **SYNAPSE**
> (arxiv 2601.02744, based on ACT-R/Collins&Loftus, LoCoMo benchmark). Strength/decay is from **MemoryBank**
> (arxiv 2305.10250, the Ebbinghaus curve). All hyperparameters below are the **authors' defaults**, our starting
> values for calibration (§11.4). Detailed prior-art analysis — `docs/RESEARCH-PRIOR-ART.md`.

**(a) Spreading activation — SYNAPSE, discrete propagation over `fact_links`.**

Anchor nodes (direct hits L1.1/L1.2) receive an initial activation, the rest — 0:
```
a_i(0) = α · sim(h_i, h_q)     if fact i is an anchor (cue hit),  else 0
```
Activation spreads over links with a **fan effect** (dividing by the outgoing degree — protection against hubs) and
**retention** (1−δ):
```
u_i(t+1) = (1−δ)·a_i(t) + Σ_{j∈N(i)} S · w_ji · a_j(t) / fan(j)
```
- `S = 0.8` — the spreading factor; `δ` — retention (decay of a node's own activation per step);
  `fan(j) = degout(j)` — the outgoing degree of node j (a direct analogue of the ACT-R fan effect).
- Edge weight `w_ji`: for **temporal** links `w_ji = e^(−ρ·|τ_i − τ_j|)`, `ρ = 0.01` (facts close in
  time are linked more strongly); for **semantic** ones — `w_ji = sim(h_i, h_j)`.
- This replaces our former `link_decay^hops`: decay is not "by the number of hops", but by `(1−δ)` per iteration ×
  edge weight — softer and more stable. 1–2 iterations are enough (our latency budget < 150 ms).

**(b) Fact strength — MemoryBank, the Ebbinghaus forgetting curve.**

Replaces the vague `f(recency, frequency)` with an exact formula:
```
strength_i = R = exp(−Δt / S_i)
```
- `R` — the current trace strength (probability of "surfacing"), `Δt` — the time (in days/hours) since the fact's
  last activation, `S_i` — the accumulated "memory strength".
- **Reinforcement on activation (spaced-repetition):** on each successful recall `S_i ← S_i + 1` and
  `Δt ← 0`. A new fact starts at `S_i = 1`. The more often a fact is demanded — the slower it is forgotten
  (S grows → the curve `e^(−Δt/S)` falls flatter). This is our "the trace strengthens from use".
- **Where the pain axis lives:** `S_i` is the entry point for §10.3. A painful/repeated fact gets
  an addition to `S_i` beyond the usual `+1` (see §10.3), which keeps its `R` high for longer → a lower
  effective surfacing threshold. Pain is a **multiplier/addition to S**, not a separate number.

**(c) Final candidate scoring — Triple Hybrid (SYNAPSE, eq.5), adapted.**
```
score_i = λ1 · sim(h_i, h_q)  +  λ2 · a_i(T)  +  λ3 · strength_i
```
SYNAPSE defaults `λ = {0.5, 0.3, 0.2}` (semantics / activation / structural importance); in our case the third
term is `strength_i` from (b) instead of PageRank (our graph is small, global node importance is less meaningful than
a live trace). The symbolic exact-match (L1.1) goes **outside** this sum — it is a deterministic "must show"
gate, not a probabilistic candidate.

It is cheap to keep the read path in a **local SQLite mirror** (cues + low-dimensional vectors), while the Postgres
brain is the system of record and offline processing. The hook pokes the mirror, not a network API on every turn.

### L2 — Salience-gate and budget (protection against noise)

Push is dangerous in exactly one way: **false positives are actively harmful** — junk in the context every turn throws
the agent off and eats tokens. Therefore precision-at-low-k matters more than recall. The gate:

- **Threshold** on the final score (below it — do not show at all; better to stay silent than to add noise).
- **Budget**: ≤ N facts (start 3–5) and ≤ M tokens (start ~300–400) per event.
- **Habituation**: a fact that has already surfaced in this session is suppressed — unless it triggered
  significantly more strongly (as in a human: a repeated stimulus habituates).
- **Diversity/dedup**: do not show 5 nearly-identical facts; take one from each cluster.

#### Gate by SYNAPSE formulas: lateral inhibition + sigmoid threshold

> Our former "threshold + dedup" is crude. SYNAPSE gives a **proven anti-noise mechanism** (eq.3–4),
> directly solving our main risk (§7.1). We take it.

**(d) Lateral inhibition — strong traces suppress competing neighbors** (winner-take-all among the top):
```
û_i = max(0,  u_i − β · Σ_{k∈T_M} (u_k − u_i) · 𝟙[u_k > u_i] )
```
- `T_M` — the M nodes with the highest activation (`M = 7` in the authors' setup); `β` — the inhibition strength; `𝟙[·]` —
  the indicator. Each node weaker than the leaders is suppressed proportionally to the gap → what remains is a **sparse**
  set of winners, not a diffuse cloud of "everything is a bit relevant". This is stricter than our dedup: it removes not
  duplicates, but **weak competitors next to a strong signal** — exactly the source of push noise.

**(e) Sigmoid threshold instead of a hard cutoff** (a soft "show or not" gate):
```
a_i = σ(û_i) = 1 / (1 + exp(−γ·(û_i − θ)))
```
- `θ` — the surfacing threshold, `γ` — the steepness. The pain axis (§10.3) acts as a **downward shift of θ** for charged
  traces: the same `û_i` breaks a threshold that a neutral fact would not have broken → "the armed trigger".
  This is a mathematically clean entry point for pain into the gate (see §10.3, "threshold lowering").

L2 order: compute `u_i` (L1-b/c) → **inhibition (d)** → **sigmoid threshold (e)** → budget/diversity →
delivery. Symbolic env-facts (L1.1) bypass (e), but **participate** in (d) as the strongest leaders.

### L3 — Delivery

The hook returns `additionalContext` (⚠️ verify the exact field/contract of Claude Code hooks) → the agent sees a
`<system-reminder>` with a note that this is a **low-priority background hint, possibly outdated —
verify before use**. This is exactly the policy already baked into Claude Code memory ("recalled memories
in system-reminder are background, not instructions; if a file/flag is named — verify it still exists").
The wording of push-memory inherits the same caution: "recalled: …", not "do: …".

### L4 — Encoding (offline / at session-end, salience-gated)

Memory forms **without manual labor**, but does not write everything indiscriminately (otherwise the index is a dump). We encode
only the **significant** (analogous to: the brain encodes the new/unexpected/emotional):

- **Novelty** — a fact that does not yet exist (dedup against `fact_embeddings`, already in PLAN).
- **Corrections** — the user corrected the agent → high write priority.
- **User repeat** — *the most valuable signal*: if the user repeats what "should already be in
  memory", it means memory **did not fire**. This is (a) a strong encode, (b) a rise of `strength`/`salience`
  of the existing fact, (c) a check of its cue — why it did not trigger. A repeat = a function of the user's pain
  as a metric of memory quality.
- **Env discoveries** — we discovered a project's PHP version, DB name, webroot path, an nginx-config peculiarity →
  a fact with symbolic cues (`project`, `path_prefix`, `container`).
- **Resolved errors** — "error E → solution S" with a cue on the error signature. Next time the same
  error → the solution surfaces by itself (L1 by `cue(error=…)`).

Cue extraction at encoding is automatic from the context of the moment (cwd, commands, paths around the fact).
By default the write is `pending` (the human-in-the-loop from PLAN is preserved); env-facts with a deterministic cue
can be auto-confirmed by policy — to be discussed.

### L5 — Consolidation (periodic, "sleep")

- **Decay** — `strength` falls over time; what has not been activated for a long time goes deeper.
- **Prune** — `stale`/zero strength → archived.
- **Merge** — duplicates are merged (similarity), as in PLAN.
- **Hebbian reinforcement** — facts that often surfaced **together** in one event get/strengthen
  `fact_links.weight` (co-activation counter). Memory itself builds up the graph of associations from recall
  practice — this is "learning associations" without retraining the model.

---

## 4. Data model extensions (on top of PLAN.md)

The substrate (`facts`, `fact_links`, `fact_embeddings`) is taken from PLAN. We add what makes memory
**active**:

```
facts                         + strength FLOAT            -- current activation trace
                              + last_activated_at TIMESTAMP
                              + activation_count INT
                              + salience FLOAT             -- significance at the moment of encoding

fact_cues  (NEW)             id, fact_id,
                              cue_type ENUM(path|path_prefix|container|command|error|project|domain|keyword),
                              cue_value TEXT,
                              match ENUM(exact|prefix|regex)
                              -- symbolic trigger index; the core of reliable env-recall
                              -- GIN/btree on (cue_type, cue_value)

fact_links                    + weight FLOAT              -- association strength
                              + co_activation_count INT   -- Hebbian

activation_log  (NEW)        session_id, event_type, cue_json,
                              fact_id, score, was_used BOOL, created_at
                              -- what surfaced, whether the agent used it; feeds reinforcement and eval precision
```

`fact_cues` is the main addition. It is precisely the discrete cue index that turns "semantically similar" into
"exactly about this command/path/container" and fixes the environment pain deterministically.

---

## 5. Integration with Claude Code (the user's real environment)

Push is implemented via **Claude Code hooks** — this is that very "channel into the context without the agent's request".
A hook is a small fast CLI, reads the event from stdin, writes context to stdout, goes to the local
SQLite mirror of dejavu (not to the network Postgres — latency).

| Hook ⚠️ (verify names/contract) | When | What it does |
|---|---|---|
| `SessionStart` | session start | loads the project "core": the environment profile (PHP version, DB, paths) **by project_id**, not statically in CLAUDE.md |
| `UserPromptSubmit` | the user submitted a query | L0–L3 over the query text → injection of relevant facts before the agent starts thinking |
| `PreToolUse` (matcher: Bash) | **before** running the command | **the killer feature for the environment**: the command `docker/psql/mysql/./yii …` → injection of facts by container/path/command BEFORE execution. The agent learns "mysql 8.0, port 3306, password in env" exactly when it reaches into mysql |
| `PreToolUse` (Read/Edit) | before working with a file | facts by project/file path |
| `PostToolUse` | after the result | L4 candidate: an error in the result → remember the signature; later — "solved something similar" |
| `Stop` / session-end | end | L4 batch encoding + L5 consolidation (via session ingest, PLAN Stage 2) |

**Ties to what already exists in the user's environment:**
- `~/.claude/CLAUDE.md` + `env/*.md` **stops being a static blob**. Its content (the Docker map,
  DB, web) migrates into `facts` with cues by containers/paths/domains. Only an **index pointer** remains in the
  base prompt (like the current `MEMORY.md`) — cheap; full facts surface by cue. This is a direct
  solution to pain #1 (context overflow): we load not "everything always", but "what is needed by trigger".
- The existing memory format (`MEMORY.md` + fact files with frontmatter and `[[links]]`) is **already almost this
  model**: one fact = one file, `[[name]]` = `fact_links`, `description` = a cue for relevance. dejavu
  builds an **active layer** on top of it: the cue index, strength, push delivery. The migration is evolutionary, not
  from scratch.

---

## 6. Comparison with pull approaches (why push is something else)

| Criterion | RAG/pull (`memory_search`) | RLM mode | **Associative push (this doc)** |
|---|---|---|---|
| Who initiates recall | the agent (must remember to remember) | the agent | **the environment/harness — automatically** |
| When it fires | when the agent decided to ask | on a heavy query | on a cue: prompt / command / path / error |
| Solves "they don't read memory" | no (this is pull) | no | **yes — reading is not at the agent's discretion** |
| Solves "forgot the environment" | partially (if it asks) | no | **yes — PreToolUse(Bash) by container/path** |
| Solves context overflow | yes (loads on request) | yes | **yes + the base prompt slims down to an index** |
| Risk | low recall (didn't ask) | latency/cost | **noise from false positives** ← the main engineering challenge |
| Main defense | — | limits | **salience-gate + budget + habituation** |

Push and pull **complement** each other: push gives an ambient background ("recalled"), pull gives a conscious deep query. On
a heavy triggered investigation, push can **hand control over to RLM mode** (`RecursiveRecall` from
RESEARCH-RLM). Three documents compose into one system: PLAN — memory, RLM — heavy recall, this one —
trigger and delivery.

---

## 7. Risks and how we close them

1. **Noise/false positives (the main one).** Push junk harms every turn. → high threshold, small budget,
   habituation, dedup; a precision-at-low-k metric by `activation_log.was_used`; A/B of the threshold. The principle:
   **better to stay silent than to add noise** (unlike pull, where staying silent = failure).
2. **Latency of every turn.** Push sits on the critical path of every event. → L0/L1 without LLM,
   a local SQLite mirror, caching cue maps at ingest, a time budget < 150 ms, otherwise the hook returns empty
   (fail-open: no memory is better than a slowdown).
3. **Prompt injection.** The content of sessions/files is untrusted (PLAN §Security). Push **amplifies**
   the surface — it inserts into the context automatically. → deliver only the **distilled** layer
   (never raw `tool_use`/`tool_result`/`thinking`), mark it low-authority, no auto-writes,
   env-facts with a deterministic cue are a separate trusted category.
4. **Sticking/self-confirmation.** Hebbian may cement a false association. → decay; `was_used=false`
   lowers strength; the human sees and cleans up (the inbox UI from PLAN Stage 4).
5. **Privacy/index scale.** Everything is local (Ollama embed, SQLite mirror, Postgres on the host) — nothing
   goes outside. Consolidation keeps the index compact.

---

## 8. Roadmap (from maximum benefit to completeness)

**MVP (1 pain, minimum risk, measurable) — "environment memory on a single hook".**
- Only symbolic cues (`fact_cues`), without embeddings and LLM.
- A single hook `PreToolUse(Bash)`: parses container/path/command → injection of env-facts from the SQLite mirror.
- Seeding facts from the existing `env/*.md` (Docker map, DB, web) with cues.
- Metric: a drop in the number of user repeats "the container is called X". This is a direct measurement of pain #3.

**Phase 2 — semantics and the prompt trigger.** L1 semantic match (Ollama embed), the hook `UserPromptSubmit`,
the L2 threshold/budget.

**Phase 3 — associativity.** Spreading activation over `fact_links`, strength/decay, in-session priming.

**Phase 4 — auto-encoding.** L4 at session-end via session ingest (PLAN Stage 2), the "user repeat"
and "resolved error" detectors.

**Phase 5 — consolidation.** L5: decay/prune/merge + Hebbian. Eval precision by `activation_log`.

Each phase is a self-contained benefit; you can stop at any of them.

---

## 9. Open questions (for discussion with the user)

1. **The Claude Code hooks contract** ⚠️ — the exact event names and the field for injecting context
   (`additionalContext`?), size limits, whether from `PreToolUse` one can append to the context without blocking
   the tool. The entire delivery depends on this — verify before coding.
2. **Auto-confirm of env-facts** or everything via `pending`? Deterministic env-cues may deserve
   a trusted auto-category — otherwise the human-in-the-loop becomes a bottleneck exactly where the pain is sharpest.
3. **Where the push threshold is.** The starting numbers (3–5 facts, ~300–400 tokens, a similarity threshold) are to be tuned on
   real sessions; a calibration period with the `was_used` log is needed.
4. **The local mirror**: a SQLite copy of cues+vectors on the read path — do we accept it for the sake of latency, or
   do we read Postgres directly and measure whether it is really < 150 ms?
5. **The volume of migration from `env/*.md` and `~/.claude/memory`** — do we seed the key facts manually or write a
   one-time importer with automatic cue extraction?

---

## 10. Concept revision: text trigger, two-phase approach, pain axis

Three refinements discussed in the 2026-06-27 conversation. They are **not three separate edits, but one coherent
move**: it is worth expanding the trigger to any word — and the other two points become not wishes, but a
forced necessity. Widened the input → obliged to strengthen the filter → the filter is obliged to prioritize by
significance (pain), and to retrieve in two steps.

### 10.1. Trigger — any word, not just an action

The cue is not a structural action (ran a command, opened a file), but **the meaning of the conversation as a whole** —
any word or phrase. As in a human: a mention by itself wakes memory even before you started doing
something with your hands ("need to sort out docker" → already stirred, there was no command yet). This is **earlier
in time and broader in coverage** than the action trigger.

The cost is honest: **a word is a weak signal.** In every phrase there are dozens of them, almost all empty. An action-cue by
itself is strong ("ran docker" = docker is definitely important now); a text-cue is weak ("mentioned docker in passing").
This means deciding "surface or not" by the word itself is **impossible** — it is too cheap. You decide by the
**significance of the trace accumulated behind the word**. Hence the direct consequence: a broad trigger is impossible without a
strong significance filter (§10.3) and without splitting into a cheap signal and expensive retrieval (§10.2).

Engineering-wise, this shifts L0/L1: the cue surface is the entire token stream of the dialogue, not only action events.
An action-cue remains a **strong special case** (environment: container/path/command — a deterministic
exact-match, §3 L1.1). A text-cue is a general weak case on top of it.

### 10.2. Two-phase approach: first "there is a cue", then "what exactly"

Human memory separates two processes (memory psychology: **familiarity vs recollection**):
- **Familiarity** — a fast, cheap, almost contentless signal "there is something here, I have encountered this".
  It comes first and is **empty**: you do not yet know *what*, you know only *that it exists* (tip-of-the-tongue,
  dejavu).
- **Recollection** — slow, effortful, retrieving details (that very cluster of the related, §3 L1.3 spreading
  activation).

The system does the same, and this solves two things at once:

1. **Economy against the noise driven up by §10.1.** A cheap broad **familiarity flag** ("on the topic 'docker'
   there is sensitive memory") can be set on almost everything — it is tiny. But the expensive **recollection**
   of the cluster is rare and selective. Familiarity is generous, recollection is stingy. You can flag broadly, retrieve
   narrowly — exactly the human compromise.
2. **Push elegantly passes the baton to pull.** Phase 1 is involuntary (the environment pushes the flag). Phase 2 can be
   conscious: the flag **prompts the agent to start recalling**, and it does the digging itself. Push provides the reason to pick up
   the shovel, pull retrieves the depth. This is a direct answer to the original pain "no reason to look" — the reason
   comes by itself.

> ⚠️ **Honestly: the two-phase approach shifts the will problem from phase 1 to phase 2, it does not remove it.** The flag is
> push, but "leaning in and recalling" is again pull, the very will that failed initially.
> The fork is not resolved (see §11): either auto-deliver the content in the second phase (we lose the economy,
> the noise risk returns), or rely on the agent's will to lean in (the "didn't look" returns).
> The likely compromise: high-pain (§10.3) — auto-deliver; neutral — leave as a flag.

A caveat about false familiarity: as humans have false dejavu (the flag fired — behind it is emptiness), so
here the flag will sometimes light up on an irrelevant topic. This is tolerable precisely because the flag is cheap, and phase 2 will filter it out.
The main thing is not to confuse "there is a signal" with "definitely needed".

#### Concrete formulas: RF-Mem (the two-phase approach is not an abstraction — it is already implemented)

> **RF-Mem** (arxiv 2603.09250, ICLR 2026) is an almost isomorphic implementation of this section: a cheap
> familiarity path as a gate → an expensive recollection path with iterative expansion. We take its mechanics;
> our only difference is **what sets the flag** (see below). ⚠️ Exact thresholds/coefficients are in the PDF
> (FlateDecode), the forms below are canonical and verified against the paper's prose; the numbers are to be calibrated per §11.4.

**(f) Familiarity flag — mean + entropy over top-K similarities (phase 1, cheap).**

We take the top-K similarity scores `{s_1..s_K}` from L1.2 (one embed pass, without spreading — hence cheap).
We normalize into a distribution and compute two numbers:
```
p_i = softmax(s_i) = exp(s_i) / Σ_j exp(s_j)
s̄   = (1/K) · Σ_i s_i              -- mean cue strength
H   = − Σ_i p_i · log p_i           -- entropy (signal spread)
```
- **High familiarity** ⟺ high `s̄` AND low `H` (one or two facts dominate — there is a clear cue).
- **Low/diffuse** ⟺ low `s̄` or high `H` (either nothing, or "a bit of everything" = noise).

**(g) Phase gate — by the flag (and in our case — also by pain).**
```
if s̄ ≥ θ_fam  AND  H ≤ θ_H   → Phase 2: recollection (spreading L1.3 → the expensive cluster)
else                           → stop at the flag (or stay silent entirely)
```
RF-Mem triggers phase 2 by the uncertainty of the embeddings. **Our differentiator (§10.3):** for **painful**
topics the threshold `θ_fam` is lowered (a painful trace breaks through phase 2 on a weak cue), and for them phase 2 is
**auto-delivered**, without waiting for the agent's will — this closes the knot §11.2 (see below).

**(h) Recollection path — α-mix iterative expansion (phase 2).**

Candidates are clustered; the query is iteratively mixed into cluster centroids and re-retrieved —
"retrieval of the cluster" instead of a single top-K:
```
q^(t+1) = α · q^(t) + (1−α) · c_t        -- c_t = the centroid of the selected candidate cluster
        → re-retrieval by q^(t+1), 1–3 iterations
```
This is exactly our spreading activation (§3-L1-a) from the other end: RF-Mem expands in the embedding
space through centroids, we — over `fact_links`. **Both can be used:** α-mix as a fallback where
the link graph is sparse (cold start, §11.4).

**Closing knot §11.2 with a formula.** The fork "will in the second phase" we resolve as RF-Mem — **the system itself
executes both phases** (phase 2 is auto), the agent does not "decide to lean in". Policy: pain charge `> θ_pain` →
auto-recollection and auto-delivery; neutral ones — flag + budget (g). Will is no longer needed.

### 10.3. Pain axis: trace significance that lowers the threshold

Trace strength is **three-dimensional**: recency × frequency × **pain**. Pain is whether the topic was a source of an error, conflict,
your irritation, a **repeat**. This is not just "a higher weight in ranking":

- **Pain lowers the firing threshold.** A neutral fact surfaces only on a strong direct cue.
  A painful one — on the **faintest mention**, on a single passing word. Painful memory has an **armed trigger**.
  This literally copies the property of human emotional memory: the charged climbs by itself, involuntarily, from
  a smaller stimulus. This is exactly why emotion exists in memory — a priority signal "this has already hurt, do not
  let it recur".
- **Pain cools from silence.** If a topic has not hit for a long time, has been resolved, does not surface in conflicts — the pain
  weight **falls**. Otherwise we get a neurotic, forever dragging an old grievance ("docker is DANGEROUS" a year
  after everything settled). Healthy memory lets the charge fade; getting stuck is a pathology. Pain
  both accumulates from friction and heals from silence.
- **Kinds of pain are not equal.** An error is "something broke in the world". Conflict/irritation is "the relationship suffered".
  **A repeat is special**: it is memory giving a failing grade **to itself**, a meta-signal about its own failure.
  The highest priority of the three — the only signal by which the system fixes not knowledge about the world, but itself.

The circle closes right back to the user's original pain: **his irritation is fuel** that makes
memory aggressive exactly on the topics where it let him down. Pain → a strong trace → a low threshold → next
time it surfaces earlier and from less → the friction ceases. Memory itself learns to be a paranoid pointwise,
by scars, not in general.

The pain axis must be **orthogonal to reliability** (§11): a high-pain but unreliable fact
(a fabrication with an armed trigger) is the worst case, climbs constantly and confidently. Pain and trust are different numbers.

#### Concrete formulas: pain via S (MemoryBank) and a shift of θ (SYNAPSE gate)

> The pain axis is the only **original** part of ours (§RESEARCH-PRIOR-ART: there is no holistic analogue).
> But we implement it **not as a new number**, but as a modulation of the already inscribed proven formulas — two levers:

**(i) Pain — an addition to trace strength `S` (input into the MemoryBank formula b).** Each pain event
adds to a fact's `S` more than the usual `+1`:
```
on recall:            S_i ← S_i + 1                       (neutral reinforcement, formula b)
on a pain event:      S_i ← S_i + w_pain(kind)            kind∈{error, conflict, repeat}
                       w_pain: repeat > conflict > error  (a repeat is a meta-failure of memory itself, the highest)
```
Through `strength_i = exp(−Δt/S_i)`, a high `S_i` keeps `R` high for longer → a painful fact "surfaces
earlier and from less" **automatically**, without a separate code branch. Pain enters the same place where strength lives.

**(j) Pain — a downward shift of the threshold θ (input into the SYNAPSE sigmoid e).** The "armed trigger" = for a charged trace
the threshold is easier to break:
```
θ_i = θ_0 − κ · pain_i        →   a_i = σ(û_i) = 1/(1 + exp(−γ·(û_i − θ_i)))
```
`pain_i` — the pain charge of the fact, `κ` — how strongly pain lowers the threshold. The same `û_i` that a neutral
fact would not break (`û_i < θ_0`), a painful one breaks (`û_i > θ_i`). This is §10.2-(g) "θ_fam lower for
painful" and §10.3 "lowers the threshold" — in a single formula.

**(k) Cooling — exponential decay of pain from silence (anti-neurotic).**
```
pain_i(Δt_pain) = pain_i^max · exp(−Δt_pain / τ_pain)
```
`Δt_pain` — the time since the last pain firing on the topic; `τ_pain` — the characteristic healing time
(days–weeks, to be calibrated). A resolved topic → `pain_i → 0` → the threshold returns to `θ_0`. Pain
"accumulates from friction (i) and heals from silence (k)" — literally two formulas with different signs.

**Relation of the levers.** (i) accumulates **long** strength (between sessions, the slow L5 contour); (k) holds the
**acute** pain charge, which cools quickly; (j) translates the charge into an instant surfacing threshold. Three
independent timescales — this is exactly "separate the ephemeral and the persistent", which §11.1 noted as a
gap. Reliability (§11.1) is a **fourth, separate** number `trust_i`, not mixed into these formulas:
it cuts delivery at L2 (unreliable — we do not show confidently), but does not touch `S`, `θ`, `pain`.

---

## 11. Weak points and unresolved knots (an honest breakdown)

A concept no one has tried to break is worth nothing. Below — without anesthesia. The general pattern:
**weaknesses scale together with ambition.** The narrow version (environment memory on exact action-cues) avoids almost
all of them; the grand one (ambient text trigger + rich spreading activation + "emotion") catches them
all at once.

### 11.1. What to add to make it more complete (coverage gaps)

- **Memory can be wrong — this is scarier than empty.** The document is about *adding* a fact and about how it
  *surfaces*, but almost nothing about **going stale**: the container was renamed, the path moved. A confidently surfaced
  outdated fact with an armed trigger is active harm. A mechanism for **refutation/replacement** is needed, not
  only accumulation (`supersedes`/`contradicts` from PLAN are a stub, but not a policy).
- **Trust is a separate axis, orthogonal to pain.** "You said X directly" ≠ "inferred from a session" ≠ "fabricated".
  Trace strength and trace reliability are different things, they cannot be mixed into one number.
- **The policy of "what is worthy of remembering" is not spelled out — and this is the hardest thing.** "We encode the significant" is
  a slogan; *what* to count as significant is the unattractive core itself. Overdo it — a dump (pain #1 returns);
  underdo it — the original problem returns.
- **Horizons are mixed:** priming (within a session) and persistent memory (between sessions) are different
  timescales, they must be separated: what is ephemeral, what is preserved.
- **Scope with a pile of agents/projects.** Memory — per agent, per project, shared? Should a lesson from project
  A surface in B? Rather: env-memory is shared, project memory is isolated. Not formulated —
  and this is an architectural fork.
- **Conflict of two memories** — two contradictory facts triggered; there is no resolution mechanism.
  At a minimum — "they disagree, flag to the human", not silently pick one.
- **Outcome signal: it helped or it misled.** Reinforcement now relies on "it surfaced" (activation), but a
  fact should strengthen from having **helped**. Distinguishing "helped" from "misled" without a clean outcome criterion is very
  hard; without this, strength and pain drift on noise.

### 11.2. Architectural knots

- **The two-phase approach quietly brings back "didn't look"** (the main crack, §10.2): phase 2 is again pull.
  The auto-delivery-vs-will fork is not resolved.
- **Strobe vs continuity, and the text trigger lays it bare.** The ideal "any word triggers"
  requires continuous observation; reality is a check only at discrete boundaries (turn, tool-call).
  A word in the middle of the agent's own generation will not trigger anything until the end of the turn. The more ambitious the
  text trigger, the more noticeable the gap between the dream and reality.
- **A broad cheap accurate flag — three wishes, two come true.** The familiarity flag must run over every
  word, be cheap AND accurate. "Cheap + over everything + good" is a separate unsolved engineering task, not a
  given.

### 11.3. Philosophical knots

- **This is not the agent's memory, but our model of what it should associate.** The links in the network are those that
  **we** prescribed, not those the model formed (its real memory is the frozen weights). In a human,
  associations are emergent and surprise him; ours are limited by our imagination of linking. This is a **ceiling**
  of human-likeness: "associative memory" here is more honest as a curated index imitating its function.
- **The "pain" is borrowed, not the agent's.** The agent feels nothing; under the pain axis, **your** friction is encoded,
  projected into its memory. This is "the agent remembers that it hurt the user", not "him". Functionally
  it works, philosophically it is a different thing; substituting one for the other is self-deception.
- **There is no guarantee that injection works *as memory*, and not as just-more-text.** The agent may pattern-
  match on what was slipped in, take it as an instruction or as noise. No real recollection happens — there is
  context augmentation dressed as recollection. Whether it works as memory or as a hindrance is an empirical question.

### 11.4. Practical knots

- **Calibration will eat more effort than the store.** Thresholds, pain weights, decay rates, budget — everything
  is tuned on live sessions, and the goal (surfacing accuracy) is almost unmeasurable: "was the memory useful"
  has no clean ground truth.
- **Cold start and learning on pain already inflicted.** Empty memory is useless; painful traces by
  definition form **after** the pain has already occurred. The system will not prevent the first repeats —
  at best the next one. Constructively, it learns on friction it did not forewarn.
- **The push channel can be habituated — and then "didn't look" is recreated at a new level.** A couple of surfacings
  with junk — and the agent (and you) learn to ignore the channel, as the file was ignored. Noise **burns trust in
  the channel**; one loud false signal is costlier than ten missed quiet ones. Hence the §L2 rule: **better
  to stay silent than to add noise.**
- **The store rots, and no one likes weeding.** Without a working consolidation contour (§L5), memory
  degrades into the same mush as a pile of files. The most important and most disliked part.
- **The pain axis can be noised.** A flaky error, firing time after time, will inflate pain on an irrelevant
  topic. Distinguishing "really painful" from "noisily repeated" is a separate task about signal quality.

### 11.5. Honest conclusion

The concept is **correct in its diagnosis** (delivery, not search; push, not pull; pain as an axis) and **strong in narrow
application** (environment, action-cue, real pain from repeats — there almost all the knots above do not bite). Its
full form rests on three unresolved knots: the return of will in the second phase (§11.2), the unproven claim
that injection works *as memory* (§11.3), and the unmeasurability of selection quality (§11.4). The conclusion is not "don't
do it", but: **build from a narrow strong core**, and pay for each step toward the "full brain" with a separate check
that the next beautiful principle did not fall apart on one of these knots.

---

## References and relation

- `docs/PLAN.md` — the store, ingest, REST/MCP, the `facts` layer (the substrate of this document).
- `docs/RESEARCH-RLM.md` — the heavy-recall mode (push can hand control over to it).
- Anthropic Contextual Retrieval — https://www.anthropic.com/news/contextual-retrieval
- Spreading activation (Collins & Loftus), encoding specificity (Tulving), familiarity vs recollection
  (dual-process memory), emotional modulation of memory (amygdala), strength models — classical
  cognitive psychology; used here as engineering requirements (function), not as exact models.
