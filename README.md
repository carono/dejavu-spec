# Agent Push-Memory — Conceptual Specification

> 🇷🇺 **Русская версия:** see [CONCEPT.ru.md](CONCEPT.ru.md).

> **Purpose.** To describe the pattern of associative push-memory for a software agent at the level
> of mechanics, flows, and principles — independent of language, storage, and harness. From this
> document one can build an implementation on any stack: there is no binding to a specific DBMS,
> language, or framework here. Only the "what" and "why"; the "exactly how" (schemas, indexes, calls)
> is the subject of implementation documents.
>
> The document describes a **pattern**, not a specific project. Terms are given in two registers:
> the conceptual name (for speaking) and the operational name (for code). An implementation is free
> to choose its own identifiers — what matters are the roles, not the letters.

---

## 1. The problem

Classic agent memory works on a **pull** model: somewhere there is a knowledge store (a vector
database, a graph, files, a RAG index), and the agent **decides for itself** when to look into it —
it calls a search tool if it thought to search. Hence the typical diagnosis:

> **"The memory exists, but the agent doesn't read it."**

There are two causes: the agent doesn't know that a relevant fact exists — so it won't formulate a
query; and even knowing, it economizes on moves and won't go to memory "just in case." The result is
that knowledge sits as dead weight: the user repeats the same things, the agent steps on the same
rakes, and the environment (versions, paths, passwords, configuration quirks) is recalled unreliably.

**Push-memory inverts the initiative.** It is not the agent that requests knowledge — **the
environment itself injects** the relevant fact into context at the moment it is needed, without the
agent's explicit intent. This is a direct analogue of human recollection: we remember what we need
by the cues of a situation (*cue-driven retrieval*), not because we "decided to go to memory."

Push is an **overlay on top of pull, not a replacement**. Deliberate search over the store remains;
push adds an involuntary, ambient layer on top of it.

**An honest limitation.** Knowledge cannot be inserted into a single forward pass of the model — the
middle of a reasoning step is unreachable. But in the agent loop **boundaries arrive frequently**:
the start of a turn, each tool call, the receipt of a result, the end of a session. Push fires at
these boundaries and subjectively feels like continuous background memory. This is an
**approximation** of "thinking out loud" — a stroboscope instead of continuity — and it is enough.

---

## 2. Overview: two entities, three flows

The system consists of **two** memory mechanisms with different roles and **three** flows between
them and the world.

**Two entities:**

|  | **`factStore`** (the vault, LTM) | **`cueMemory`** (the cues, trigger index) |
|---|---|---|
| Role | **content**: "what exactly" | **prompting**: "is it worth looking" |
| Psych canon | long-term memory, recollection (phase 2) | familiarity / recognition (phase 1) |
| What it carries | selected facts + links + trace strength | a cheap distillate of triggers |
| Cost | more expensive: reading content, ranking | no model/network, tens of ms |
| Horizon | cross-session, persistent | index is persistent; the "cocking" — for one turn |

**Three flows** (not one — this is the central idea; the naive "fast memory feeds long-term memory"
conflates different things):

```
(A) PROJECTION     factStore ──extract triggers──► cueMemory
    LTM is distilled into a cheap matchable index. Direction: factStore → cueMemory.

(B) PROMPTING      live context ──match──► cueMemory ──fire──► push into prompt ──► read factStore
    A cue fires → a fact is injected. The READ path of push-memory.

(C) CONSOLIDATION  session transcripts ──gate / "sleep"──► factStore
    New facts land in LTM. The WRITE path. A separate mechanism, NOT cueMemory.
```

In one phrase: `cueMemory` is a distillate of triggers, **projected from** `factStore` (A); every
turn it is matched against context and on a match **injects a fact** into the prompt (B); and
`factStore` itself is replenished by a separate write process from transcripts (C). Memory becomes
**push** precisely thanks to `cueMemory`. The three flows must not be conflated — this is the most
common mistake when discussing the pattern.

---

## 3. The three flows

### A — Projection (`factStore → cueMemory`)

The vault is authoritative but **passive**: on its own it prompts nothing. Projection makes it
active by extracting from each fact its **triggers** and laying them out into a cheap,
instantly-matchable index.

