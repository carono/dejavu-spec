# Agent Push-Memory — Conceptual Specification

> **Purpose.** To describe the pattern of associative push-memory for a software agent at the level of
> mechanics, flows, and principles — independent of language, storage, and harness. This document is
> sufficient to build an implementation on any stack: there is no binding to a particular DBMS, language,
> or framework here. Only "what" and "why"; the "exactly how" (schemas, indexes, calls) is the subject of
> implementation documents.
>
> The document describes a **pattern**, not a specific project. Terms are given in two registers:
> the conceptual name (for prose) and the operational name (for code). An implementation is free to choose
> its own identifiers — what matters are the roles, not the letters.

---

## 1. The Problem

Classical agent memory operates on a **pull** model: knowledge is stored somewhere (a vector
database, a graph, files, a RAG index), and the agent **decides for itself** when to look into it — it
invokes a search tool if it happens to think of searching. Hence the typical diagnosis:

> **"The memory exists, but the agent doesn't read it."**

There are two reasons: the agent doesn't know that a relevant fact exists — so it won't formulate a
query; and even when it knows, it economizes on turns and doesn't go to memory "just in case." The
result is that knowledge lies dead weight: the user repeats the same things, the agent steps on the
same rakes, and the environment (versions, paths, passwords, configuration specifics) is recalled
unreliably.

**Push-memory inverts the initiative.** It is not the agent that requests knowledge — **the environment
itself injects** the relevant fact into the context at the moment it is needed, without explicit intent
from the agent. This is a direct analog of human recall: we remember what we need from the features of
the situation (*cue-driven retrieval*), not because we "decided to go to memory."

Push is an **overlay on top of pull, not a replacement**. Deliberate search of the store remains; push
adds an involuntary, ambient layer on top of it.

**An honest limitation.** Knowledge cannot be inserted into a single forward-pass of the model — the
middle of a reasoning step is unreachable. But in the agent loop, **boundaries occur frequently**: the
start of a turn, each tool invocation, receipt of a result, the end of a session. Push fires at these
boundaries and is subjectively perceived as continuous background memory. This is an **approximation**
of "thinking aloud" — a stroboscope instead of continuity — and it is sufficient.

---

## 2. Overview: Two Entities, Three Flows

The system consists of **two** memory mechanisms with different roles and **three** flows between them
and the world.

**Two entities:**

|  | **`factStore`** (the store, LTM) | **`cueMemory`** (cues, trigger index) |
|---|---|---|
| Role | **content**: "what exactly" | **priming**: "is it worth looking" |
| Psycho-canon | long-term memory, recollection (phase 2) | familiarity / recognition (phase 1) |
| What it carries | selected facts + links + trace strength | cheap distillate of triggers |
| Cost | more expensive: reading content, ranking | no model/network, tens of ms |
| Horizon | cross-session, persistent | index is persistent; "arming" — for one turn |

**Three flows** (not one — this is the central idea; the naive "fast memory feeds long-term"
conflates different things):

```
(A) PROJECTION     factStore ──extract triggers──► cueMemory
    LTM is distilled into a cheap, matchable index. Direction: factStore → cueMemory.

(B) PRIMING        live context ──match──► cueMemory ──fire──► push into prompt ──► read factStore
    A cue fires → the fact is injected. The READ path of push-memory.

(C) CONSOLIDATION  session transcripts ──gate / "sleep"──► factStore
    New facts enter LTM. The WRITE path. A separate mechanism, NOT cueMemory.
```

In one phrase: `cueMemory` is a distillate of triggers, **projected from** `factStore` (A); each turn
it is matched against the context and, on a match, **injects a fact** into the prompt (B); and
`factStore` itself is replenished by a separate write-process from transcripts (C). Memory becomes
**push** precisely thanks to `cueMemory`. The three flows must not be conflated — this is the most
common mistake when discussing the pattern.

---

## 3. The Three Flows

### A — Projection (`factStore → cueMemory`)

The store is authoritative but **passive**: on its own it prompts nothing. Projection makes it active
by extracting from each fact its **triggers** and laying them out into a cheap, instantly-matchable
index.

