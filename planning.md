# How the DSPy tech-share deck was built

A process record of one piece of work: turning "explain DSPy" into a four-slide
deck grounded in a real experiment.

This is written so a **fresh session** with no prior context can replay the method.
It is not a plan for new software. It is the sequence that produced the artifact, the
decisions that shaped it, and the failures that had to be corrected along the way.

---

## The pipeline

```
  framing          research           code               run            deck          verify
  ────────         ────────           ────               ───            ────          ──────
  what is        read DSPy 3.5     build the         GEPA over       single-file    syntax, DOM,
  the task?      source, not       measurement       the rig:        HTML deck      self-contain,
  what proves    blog posts        rig               trainset /      generated      rendered
  a prompt                         (catalogue,        val / test      from data      output
  improved?                        cases, metric)                                   
       │                │                │                │               │              │
       ▼                ▼                ▼                ▼               ▼              ▼
   one sentence    verified facts   89 instruments   0.635 → 0.767   4 slides,      screenshots
   + a metric      + corrected       + 30 cases       (2 of 14       3 widgets     by a human
                   claims           + 16 gates       kept)
```

Each phase gates the next. Nothing in the deck was hand-typed from memory — every
number traces to a file the previous phase produced.

---

## Phase 0 — Framing

**Decision:** do not write a deck about "what DSPy is". Write a deck about one
experiment that answers a question the audience actually has: *can a prompt be
systematically improved, or is prompt tuning just vibes?*

**The task chosen:** given an equity holding, pick three defensible benchmark
references. Named after a concrete failure the audience recognises:

> NVDA should get AMD or TSMC — not Salesforce.

**What made it work:** the task has a *checkable* answer, the answer is *arguable*
(so the metric has to encode judgement), and the wrong answer is *plausible*
(software is also "technology", which is exactly the trap).

**Output:** one sentence defining the task, one sentence defining the near-miss.

---

## Phase 1 — Research

**Rule applied:** read the installed library source, not documentation or blog posts.
Every claim that reached a slide was checked against a file.

Files read:

| file | what it settled |
|---|---|
| `dspy/teleprompt/mipro_optimizer_v2.py` | where minibatches come from; when full evaluation fires; how candidate counts are set |
| `gepa/optimize_anything.py` | the reflection prompt template; candidate selection strategy |

**Facts extracted, and the corrections they forced:**

| first assumption | what the source said | consequence |
|---|---|---|
| "36 combinations at `light`" | `num_instruct_candidates = n // 2` when demos are optimised | 3 instructions × 6 demo sets = **18**, not 36 |
| "minibatches come from the training set" | `eval_candidate_program(batch_size, valset, ...)` | every score in MIPROv2 is on the **validation** set |
| "it promotes the top candidate" | one candidate per full-eval step, every `minibatch_full_eval_steps + 1` trials | a *checkpoint*, not a stage |
| "the child replaces the parent" | Pareto frontier; a child joins if it wins on **any** case | lineage branches; specialists survive |
| "reflection rewrites the prompt" | template carries `<curr_param>` + `<side_info>` | it **edits** the parent's wording; it does not start over |

Numbers that mattered: `num_trials = max(2 · vars · log₂(N), 1.5 · N)` → 10 trials
at `light` with one predictor, so 3 full evaluations. `minibatch` mode only engages
above 50 validation cases.

**Output:** a fact table, each row with a file and line behind it.

---

## Phase 2 — Code (the measurement rig)

The rig exists to make "is this benchmark defensible?" a number.

Four layers, built in this order:

| layer | module | what it holds |
|---|---|---|
| **data** | `peerlens/pack.py` | 89 instruments; 30 labelled assets |
| **labels** | `peerlens/labeling.py` | three independent signals (taxonomy, market-orthogonalised co-movement, description similarity), 2-of-3 voting, a conflict queue |
| **arithmetic** | `peerlens/metric.py` | the scoring function, pure, no DSPy import |
| **binding** | `run_teach.py` | the adapter that turns the metric into `(score, feedback)` for the optimiser |

**The four tiers** — the single most important design choice:

| tier | credit | why |
|---|---|---|
| peer | +1.0 | same industry, defensible |
| adjacent | +0.5 | credible, different exposure |
| hard negative | −1.5 | sounds right, is wrong — software for a chip company |
| unrelated | −1.5 | obviously wrong |

```
credit  = (peers · 1.0 + adjacent · 0.5) / 3
penalty = 1.5 · (wrong / 3)
score   = clamp01(credit − penalty)  × 0.8 if three funds on one industry
         = 0.0 if wrong count / unknown id / repeated id
```

**Why the hard negative costs 1.5:** a wrong benchmark quietly misleads a review; a
weak one merely disappoints. That is a choice, not a derivation — so the deck shows a
slider that re-scores the same answer at penalty 0.5 and 3.0.

**16 of 22 prompt rules became code.** Schema (the field does not exist), validator
(checked after generation), or Python (computed, not written by the model). The
remaining 6 are the docstring the optimiser may rewrite. That split is what makes
optimisation safe: the optimiser cannot break the contract, only improve the prose.