- For each fact, cues are computed: symbolic (entity names, path fragments, commands, error
  signatures, projects, domains, keywords) and, optionally, semantic (a vector anchor).
- The cue index is a **derivative** of the vault, not an independent store. The vault remains the
  source of truth; when a fact changes, its cues are re-projected.
- It is cheap precisely because it **carries no content** — only "what to latch onto" and a
  reference to the fact.

The direction of generation is strictly `factStore → cueMemory`. "One from the other" is preserved,
but not in the sense of "the fast feeds the slow."

### B — Prompting (the read path of push-memory)

This is what the whole thing is built for: the involuntary injection of knowledge into context.

1. From the current context (the query text, the action being prepared, the result, the environment),
   features are extracted cheaply — the **cues of the moment** (without calling a model).
2. The cues of the moment are matched against the index (`cueMemory`). A match **cocks the signal**
   "there is relevant memory here."
3. Candidates are ranked, pass through the salience gate and the budget (§5), and the selected facts
   are **injected into the prompt** as a low-priority background hint.

Two-phase-ness (familiarity → recollection): phase 1 is a cheap broad match, yielding a weak signal
"something familiar here" almost for free; phase 2 is an expensive selective reading of the vault,
triggered only if phase 1 cocked confidently enough. This keeps the cost of the read path low on
**every** event and spends resources only where there is a real signal.

### C — Consolidation (the write path)

The vault is filled **not by direct writes** but by a separate offline process that parses the
transcripts of completed sessions.

- From the transcript, **fact candidates** are extracted and filtered by the salience gate (§5.3):
  only what is new/important is encoded, not everything indiscriminately.
- A new fact lands in the vault (by default as `pending`, §6), with automatically extracted cues.
- Periodically a **"sleep"** runs: decay of trace strength, culling of the obsolete, merging of
  duplicates, Hebbian strengthening of links (§5).
- Closing the loop: recall (B) writes activation telemetry → consolidation (C) uses it to reinforce
  the strength of demanded facts. The most valuable signal is **user repetition**: if the user
  repeats something that "should be in memory," it means push didn't fire (see the pain axis §5.3).

Consolidation is a **separate mechanism**, not `cueMemory`. Activation telemetry is the write path,
not the trigger index.

---

## 4. Components

### 4.1. `factStore` — the vault (LTM)

Persistent cross-session storage of **selected** facts. The unit is a **fact**: one distilled
assertion (reconstructive, the gist, not a raw log). Fact metadata:

- `kind` — the type: user preference/correction, environment fact, error fix, project fact,
  reference pointer, etc.;
- `status` — the lifecycle stage: `pending → confirmed → stale → archived` (§6);
- `strength` — the current trace strength (recency × frequency), plus `last_activated_at` and
  `activation_count` for reinforcement and decay;
- `salience` — the significance at the moment of encoding (including the pain component, §5.3);
- `confidence` — confidence in the fact;
- `domain` — the domain/group (§7);
- links to other facts and (optionally) an embedding.

The vault is the single authoritative source. The fast read path may be duplicated into a cheap
local mirror (§9), but the mirror is a derivative.

### 4.2. `cueMemory` — the cues (trigger index)

A cheap, fast-matchable index of triggers projected from the vault (flow A). It stores no events —
it **awakens**. Three matching channels, by cost and reliability:

- **Keyword / symbolic** — a discrete exact or prefix match (an entity name, a path, a command, an
  error signature). Deterministic, instant. **The core of reliable environment recall**: an exact
  hit, not a probability.
- **Graph** — from direct hits, activation spreads across the link graph by 1–2 steps (§5.1). What
  surfaces is the related, which was not in the cue directly — the essence of associativity.
- **Semantics** — vector proximity of the context to fact embeddings; a fallback for fuzzy
  associations when the symbolic channel is silent. The most expensive, apply last.

A cue is typed: `{type, value, match mode}`, where the type is path / path prefix / entity name /
command / error signature / project / domain / keyword, and the mode is exact / prefix / pattern.
The discrete symbolic index is **the main thing** that turns "semantically similar" into "exactly
about this command/path/entity" and fixes environment recall deterministically.