- For each fact, cues are computed: symbolic (entity names, path fragments, commands, error
  signatures, projects, domains, keywords) and, optionally, semantic (a vector anchor).
- The cue index is a **derivative** of the store, not an independent storage. The store remains the
  source of truth; when a fact changes, its cues are re-projected.
- It is cheap precisely because it **carries no content** — only "what to latch onto" and a reference
  to the fact.

The direction of generation is strictly `factStore → cueMemory`. "One from the other" holds, but not
in the sense of "the fast one feeds the slow one."

### B — Priming (the read path of push-memory)

This is what everything is built for: the involuntary injection of knowledge into the context.

1. From the current context (query text, action being prepared, result, environment), features are
   cheaply extracted — **cues of the moment** (without invoking the model).
2. The cues of the moment are matched against the index (`cueMemory`). A match **arms the signal**
   "there is relevant memory here."
3. Candidates are ranked, pass through the salience gate and budget (§5), and the selected facts are
   **injected into the prompt** as a low-priority background hint.

Two-phase nature (familiarity → recollection): phase 1 — cheap broad matching, gives a weak signal
"something familiar here" almost for free; phase 2 — expensive selective reading of the store, launched
only if phase 1 armed confidently enough. This keeps the cost of the read path low on **every** event
and spends resources only where there is a real signal.

### C — Consolidation (the write path)

The store is filled **not by direct writing**, but by a separate offline process that parses the
transcripts of completed sessions.

- From the transcript, **fact candidates** are extracted and filtered by the salience gate (§5.3):
  only the new/important is encoded, not everything indiscriminately.
- A new fact enters the store (by default as `pending`, §6), with automatically extracted cues.
- Periodically, **"sleep"** is run: decay of trace strength, culling of the obsolete, merging of
  duplicates, Hebbian strengthening of links (§5).
- Closing the loop: recall (B) writes activation telemetry → consolidation (C) uses it to reinforce
  the strength of demanded facts. The most valuable signal is a **user repeat**: if the user repeats
  something that "should be in memory," then push failed (see the pain axis §5.3).

Consolidation is a **separate mechanism**, not `cueMemory`. Activation telemetry is the write path, not
the trigger index.

---

## 4. Components

### 4.1. `factStore` — the store (LTM)

Persistent cross-session storage of **selected** facts. The unit is a **fact**: one distilled
assertion (reconstructive, the gist, not a raw log). Fact metadata:

- `kind` — the type: user preference/correction, environment fact, error resolution, project fact,
  reference pointer, etc.;
- `status` — the lifecycle stage: `pending → confirmed → stale → archived` (§6);
- `strength` — current trace strength (recency × frequency), plus `last_activated_at` and
  `activation_count` for reinforcement and decay;
- `salience` — significance at the time of encoding (including the pain component, §5.3);
- `confidence` — confidence in the fact;
- `domain` — domain/group (§7);
- links to other facts and (optionally) an embedding.

The store is the single authoritative source. The fast read path may be duplicated in a cheap local
mirror (§9), but the mirror is a derivative.

### 4.2. `cueMemory` — cues (trigger index)

A cheap, quickly-matchable index of triggers, projected from the store (flow A). It does not store
events — it **awakens**. Three matching channels, by cost and reliability:

- **Keyword / symbolic** — a discrete exact or prefix match (entity name, path, command, error
  signature). Deterministic, instant. **The core of reliable environment recall**: an exact hit, not
  a probability.
- **Graph** — from direct hits, activation spreads across the link graph by 1–2 steps (§5.1). Related
  material surfaces that was not directly in the cue — the essence of associativity.
- **Semantics** — vector proximity of the context to fact embeddings; a fallback for fuzzy
  associations when the symbolic channel is silent. The most expensive, apply last.

A cue is typed: `{type, value, matching mode}`, where the type is path / path prefix / entity name /
command / error signature / project / domain / keyword, and the mode is exact / prefix / pattern. The
discrete symbolic index is the **main thing** that turns "semantically similar" into "exactly about
this command/path/entity" and repairs environment recall deterministically.

### 4.3. The push channel — the injection mechanism