**Output:** a rig that scores any answer, plus 79 tests.

---

## Phase 3 — Run the optimiser

Split, decided before the run:

| split | n | purpose |
|---|---|---|
| trainset | 9 | the optimiser rewrites against these |
| validation | 3 | used to choose between candidates |
| **test** | **10** | **never touched by the optimiser** |

The test set is the whole point. If it were used for ranking, the reported score
would belong to the winner that the same test set chose.

Run: `GEPA`, `max_metric_calls=120`, `use_merge=False`, `track_stats=True`.

Result: **21 iterations, 14 proposals, 2 kept.** Baseline 0.635 → best 0.767 on the
ten held-out cases. The gain is concentrated: three cases reached a perfect score,
seven did not move.

**Output:** `leaderboard.json` with 15 scored versions, each carrying its own
per-case array.

---

## Phase 4 — The deck

**Approach:** a generator, not hand-written HTML. `build_deck.py` emits one
self-contained file — inline CSS, inline JS, an SVG sprite, no network, no storage.

**Why generate:** the numbers come from `leaderboard.json` and `baseline.md` at build
time. The deck cannot drift from the experiment, because it is built from it.

**Four slides, each answering one question:**

1. **What DSPy is** — six concept cards (click for code + real output), the trade, when to use it
2. **The setup** — the three inputs (catalogue, labelled cases, metric), the split, the formula
3. **How the optimiser works** — the loop diagram, MIPROv2's checkpoint funnel, GEPA's five loops
4. **The optimiser at work** — one iteration in five stages, every version, v0 vs best

**Interactive elements, each chosen because a static image would be weaker:**

| element | what it demonstrates |
|---|---|
| concept modal | code and real output side by side |
| score calculator | click three ids, watch the real metric recompute |
| MIPROv2 trial log | one candidate scoring 1.00 then 0.33 on different batches — noise, visible |
| GEPA five loops | two accepted, three rejected, with the metric's sentence driving each edit |
| anatomy stepper | the reflection prompt as the model actually receives it |

**Output:** `deck.html` (~250 KB) plus `deck_nojs.html`, a script-free edition.

---

## Phase 5 — Verification

| check | what it catches |
|---|---|
| `python -m pytest` | the metric arithmetic, 79 tests |
| `node --check` on the inline JS | syntax errors before they reach a browser |
| DOM shims in Node | widgets that render empty |
| `audit_deck.py` | external references, storage APIs — proves the file is offline |
| **a human looking at it** | **everything else** |

The last row is the honest one. The shims could not see layout. Three defects were
caught only by screenshot: a `max-width:1100px` breakpoint collapsing two-column
layouts at 1038px, a CSS escape rendering as `003`, and a counter stuck at `0`
because its animation only ran on one slide.

---

## What went wrong

Recorded because it is the most transferable part.

| failure | cost | fix |
|---|---|---|
| DeepSeek rejects `response_format` | `JSONAdapter` unusable | `ChatAdapter(use_json_adapter_fallback=False)` |
| `XMLAdapter` parse failures | 23% of cases | Chat adapter → 3% |
| Strict gates zeroed every answer | metric had no gradient | split fatal (→0) from soft (−0.12) |
| `ValueError: No LM is loaded` | all 392 metric calls failed instantly | wrap `optimizer.compile` in `dspy.context(lm=...)` |
| Reflection proposed nothing ×135 | `InstructionProposer` defaulted to `JSONAdapter` | pass it the chat adapter |
| Validator filtered `None` from industries | "one peer + two broad indices" flagged as a duplicate; −0.12 per case | require all three to carry an industry |
| CSS emitted into the JS block (twice) | widgets dead, JS dead | anchor on a unique marker, verify the CSS/JS split |
| Inverted slice in a splice | ~1.3 KB of JS duplicated | caught by a printed size of `0 chars` |
| Splicing a page deleted helper variables | build failed | splice to the block's own end, not the next block's start |
| Breakpoint at 1100px | every two-column layout collapsed on a laptop | lowered to 860px |
| `grow()` only ran on slide 2 | page 3 showed a score of `0` | static fallback values + run on both slides |
| PowerShell body splices | mojibake (`Ã‚·`, `a†’`) | escape-only repair table, ASCII-only new text |
| **scope drift** | asked to replace one section, deleted four | restate the change and what is *not* changing, before editing |

---

## What I would do differently

1. **Template the generator.** All the splice bugs share one cause: editing a 2,500-line
   f-string builder by string replacement. Named slots per slide would have eliminated
   the entire category.
2. **Git from the first day.** Twice the build broke and a human found it first.
3. **Test the generator, not just its output.** The output was verified a dozen ways;
   the thing producing it was verified zero times.
4. **Verify in a real browser.** A DOM shim cannot see a breakpoint, a font, or a
   colour. This was the single largest gap, and screenshots closed it instantly.
5. **Write the plan down before big edits.** Every scope-drift incident was one
   sentence of restatement away from not happening.
