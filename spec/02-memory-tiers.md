# Dejavu — two memory mechanisms: the cue-primer and the long-term store

> Definition document (task #260, 2026-06-28, revision 2). Purpose — to fix the **established
> names** for the agent's two memory mechanisms, so that in code, discussions and the tracker everyone
> calls them the same way. This is a glossary + role model, not a design of the mechanics: layers L0–L5 are
> described by `ASSOCIATIVE-MEMORY.md`, storage — by `PLAN.md`.
>
> **Revision 2 (important):** the fast layer was rethought. Previously it was called "working memory /
> `flashMemory`" and defined as "a log of what is happening right now" (`activation_log`). **This was
> incorrect.** Its essence is not to store the current state, but to **carry cue signals**: a distillate of triggers
> that awakens access to the long-term store. This is phase-1 (familiarity) of the two-phase model of
> `ASSOCIATIVE-MEMORY.md §10.2`. The name was changed to `cueMemory`; `activation_log` is attributed to write-path
> telemetry, not to the fast mechanism.

---

## TL;DR

| | **Cue-memory (primer)** | **Long-term store** |
|---|---|---|
| Code name (API) | **`cueMemory`** (handle `cues`) | **`factStore`** (handle `facts`) |
| Russian name | **«зацепки»** (cues) (alt: «наводки»/leads, «маяки»/beacons) | **«свод»** (the store) (alt: «копилка»/piggy-bank, «факты»/facts) |
| Ergonomics | `cues.match(context)` · "the cue fired" | `facts.get(cue)` · "pulled it from the store" |
| Psycho-canon | familiarity / recognition (phase 1) | long-term memory (LTM), phase 2 recollection |
| Role | **priming**: "is it worth looking" | **content**: "what exactly" |
| What it carries | distillate of triggers (cue + topics) | selected facts + links + trace strength |
| Horizon | index is persistent; the arming — per turn | cross-session, persistent |
| Carrier | `fact_cues` + SQLite mirror of the read-path + familiarity flag (L3) | `facts` + `fact_cues` + `fact_links` + `fact_embeddings` |
| Relation | **projection** from `factStore` (factStore → cue) | source of truth |

**In one phrase:** `cueMemory` is a cheap distillate of triggers, **projected from** `factStore`;
it is matched against the live context every turn and, on a match, **arms a cue signal (primer)**,
pushing the agent to read `factStore`. `cueMemory` answers "when/whether to look", `factStore` —
"what exactly". Memory becomes **push**, not pull, precisely thanks to `cueMemory`.

---

## 1. Cue-memory (`cueMemory`, phase-1 familiarity)

**Definition.** A cheap, fast-to-match **distillate of triggers**, projected from
`factStore`. Its job is not to store events, but to **awaken**: to match the cue of the current context
(L0) against its set of triggers and, on a match, **arm a cue signal (primer)** ("there is
relevant memory here, go look"), which pushes the agent to read `factStore`. This is the active front of the
passive store — the thing that turns memory from pull into push.

**What it carries.**
- Symbolic triggers (`fact_cues`): container names, ports, verbs, path fragments, error
  signatures, topics/projects — discrete, instantly-matchable cues (the core of reliable env-recall).
- (Priority 2) semantic anchors for fuzzy recognition.
- The output is a **familiarity flag** (L3): a short low-authority line "⚡ memory on topic: …",
  almost content-free, whose role is to prompt, not to tell.

**Properties.**
- *Horizon.* The trigger index is persistent (mirrors `factStore`); the **arming** is ephemeral, for a single
  turn (with habituation: do not fire it repeatedly within a session).
- *Cost.* No LLM/network: a symbolic match over a flat/indexed set, tens of ms
  (`ASSOCIATIVE-MEMORY.md §3-L1.1`). Cheap precisely because it does not carry content.
- *Carrier.* `fact_cues` + a local SQLite mirror of the read-path + the delivered flag (L3).
  **In the MVP today** — the `cues` field in `facts.jsonl` (triggers attached to a fact) + a scan in
  `dejavu-push.sh`; the familiarity flag — the line `⚡ environment memory on topic: …`.
- *What it is NOT.* Not `activation_log` (that is write-path telemetry, see §3). Not "working memory /
  the state right now". Not the LLM context window.

## 2. Long-term store (`factStore`, LTM, phase-2 recollection)

**Definition.** A persistent, cross-session store of **selected** facts with triggers,
links, embeddings and trace strength — the content depth that recollection retrieves after
`cueMemory` has armed the priming.

**What it stores.** `facts` (+ `strength` per Ebbinghaus, `last_activated_at`, `activation_count`,
`salience`), `fact_cues` (the symbolic index — also the source of the projection into `cueMemory`), `fact_links`
(the graph, `weight`, `co_activation_count`), `fact_embeddings`.

**Properties.**
- *Horizon.* Permanent, cross-session. System of record — Postgres `brain`; the read-path is duplicated
  into a SQLite mirror (the very one that serves `cueMemory`).
- *Write.* Expensive, **salience-gated**: only what matters (novelty, corrections, user repetitions,
  env-discoveries, resolved errors). It is filled by **consolidation** (see §3), not by direct writes.
- *Read.* `facts.get(cue)` — recollection: symbolic exact + semantics + spreading over `fact_links`
  (L1), the L2 gate, L3 delivery. Plus a deliberate pull `memory_search` over the same store.
- *MVP today.* Degenerated to a flat `facts.jsonl` (`id`/`cues`/`text`), edited by hand.

---

## 3. The relation: three streams, not one

The earlier formulation "the fast feeds the long-term" conflated different things. The honest model is **three
separate streams**:

```
(A) PROJECTION       factStore ──extract cues──► cueMemory
    factStore hands the triggers of each fact into a cheap matchable index.
    Direction of generation: factStore → cueMemory.

(B) PRIMING          live context ──match──► cueMemory ──fire──► flag (L3) ──► facts.get → recollection
    Phase-1 (familiarity) arms, phase-2 (recollection) retrieves the depth. This is the read-path of push-memory.

(C) CONSOLIDATION    session signals (activation_log) ──L4 (salience-gate) / L5 (sleep)──► factStore
    Write-path: filling and maintaining the store. A SEPARATE mechanism, NOT cueMemory.
```

- **`cueMemory` originates from `factStore`** (stream A) — a projection of triggers. "One from the other"
  is preserved, but the direction is: `factStore → cueMemory` (not "fast → feeds the long-term").
- **The purpose of `cueMemory` is to wake up `factStore`** (stream B), not to fill it.
- **Filling `factStore`** is **consolidation** (stream C, L4+L5) from session telemetry
  (`activation_log`: what surfaced, whether it was used; the most valuable signal is a user repetition =
  push did not fire). This is a write-process, and it must not be confused with the fast mechanism.
- The origin of a fact from the episode that produced it is recorded by `fact_links.relation = derived_from`.
  The feedback loop: recall (B) writes to `activation_log` → consolidation (C) reinforces `strength`.

**Status in the MVP.** Stream B is implemented on degenerate data (symbolic L1 over `facts.jsonl`,
the familiarity flag, habituation). Stream A is trivial (cues live in the fact itself). Stream C (consolidation
L4/L5) — Priority 2, not yet present.

---

## 4. Names in code

Three registers, one pair of entities:
- **conceptual** (psycho-analogy) — "cue-primer / familiarity" and "long-term store / LTM";
- **operational** (code) — `cueMemory` (handle `cues`) and `factStore` (handle `facts`);
- **Russian** (speech, docs) — **«зацепки»** (cues) and **«свод»** (the store): "the cue fired on topic mysql → the fact
  was pulled from the store". It reads in the right direction: the cue catches → you dig into the store for the fact. By design
  the "store" does not "fire" (it is passive) — the "cue" fires. Alts: «зацепки» → «наводки»/«маяки» (leads/beacons);
  «свод» → «копилка»/«факты» (piggy-bank/facts).

```js
cues.match(context)   // phase-1: context × triggers → arm the priming (familiarity flag)
facts.get(hit)        // phase-2: recollection — retrieve the content
```

**Why `cue`, and not `flash`/`working`.** The essence of the fast layer is *priming*, not
"the state right now" (working memory) and not "an ephemeral log" (flash). `cue` is the native term of the concept
(L0, `fact_cues`, `cue_json`), literally a "distillate of triggers", and the stream `cue → fact` reads by itself.
Alternatives, if the action needs to be emphasized: `triggerMemory`/`triggers` (a trigger,
mapping onto the "cocked trigger" of the pain axis §10.3) or `primeMemory`/`prime` (the psycho-precise
"pre-setting for recall"). `coreMemory` for LTM **we do not take** — in MemGPT/Letta that is the opposite
(a small in-context) block; their analogue of our LTM is `archival memory`.

## 5. How to use it in development

- **In code/schema:** `cueMemory` / `factStore` (handles `cues` / `facts`). Carriers: `cueMemory` =
  `fact_cues` + SQLite mirror + flag (in the MVP — the `cues` field in `facts.jsonl`); `factStore` =
  `facts`/`fact_*` (in the MVP — `facts.jsonl`). `activation_log` is **consolidation telemetry**, not
  `cueMemory`.
- **"Priming / familiarity / phase-1"** — about `cueMemory` (the arming, the familiarity flag).
  **"Recollection / phase-2"** — about reading `factStore`. **"Consolidation"** — only about stream C
  (L4+L5, filling the store). Do not conflate the three streams.
- **Do not confuse** `cueMemory` with the LLM context window (that is the harness) and with `activation_log` (telemetry).