The channel that delivers a fact into the context **without a request from the agent**. It is
implemented by intercepting events of the agent loop (hook/middleware/wrapper — depends on the
harness). Properties of the channel:

- **Event-driven.** It fires at boundaries: session start, turn start, before action, after result,
  session end (§8).
- **Budget.** No more than N facts (start 3–5) and M tokens (start ~300–400) per event — a hard limit
  against context overflow.
- **Habituation.** A fact that has already surfaced in this session is suppressed — unless it triggered
  substantially more strongly (§5.4).
- **Low-authority delivery.** The injection is always marked as a background, possibly stale hint —
  "recalled: …", not "do: …"; the agent verifies named files/values before use.
- **Fail-open.** A channel failure (storage unavailable, timeout, error) **never breaks** the main
  work — it simply injects nothing, silently.

---

## 5. Activation Mechanics

How a fact is selected for display. Below is stack-independent math; the concrete coefficients are
starting defaults for calibration on real telemetry.

### 5.1. Spreading activation over the graph (Hebbian)

Facts form an **adjacency graph**: an edge is an association between two facts, with a type (`relates`,
`derived_from`, `supersedes`, `contradicts`, …), a `weight`, and a co-activation counter
`co_activation_count`.

From direct hits (anchor nodes), activation spreads across links with a fan effect (dividing by the
node's out-degree — protection against hubs) and retention:
```
u_i(t+1) = (1−δ)·a_i(t) + Σ_{j∈neighbors(i)} S · w_ji · a_j(t) / fan(j)
```
where `S` is the spreading factor (≈0.8), `δ` is the per-step retention, and `fan(j)` is the out-degree
of node j. 1–2 iterations suffice (latency budget). This is how related material surfaces that was not
in the cue.

**Hebbian learning of the graph** ("fire together, wire together"): facts that frequently surfaced
**together** in one event increase their `co_activation_count` and link weight. The association graph
is built up from recall practice — learning without retraining the model.

### 5.2. Trace strength: `strength = recency × frequency`

The trace strengthens with use and weakens over time (the Ebbinghaus forgetting curve):
```
strength_i = exp(−Δt / S_i)
```
where `Δt` is the time since the last activation, `S_i` is the accumulated "durability." On each
successful recall: `S_i ← S_i + 1`, `Δt ← 0`. The more frequently a fact is demanded, the more slowly
it is forgotten (the frequency term; many analogs account for recency only). `S_i` is the entry point
of the pain axis (§5.3).

The final candidate scoring combines the channels:
```
score_i = λ1·semantics  +  λ2·activation(spreading)  +  λ3·trace_strength
```
starting weights ≈ {0.5, 0.3, 0.2}. A symbolic exact match goes **outside** this sum — it is a
deterministic "must show" gate, not a probabilistic candidate.

### 5.3. The salience gate and the pain axis

Push is dangerous in exactly one way: **false triggers do active harm** — junk in the context every
turn throws the agent off and eats tokens. Therefore **precision matters more than recall**: better to
stay silent than to make noise. The salience gate cuts off candidates below a threshold before they
enter the budget.

**The pain axis (`pain`)** is a separate salience component, an analog of the emotional coloring of a
trace: a measure of the **urgency/importance** of a fact.

- *What charges pain.* A user repeat (memory failed — this is "painful"), corrections, recurring
  errors. Pain is a memory-quality metric expressed as a charge on a fact.
- *How it acts.* Pain **lowers the effective firing threshold**: the same activation level breaks
  through the gate that a neutral fact would not have broken through (a "cocked hammer"). Technically —
  an addition to the accumulated durability `S_i` / a downward shift of the threshold, not a separate
  "magic" number.
- *Cooling.* The charge **cools** over time (as emotional coloring fades), otherwise the system would
  loop forever on old pains.

Advanced anti-noise (optional): lateral inhibition (strong traces suppress weak competitors nearby,
leaving a sparse set of winners) and a soft sigmoid threshold instead of a hard cutoff — and it is on
this that the pain axis acts with a downward shift.

### 5.4. Habituation

A fact that has already been shown in the current session is **suppressed** on repeated firings within
the same session — just as a repeated stimulus habituates in a human. Exception: if the fact triggered
substantially more strongly than before, the suppression is lifted. Habituation guards against one and
the same hint "getting stuck."

### 5.5. Context budget

A hard limit on the injection volume: **N facts per prompt** (start 3–5) and a token ceiling. When
candidates are in excess, dedup/diversity applies — do not show several near-identical facts, take one
from each cluster. The budget is direct control of the risk of context overflow (§10).

---

## 6. Fact Lifecycle

```
pending ──confirmation──► confirmed ──obsolescence──► stale ──cleanup──► archived
   ▲                            │                         │
   └──────── revival ◄──────────┴──── re-activation ──────┘
```

- **`pending`** — encoded by consolidation, but awaiting confirmation (by a human or a policy); it is
  **not shown** in push. Deterministic environment facts may be auto-confirmed by policy.
- **`confirmed`** — confirmed; visible on the read path, participates in push.
- **`stale`** — obsolete: auto by time (e.g., N days without activations) or on detection of a
  contradiction/replacement (`supersedes`/`contradicts`). Excluded from push, but retained.
- **`archived`** — cleaned out of the active set by consolidation (zero trace strength, a duplicate
  merged into another fact). Does not participate in recall, but is retained for history and audit.

**Revival.** Re-activation of a fact resets the obsolescence markers and raises trace strength — a
`stale` fact may return to `confirmed` if it is once again in demand. Forgetting in this model is
controllable and reversible, not irreversible deletion.

---

## 7. Domains and Grouping

Facts are divided by **domains** — semantic groups (e.g.: database, environment, AI stack, workflow,
debugging). A domain provides:
- grouping for navigation, visualization, and targeted recall;
- an additional cue type (`domain`) for the matching channel;
- scalability: as the store grows, domains keep it surveyable.

**Classification** of a domain is by rules (identifier prefixes, fact kind, key entities). The rules
are the single source of truth, used by both the auto-classifier and the manual layout command.

**Invariant: manual domains are not overwritten by the consolidator.** Some domains are assigned
**only manually** (by the operator or via a panel). Automatic classification and offline consolidation
**never** assign or remove such a domain. This protects intentional manual labeling from being
overwritten by background processes — the operator can always "pin" a fact to a domain, and the system
respects this.

---

## 8. Harness Integration (event points)

Push is implemented by intercepting events of the agent loop — this is the "channel into the context
without a request from the agent." The mechanism depends on the harness; what matters are the points
themselves:

| Event point | When | What push does |
|---|---|---|
| **Session start** | initialization | load the "core": environment/project profile by identifier — not statically, but by fact |
| **Turn start** | the user submitted input | matching against text → injection of relevant facts **before** the agent has begun thinking |
| **Before action** | before executing a command / reading a file | **the environment killer feature**: before addressing an entity, facts about it surface exactly at the moment of need |
| **After action** | a result is obtained | a candidate for encoding: an error in the result → remember the signature and the resolution |
| **Turn / session end** | completion | batch encoding + consolidation from the transcript (flow C) |

Consequence: a large static block of instructions/reference about the environment **ceases to be a
monolith**. Its content migrates into facts with cues; the base prompt retains only a compact
index-pointer, and full facts surface by trigger. This is a direct solution to context overflow: we
load not "everything always," but "what is needed by cue."

---

## 9. Implementation Invariants

What must be preserved in any implementation, regardless of stack:

1. **Push on top of pull, not instead of.** Deliberate search of the store remains available.
2. **Fail-open.** A memory failure never breaks the agent's main work.
3. **Precision over recall on the read path.** A false trigger harms more actively than a miss.
4. **Cheap on read, expensive on write.** The read path is fast (tens–hundreds of ms) on every event;
   the heavy work (distillation, merging, reindexing) is offline.
5. **The fast read path may live on a local mirror** (cues + compact vectors only), so as not to hit
   the network on every turn. The mirror is a derivative, not the source of truth.
6. **Delivery is low-priority, with the caveat "verify before use."**
7. **Writing goes through the salience gate and, by default, through confirmation** (`pending`).
8. **Idempotency and reversibility.** Projection (A) and consolidation (C) are safe on repeat;
   forgetting is controllable and loggable; manual labeling (domains) is not overwritten by automation.

### Checklist of the minimal viable loop
1. Fact storage (§4.1) + symbolic cue index (§4.2). Graph and embeddings — later.
2. Interception of at least one event (easiest — turn start) with injection of text into the context.
3. Extraction of cues from the event (without a model).
4. A symbolic matcher against the index.
5. A gate: threshold + budget + habituation within the session.
6. Delivery with a low-priority mark and fail-open.
7. Filling the store at start (seed); later — consolidation from sessions (flow C).

Order of value: **first the symbolic environment recall** (cheap, reliable, maximum benefit), then
associativity (graph + semantics), then self-learning (consolidation + Hebbian + the pain axis).

---

## 10. What Is Unique in the Pattern

Based on a review of analogs (deep research, task #406; see
`docs/RESEARCH-PRIOR-ART.md`):

1. **PUSH via a harness hook on every prompt — the main differentiation.** All surveyed analogs
   (agent memory systems, RAG frameworks, cognitive architectures) are **pull**: the agent decides for
   itself to search. This pattern cures the specific failure "the memory exists, but the agent doesn't
   read it," making injection the **responsibility of the harness**, not a tool-call of the agent. There
   is **no** analog of a harness hook with per-turn push in the surveyed set — it is the only known
   approach that attacks this failure head-on.
2. **A symbolic cue index over environment features** (path/command/error/container/project) as the
   primary recall channel — cognitively grounded (content-addressable cue-based retrieval), whereas
   practical systems make the vector or graph primary. Deterministic env-recall is a niche.
3. **The combination in a single stack:** symbolic cue-index + link graph (Hebbian) + vector fallback +
   offline "sleep" (strengthening/decay) + salience gate + pain axis. The individual mechanisms exist
   in various systems, but **this specific combination together with push delivery** is unique.
4. **A frequency term in trace strength** (recency × frequency), whereas the nearest analogs account
   for recency only.

### Honest caveats
Two "neuro-grounded" mechanisms — **Hebbian learning of links** and **sleep consolidation** — are
**design hypotheses**, inspired by neuroscience, but **without a direct empirical validator** in the
surveyed literature (the corresponding biological claims were refuted by adversarial verification).
They should be positioned honestly as *inspired-by*, not "proven." This is not a design flaw — but
neither is it an argument from authority.

A separate risk is **index bloat / context poisoning**: as the memory volume grows, the frequency of
false triggers grows (the effect depends on the accumulated content volume, not on order). This is
precisely why the fact budget (§5.5), the salience gate (§5.3), habituation (§5.4), and decay/archiving
(§6) are not decorations, but **mandatory risk control**. An additional threat is memory poisoning on
the write path (flow C parses transcripts, into which a malicious "successful experience" may seep); a
write-time trust filter on the fact's source is needed.

---

## 11. Boundaries of Applicability

- **A stroboscope, not continuity.** Push fires at event boundaries, not "in the middle of a thought."
  For an agent loop with frequent boundaries this is sufficient, but it is an approximation.
- **Noise is the main enemy.** A poorly-tuned gate turns push into spam that actively interferes.
  Calibration of thresholds on real activation telemetry is required.
- **Weak text cues.** Triggers from free text are unreliable (unlike the discrete cues of the
  environment). The two-phase nature and the pain axis mitigate, but do not eliminate, this.
- **Assessing the effect is non-trivial.** "Did the memory work" is measured indirectly: a user repeat =
  a miss; use of a surfaced fact = a hit. A use detector in the telemetry is needed, otherwise
  reinforcement is blind.
- **Privacy and trust.** Memory sees the work stream and stores summaries; the write policy (what may
  be encoded) and the "verify before use" mark are part of the trust contract, not a detail.

---

*The document describes a pattern. The concrete mechanics of storage, indexing, formats, and calls are
the subject of implementation documents and are free to differ, as long as the roles of the two
entities (§2), the three flows (§3), and the invariants (§9) are preserved.*
