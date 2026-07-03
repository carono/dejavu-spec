# dejavu — associative push-memory for AI agents

> **Specification repository.** This repo holds the conceptual, implementation-independent
> specification of **dejavu** — a pattern for giving an AI agent an *associative, push-driven*
> memory. It describes *what* and *why*, not a particular tech stack. Any implementation
> (any language, store, or agent harness) can be built from these documents.

*English is the canonical language of this spec. A full Russian mirror lives under
[`spec/ru/`](spec/ru/). See the [Русская версия](#русская-версия) section below.*

---

## What is dejavu?

Classical agent memory is **pull**: a knowledge store exists (vector DB, graph, files, a RAG
index) and the agent *itself decides* when to look inside — if it remembers to search at all.
The recurring failure is blunt:

> **"The memory is there, but the agent doesn't read it."**

The agent either doesn't know a relevant fact exists (so it never forms a query), or it economizes
turns and won't go look "just in case." Knowledge sits dead: the user repeats themselves, the agent
steps on the same rakes, and the environment (versions, paths, config quirks) is recalled unreliably.

**dejavu flips the initiative.** The agent does not request knowledge — **the environment injects**
the relevant fact into context at the moment it is needed, without an explicit act of intent. This
mirrors human recall: we remember by the cues of a situation (*cue-driven retrieval*), not because
we "decided to visit memory."

Push is a **layer on top of pull, not a replacement.** Deliberate search stays; push adds an
involuntary, ambient channel over it.

---

## Architecture

### Two mechanisms, three flows

The system is built from **two** memory mechanisms with distinct roles, and **three** flows between
them and the world.

| | **`factStore`** (the store, LTM) | **`cueMemory`** (cues, trigger-index) |
|---|---|---|
| Role | **content**: "what exactly" | **prompting**: "is it worth looking" |
| Psych analogue | long-term memory, recollection (phase 2) | familiarity / recognition (phase 1) |
| Carries | selected facts + links + trace strength | a cheap distillate of triggers |
| Cost | higher: read content, rank | no model/network, tens of ms |
| Horizon | cross-session, persistent | index persistent; "arming" lasts one turn |

```
(A) PROJECTION    factStore ──extract triggers──► cueMemory
    The LTM is distilled into a cheap, matchable index. Direction: factStore → cueMemory.

(B) PROMPTING     live context ──match──► cueMemory ──fire──► push into prompt ──► read factStore
    A cue fires → a fact is injected. This is the READ path of push-memory.

(C) CONSOLIDATION session transcripts ──gate / "sleep"──► factStore
    New facts enter the LTM. This is the WRITE path. A separate mechanism, NOT cueMemory.
```

In one sentence: `cueMemory` is a distillate of triggers **projected from** `factStore` (A); each
turn it is matched against context and, on a hit, **injects a fact** into the prompt (B); while
`factStore` itself is filled by a separate write process from transcripts (C). Memory becomes
**push** precisely because of `cueMemory`. Do not conflate the three flows — that is the most common
mistake when reasoning about the pattern.

### Six activation layers (L0–L5)

The read path (arming + delivery) runs on every harness event; the write path (encoding +
consolidation) runs offline.

```
harness event (prompt / pre-tool / post-tool / session-start)
        │
[L0] Cue extraction     cheap, no LLM: tokens, paths, containers, commands, error signatures, project_id
        │
[L1] Activation         symbolic exact-match  +  semantics (local embed)  +  spreading over fact_links
        │               score = relevance × strength(recency, freq) × link-decay
        │
[L2] Salience-gate      threshold · token budget · habituation (don't repeat surfaced) · dedup
        │
[L3] Delivery           additionalContext → system-reminder, explicitly low-authority "recalled: …"
        │
   ── agent works, the loop repeats ──
        │
[L4] Encoding (offline / session-end)   salience-gated: novelty, corrections, user repeats, env-discoveries
        │
[L5] Consolidation (periodic "sleep")   decay · prune · merge · Hebbian link reinforcement
```

- **L0–L3 are the read path** — the push proper: extract cues from the live event, activate
  candidate facts, gate them for noise, and deliver a small, low-authority hint.
- **L4–L5 are the write path** — the store is filled not by direct writes but by an offline process
  that distils candidates from session transcripts, then periodically decays, prunes, merges, and
  reinforces links ("sleep").

### Push mechanics — the core invariants

- **Event-driven.** Push fires at loop boundaries: session start, turn start, before an action,
  after a result, session end. It cannot reach *inside* one forward pass, but agent-loop boundaries
  arrive often enough that it feels like ambient background memory — a stroboscope, not continuity.
- **Symbolic cue-index first.** A discrete `{type, value, match-mode}` index (path, path-prefix,
  container, command, error signature, project, domain, keyword) is the **core of reliable
  environment recall** — an exact hit, not a probability. Graph spreading and vector similarity are
  fallbacks layered on top.
- **Budget.** At most N facts (start 3–5) and M tokens (start ~300–400) per event — a hard cap
  against context flooding.
- **Precision over recall.** A false positive actively harms every turn (it distracts the agent and
  burns tokens), so the salience-gate errs toward silence. A **pain axis** lowers the effective
  firing threshold for charged facts (a user repeat = "memory failed" = pain) and cools over time.
- **Habituation.** A fact already surfaced this session is suppressed unless it fires materially
  stronger.
- **Low-authority + fail-open.** Delivery is always marked as a possibly-stale background hint
  ("recalled: …", not "do: …"), and any failure of the channel silently injects nothing rather than
  breaking the agent's main work.

---

## The specification

Read in order — each document is implementation-independent.

| # | Document | What it covers |
|---|---|---|
| 01 | [`spec/01-concept.md`](spec/01-concept.md) | **The pattern.** Problem, the two entities & three flows, components, activation mechanics, fact life-cycle, harness integration, invariants, and what is unique. Start here. |
| 02 | [`spec/02-memory-tiers.md`](spec/02-memory-tiers.md) | **Naming & roles.** Pins the canonical names for the two mechanisms — `cueMemory` (the cue-primer) vs `factStore` (the long-term store) — a glossary + role model. |
| 03 | [`spec/03-activation-layers.md`](spec/03-activation-layers.md) | **The six layers in depth.** L0–L5 with concrete formulas (spreading activation, Ebbinghaus trace strength, lateral inhibition, sigmoid gate, pain axis) drawn from prior art. |

Russian originals: [`spec/ru/`](spec/ru/).

---

## What makes the pattern distinctive

1. **Push via a harness hook on every prompt** is the main differentiation. Surveyed analogues
   (agent memory systems, RAG frameworks, cognitive architectures) are all **pull**: the agent
   decides to search. This pattern makes injection the **harness's responsibility**, attacking the
   "memory exists but the agent won't read it" failure head-on.
2. **A symbolic cue-index over environment features** (path / command / error / container / project)
   as the primary recall channel — cognitively grounded content-addressable retrieval, where
   practical systems lead with vectors or graphs. Deterministic env-recall is the niche.
3. **One stack combining** symbolic cue-index + Hebbian link graph + vector fallback + offline
   "sleep" (reinforcement/decay) + salience-gate + pain axis, together with push delivery.
4. **A frequency term in trace strength** (recency × frequency), where the nearest analogues track
   recency only.

*Honest caveats — Hebbian link learning and sleep-consolidation are neuroscience-**inspired** design
hypotheses without a direct empirical validator in the surveyed literature; position them as
"inspired-by", not "proven". Index growth / context poisoning is a real risk, which is exactly why
the fact budget, salience-gate, habituation, and decay/archival are mandatory risk controls, not
ornaments.*

---

<a id="русская-версия"></a>
## Русская версия

> **Репозиторий спецификации.** Здесь лежит концептуальная, независимая от реализации спецификация
> **dejavu** — паттерна ассоциативной **push**-памяти для ИИ-агента. Документы описывают *что* и
> *почему*, а не конкретный стек: по ним можно построить реализацию на любом языке, хранилище и
> харнесе. Английская версия — каноническая; полное русское зеркало — в [`spec/ru/`](spec/ru/).

**Что такое dejavu.** Классическая память агента работает по модели **pull**: хранилище знаний есть,
но агент *сам решает*, когда в него заглянуть — если вообще догадается поискать. Отсюда типовой
диагноз: **«память есть, а агент её не читает»**. dejavu переворачивает инициативу — не агент
запрашивает знание, а **среда сама инжектирует** релевантный факт в контекст в момент нужды
(*cue-driven retrieval*, как у человека). Push — надстройка над pull, а не замена.

**Две сущности, три потока.** `factStore` («свод», LTM) — *что именно*; `cueMemory` («зацепки»,
триггер-индекс) — *стоит ли смотреть*. Потоки: **(A) проекция** `factStore → cueMemory` (дистилляция
триггеров в дешёвый индекс); **(B) побуждение** — контекст × `cueMemory` → инъекция факта в промпт
(read-путь); **(C) консолидация** — транскрипты сессий → `factStore` (write-путь, отдельный
механизм). Память становится push именно благодаря `cueMemory`.

**Шесть слоёв активации L0–L5.** L0 — извлечение зацепок (без LLM); L1 — активация (символьный
exact-match + семантика + spreading по `fact_links`); L2 — salience-gate (порог, бюджет,
habituation, дедуп); L3 — доставка (low-authority `additionalContext`); L4 — кодирование
(salience-gated, офлайн); L5 — консолидация («сон»: затухание, prune, merge, Hebbian). L0–L3 —
read-путь (сам push), L4–L5 — write-путь.

**Push-механика.** Событийность на границах цикла агента; символьный cue-индекс как ядро надёжного
env-recall; бюджет 3–5 фактов; precision важнее recall (болевая ось снижает порог для «болезненных»
фактов); habituation; low-authority доставка и fail-open (сбой памяти не ломает работу).

**Подробная спека** (русские оригиналы): [`spec/ru/01-concept.md`](spec/ru/01-concept.md) — паттерн;
[`spec/ru/02-memory-tiers.md`](spec/ru/02-memory-tiers.md) — имена двух механизмов;
[`spec/ru/03-activation-layers.md`](spec/ru/03-activation-layers.md) — 6 слоёв с формулами.