### 4.3. The push channel — the injection mechanism

A channel that delivers a fact into context **without a request from the agent**. It is implemented
by intercepting events of the agent loop (a hook/middleware/wrapper — depends on the harness).
Channel properties:

- **Event-driven.** Fires at boundaries: session start, turn start, before an action, after a
  result, session end (§8).
- **Budget.** No more than N facts (start at 3–5) and M tokens (start at ~300–400) per event — a
  hard limit against context overflow.
- **Habituation.** A fact already surfaced in this session is suppressed — unless it triggered
  significantly stronger (§5.4).
- **Low-authority delivery.** The injection is always marked as a background, possibly stale hint —
  "recalled: …", not "do: …"; named files/values the agent verifies before use.
- **Fail-open.** A channel failure (store unavailable, timeout, error) **never breaks** the main
  work — nothing is simply injected, quietly.

---

## 5. Activation mechanics

How a fact is selected for display. Below is stack-independent math; the concrete coefficients are
starting defaults for calibration on real telemetry.

### 5.1. Spreading activation over the graph (Hebbian)

Facts form an **adjacency graph**: an edge is an association between two facts, with a type
(`relates`, `derived_from`, `supersedes`, `contradicts`, …), a `weight`, and a co-activation counter
`co_activation_count`.

From direct hits (anchor nodes), activation spreads across links with a fan effect (dividing by the
node's out-degree — protection against hubs) and retention:
```
u_i(t+1) = (1−δ)·a_i(t) + Σ_{j∈neighbors(i)} S · w_ji · a_j(t) / fan(j)
```
where `S` is the spreading factor (≈0.8), `δ` is per-step retention, and `fan(j)` is the out-degree
of node j. One or two iterations suffice (a latency budget). This is how the related surfaces, which
was not in the cue.

**Hebbian learning of the graph** ("fire together, wire together"): facts that often surfaced
**together** in one event grow their `co_activation_count` and the weight of the link. The
association graph is built up from recall practice — learning without retraining the model.

### 5.2. Trace strength: `strength = recency × frequency`

A trace grows stronger with use and weakens over time (Ebbinghaus's forgetting curve):
```
strength_i = exp(−Δt / S_i)
```
where `Δt` is the time since the last activation, `S_i` is the accumulated "durability." On each
successful recall: `S_i ← S_i + 1`, `Δt ← 0`. The more often a fact is demanded, the more slowly it
is forgotten (the frequency term; many analogues account only for recency). `S_i` is the entry point
of the pain axis (§5.3).

The final scoring of a candidate combines the channels:
```
score_i = λ1·semantics  +  λ2·activation(spreading)  +  λ3·trace_strength
```
starting weights ≈ {0.5, 0.3, 0.2}. A symbolic exact match goes **outside** this sum — it is a
deterministic "must show" gate, not a probabilistic candidate.

### 5.3. The salience gate and the pain axis

Push is dangerous in exactly one way: **false positives are actively harmful** — garbage in the
context every turn throws the agent off and devours tokens. Therefore **precision matters more than
recall**: better to stay silent than to add noise. The salience gate cuts off candidates below the
threshold before they enter the budget.

**The pain axis (`pain`)** is a separate component of significance, an analogue of the emotional
coloring of a trace: a measure of the **urgency/importance** of a fact.

- *What charges pain.* User repetition (memory didn't fire — this "hurts"), corrections, recurring
  errors. Pain is a memory-quality metric expressed as a charge on the fact.
- *How it acts.* Pain **lowers the effective firing threshold**: the same activation level breaks
  through the gate that a neutral fact would not have broken through (a "cocked trigger").
  Technically — an addend to the accumulated durability `S_i` / a downward shift of the threshold,
  not a separate "magic" number.
- *Cooling down.* The charge **cools** over time (as emotional coloring fades), otherwise the system
  would loop forever on old pains.

Advanced anti-noise (optional): lateral inhibition (strong traces suppress weak competitors nearby,
leaving a sparse set of winners) and a soft sigmoid threshold instead of a hard cutoff — and it is
precisely this that the pain axis shifts downward.

### 5.4. Habituation

A fact already shown in the current session is **suppressed** on repeat firings within the same
session — as a repeated stimulus habituates in a human. Exception: if the fact triggered
significantly stronger than before, the suppression is lifted. Habituation protects against the same
hint "getting stuck."

### 5.5. The context budget

A hard limit on the volume of injection: **N facts per prompt** (start at 3–5) and a token ceiling.
When candidates are in excess, dedup/diversity applies — do not show several near-identical facts,
take one per cluster. The budget is direct control of the risk of context overflow (§10).

---

## 6. The lifecycle of a fact

```
pending ──confirmation──► confirmed ──obsolescence──► stale ──cleanup──► archived
   ▲                          │                         │
   └──────── revival ◄────────┴──── re-activation ──────┘
```

- **`pending`** — encoded by consolidation, but awaiting confirmation (by a human or a policy); in
  push it is **not shown**. Deterministic environment facts may be auto-confirmed by policy.
- **`confirmed`** — confirmed; visible on the read path, participates in push.
- **`stale`** — obsolete: auto by time (e.g., N days without activations) or upon detecting a
  contradiction/supersession (`supersedes`/`contradicts`). Excluded from push, but retained.
- **`archived`** — cleaned out of the active set by consolidation (zero trace strength, a duplicate
  merged into another fact). Does not participate in recall, but is preserved for history and audit.

**Revival.** A re-activation of a fact resets the signs of obsolescence and raises the trace strength
— a `stale` fact may return to `confirmed` if it is again in demand. Forgetting in this model is
controlled and reversible, not an irreversible deletion.

---

## 7. Domains and grouping

Facts are partitioned by **domains** — semantic groups (for example: database, environment, AI
stack, workflow, debugging). A domain provides:
- grouping for navigation, visualization, and targeted recall;
- an additional cue type (`domain`) for the matching channel;
- scalability: as the vault grows, domains keep it surveyable.

**Classification** of a domain is by rules (identifier prefixes, fact kind, key entities). The rules
are the single source of truth, used by both the auto-classifier and the manual layout command.

**Invariant: manual domains are not overwritten by the consolidator.** Some domains are assigned
**only manually** (by an operator or through a panel). Automatic classification and offline
consolidation **never** assign or remove such a domain. This protects intentional manual labeling
from being clobbered by background processes — the operator can always "pin" a fact to a domain, and
the system respects it.

---

## 8. Integration with the harness (event points)

Push is implemented by intercepting events of the agent loop — this is the very "channel into
context without a request from the agent." The mechanism depends on the harness; what matters are
the points themselves:

| Event point | When | What push does |
|---|---|---|
| **Session start** | initialization | load the "core": the environment/project profile by identifier — not statically, but on demand |
| **Turn start** | the user submitted input | matching by text → injection of relevant facts **before** the agent starts thinking |
| **Before an action** | before running a command / reading a file | **the environment killer feature**: before addressing an entity, facts about it surface exactly at the moment of need |
| **After an action** | a result was received | a candidate for encoding: an error in the result → remember the signature and the fix |
| **Turn / session end** | completion | batch encoding + consolidation from the transcript (flow C) |

A consequence: a large static block of instructions/reference about the environment **ceases to be a
monolith**. Its content moves into facts with cues; in the base prompt only a compact
index-pointer remains, and the full facts surface on trigger. This is a direct solution to context
overflow: we load not "everything always" but "what is needed by cue."

---

## 9. Implementation invariants

What must be preserved in any implementation, regardless of stack:

1. **Push on top of pull, not instead of.** Deliberate search over the vault remains available.
2. **Fail-open.** A memory failure never breaks the agent's main work.
3. **Precision over recall on the read path.** A false positive harms more actively than a miss.
4. **Cheap on read, expensive on write.** The read path is fast (tens–hundreds of ms) on every
   event; the heavy work (distillation, merging, re-indexing) is offline.
5. **The fast read path may live on a local mirror** (cues + compact vectors only), so as not to go
   to the network on every turn. The mirror is a derivative, not the source of truth.
6. **Delivery is low-priority, with the caveat "verify before use."**
7. **Writing goes through the salience gate and, by default, through confirmation** (`pending`).
8. **Idempotency and reversibility.** Projection (A) and consolidation (C) are safe on repeat;
   forgetting is controlled and logged; manual labeling (domains) is not clobbered by automation.

### Checklist for a minimal viable loop
1. A fact store (§4.1) + a symbolic cue index (§4.2). Graph and embeddings — later.
2. Interception of at least one event (easiest — turn start) with injection of text into context.
3. Extraction of cues from the event (without a model).
4. A symbolic matcher over the index.
5. A gate: threshold + budget + habituation within the session.
6. Delivery with a low-priority mark and fail-open.
7. Seeding the vault at the start (seed); later — consolidation from sessions (flow C).

The order of value: **first symbolic environment recall** (cheap, reliable, maximum benefit), then
associativity (graph + semantics), then self-learning (consolidation + Hebbian + the pain axis).

---

## 10. What is unique in the pattern

Based on a review of analogues (deep research, task #406):

1. **PUSH via a harness hook on every prompt — the main differentiator.** All surveyed analogues
   (agent memory systems, RAG frameworks, cognitive architectures) are **pull**: the agent decides
   to search itself. This pattern cures the specific failure "the memory exists, but the agent
   doesn't read it" by making injection the **responsibility of the harness**, not a tool call by
   the agent. An analogue of a harness hook with per-turn push is **absent** from the surveyed set —
   this is the only known approach that attacks this failure head-on.
2. **A symbolic cue index over environment features** (path/command/error/container/project) as the
   primary recall channel — cognitively grounded (content-addressable cue-based retrieval), whereas
   in practical systems the primary is a vector or a graph. Deterministic env recall is a niche.
3. **The bundling in one stack:** a symbolic cue index + a link graph (Hebbian) + a vector fallback
   + an offline "sleep" (strengthening/decay) + a salience gate + the pain axis. The individual
   mechanisms exist in various systems, but **this specific combination together with push delivery**
   is unique.
4. **The frequency term in trace strength** (recency × frequency), whereas the nearest analogues
   account only for recency.

### Honest caveats
Two "neuro-grounded" mechanisms — **Hebbian learning of links** and **sleep consolidation** — are
**design hypotheses**, inspired by neuroscience but **without a direct empirical validator** in the
surveyed literature (the corresponding biological claims were refuted by adversarial verification).
They should be positioned honestly as *inspired-by*, not "proven." This is not a flaw of the design —
but neither is it an argument from authority.

A separate risk is **index bloat / context poisoning**: as memory volume grows, so does the rate of
false positives (the effect depends on accumulated content volume, not on order). This is precisely
why the fact budget (§5.5), the salience gate (§5.3), habituation (§5.4), and decay/archival (§6) are
not decorations but **mandatory risk control**. An additional threat is memory poisoning on the write
path (flow C parses transcripts, into which a malicious "successful experience" may seep); a
write-time source-trust filter for a fact is needed.

---

## 11. Limits of applicability

- **A stroboscope, not continuity.** Push fires at event boundaries, not "in the middle of a
  thought." For an agent loop with frequent boundaries this is enough, but it is an approximation.
- **Noise is the main enemy.** A poorly tuned gate turns push into spam that actively interferes.
  Calibration of thresholds on real activation telemetry is required.
- **Weak text cues.** Triggers from free text are unreliable (unlike discrete environment cues). The
  two-phase-ness and the pain axis soften but do not eliminate this.
- **Evaluating the effect is nontrivial.** "Did the memory fire" is measured indirectly: user
  repetition = a miss; use of a surfaced fact = a hit. A use detector in telemetry is needed,
  otherwise reinforcement is blind.
- **Privacy and trust.** Memory sees the flow of work and stores excerpts; the write policy (what may
  be encoded) and the "verify before use" mark are part of the trust contract, not a detail.

---

*The document describes a pattern. The concrete mechanics of storage, indexing, formats, and calls
are the subject of implementation documents and are free to differ, as long as the roles of the two
entities (§2), the three flows (§3), and the invariants (§9) are preserved.*
