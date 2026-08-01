# The unit of evolution: lesson, workflow, skill, or trajectory

_Findings for [issue #4](https://github.com/SamWongML/pi-evo/issues/4) · August 2026_

## 0. Recommendation, up front

**The atom stays the distilled lesson — a `WHEN <trigger> DO <guidance>` row — but v2's own document never argued for this, and two of its supporting claims about *why* (MUSE's numbers, and an invented "≥2–3 occurrences" rule attributed to MUSE) turn out not to match the papers they cite. This document rebuilds the argument from primary sources and corrects those citations. The conclusion doesn't change; the reasoning underneath it does.**

The four candidates are not four alternatives. They are four points on one distillation pipeline — raw trajectory → lesson → workflow/skill → code — and every primary source examined below either states this explicitly (CODESKILL's two skill "granularities," ACE's Generator→Reflector→Curator roles) or demonstrates it empirically (AWM's workflow *is* a lesson with steps; MUSE's skill body *is* a workflow with a routing header). What promotes an item from one tier to the next is not a fixed trajectory count — it is **whether the claim is discrete/verifiable or continuous/environmental**, a distinction the evidence-floor section below derives from three independent papers (MUSE, AWM, CODESKILL) making the same point in different vocabulary.

The lesson wins the atom-selection question specifically for pi-evo's operating point — 1–3 machines, single operator, zero session history at t=0 — for reasons that compound:

1. **It has the lowest evidence floor of the four.** A lesson can be created safely from a single verified failure→fix pair (exit code flips, error signature disappears). A workflow or skill needs either a test suite gating registration (MUSE) or a trained management policy (CODESKILL, 12,856 SFT examples + 500 GRPO steps + a frozen downstream policy for reward rollouts) — infrastructure pi-evo v0 does not have and cannot build from zero sessions.
2. **It is the only atom shaped to fit issue #5's credit-assignment mechanism.** #5 concluded that nothing statistical works at 50–300 sessions/week and that most individual entries see 0–5 injections/week; the only viable mechanism is a deterministic turn-level trace heuristic — did the agent's next action match *this one guidance string*, did the *next* outcome flip. That heuristic requires a single, atomic claim to check compliance against. A whole trajectory or a multi-step workflow doesn't have a single claim to check — "did the agent follow this" is exactly the fuzzy multi-step judgment that AgentProp-Bench measured at κ=0.049 (chance) for automated judges (cited and verified in #5). A lesson is the only candidate whose shape matches the only credit-assignment mechanism this system can actually run.
3. **It is the only atom shaped to fit issue #7's store.** #7 concluded embedded SQLite + `sqlite-vec` + FTS5, app-level RRF, a store in the low thousands of short structured rows. That is a lesson-shaped table (trigger/guidance/counters/scope), not a home for full trajectories (token-heavy, belongs in the trace archive) or SKILL.md packages (file-shaped, already have a home via Pi's native package system).
4. **Contrastive storage is real (MIA) but doesn't require a different atom** — it requires the lesson schema to allow a `kind='anti_pattern'` row and for retrieval to surface both `helpful` and `harmful`-leaning rows together, which is a one-column change, not a new object type.

Skills remain the correct **promotion target** — pi-evo already has T2 (Pi's native `SKILL.md` support) in its design, and that stays. What changes is *how* something becomes a skill: not single-trajectory auto-distillation gated by a weak sandbox check (which is exactly the path that produced MUSE's only catastrophic regression), but a human-reviewed promotion from a cluster of lessons, gated the same way the prior document's T4 promotion already is — PR review, never a direct commit.

---

## 1. Are these four genuine alternatives, or tiers of one pipeline?

Read as a menu, the four candidates look like a fork in the road. Read against primary sources, they are stops on one line:

| Stop | What it is | Who says so |
|---|---|---|
| Raw trajectory | The full session — every tool call, every observation | — |
| **Lesson** | One extracted claim: a trigger and a corrective/cautionary action | ACE's Reflector "extracts concrete lessons" from trajectories, which the Curator then turns into typed delta items ([arXiv 2510.04618](https://arxiv.org/abs/2510.04618), verified in the prior research pass — Generator/Reflector/Curator roles, brevity bias, context collapse, +10.6%/+8.6% figures confirmed) |
| **Workflow** | A named, multi-step procedure with a description, distilled from one or more trajectories | AWM: "workflows... structured routines distilled from past trajectories" ([arXiv 2409.07429](https://arxiv.org/abs/2409.07429), Wang, Mao, Fried & Neubig, ICML) |
| **Skill** | A workflow packaged with a routing header (name/description) for progressive disclosure, optionally with tests/scripts | MUSE: "each skill is represented as a markdown instruction file with a title, a trigger condition, and actionable instructions" ([arXiv 2605.27366](https://arxiv.org/abs/2605.27366)); CODESKILL: "each skill is represented as a markdown instruction file with a title, a trigger condition, and actionable instructions for the agent" ([arXiv 2605.25430](https://arxiv.org/abs/2605.25430)) — near-identical wording, independently arrived at |
| Code (T4) | The claim made mechanically enforced (lint rule, test, CI gate) | Not covered by this ticket; carried over from the prior document, still the best tier when reachable |

The strongest direct evidence for "tiers, not alternatives" is CODESKILL's own design: it maintains **two skill granularities in the same bank** — "event-driven skills" (local, one-trigger guidance, extractable from a single trajectory) and "task-level skills" (multi-step strategies, extracted from "trajectory evidence ranging from a single trajectory to a small set of related trajectories"). Its own ablation (Table 2) shows event-driven-only and task-level-only variants both beat the no-skill baseline on different task subsets, and **combining them beats either alone** (38.63 → 40.75 average pass rate moving from extraction-only to extraction+evolution, per CODESKILL Table 2 and §4.3) — direct evidence that the "narrow lesson" granularity and the "broad workflow" granularity are complementary layers of one system, not competing designs. An event-driven CODESKILL skill *is*, structurally, a `WHEN <trigger> DO <rule>` lesson; a task-level CODESKILL skill *is* an AWM workflow with a routing header. The paper doesn't distinguish four atom types — it distinguishes two granularities of one thing, and MUSE's five-stage lifecycle (creation → memory → management → evaluation → refinement) operates identically over both.

**What promotes an item from one tier to the next, per the primary sources:**

- **Lesson → workflow/skill:** MUSE and AWM both promote by *task-level recurrence* — when related trajectories cluster around the same multi-step procedure rather than a single reactive fix, extraction produces a task-level artifact instead of an event-driven one. CODESKILL's maintenance stage explicitly implements a `merge` operation that consolidates overlapping candidates into "a single stronger reusable skill" — this is the mechanical form of promotion.
- **Hypothesis → verified, within a tier:** gated by an *executable check* when one exists (MUSE blocks skill registration on `tests/` when the skill is code-backed) or by *independent recurrence* when it doesn't (MUSE falls back to "sandbox execution, source-trajectory checks, and runtime feedback as weaker but still useful validation signals" for text-only skills — precisely the gate that let the hvac-control regression through, see §3).
- **Workflow/skill → code (T4):** not covered by any paper surveyed here; remains the prior document's own (sound) argument that a mechanically-enforced check is strictly better than a request the model might ignore.

---

## 2. Distilled lesson — ACE

**Mechanism.** ACE (arXiv [2510.04618](https://arxiv.org/abs/2510.04618), Zhang et al., Stanford/SambaNova/UC Berkeley) separates three roles: a **Generator** runs tasks and produces trajectories exposing helpful and harmful moves; a **Reflector** critiques the trajectory and extracts concrete lessons in natural language; a **Curator** converts lessons into typed **delta items** with helpful/harmful counters, merged by deterministic non-LLM logic. Two update mechanisms: incremental delta updates (localized edits, never a monolithic rewrite) and grow-and-refine (append new, update in place, periodically dedupe by embedding).

**Retrieval / cost.** Lessons live in the always-injected context (a "playbook"), not a separate retrieval store in ACE's own design — but the atom itself (a short trigger/guidance pair) is exactly what makes retrieval cheap when it *is* externalized: fixed, small, and exact-match friendly.

**Evidence floor / creation.** ACE creates a lesson from **natural execution feedback on a single trajectory** — no labeled supervision, no multi-trajectory pool required to produce the first candidate. What ACE gates on is not occurrence count but **the merge step**: a delta item accumulates helpful/harmful counters over subsequent uses, and the two named failure modes are about *compression*, not *evidence volume* — **brevity bias** (compressing away the exact detail that made a lesson useful) and **context collapse** (iterative full rewrites eroding accumulated knowledge over many updates). Both are risks of *later* editing, not of *initial* creation from one trajectory.

**Updating / supersession.** A lesson is a mutable row with counters; ACE's non-LLM merge logic is precisely why lessons — unlike trajectories or workflows — have an unambiguous supersession semantics: two lessons that make the same claim about the same subject are a direct conflict a deterministic rule can detect (this is the basis of the prior document's `one_active_assertion` unique index, which this document keeps, see §9).

**Reported gains.** +10.6% on AppWorld agent tasks, +8.6% on finance reasoning, ~86.9% latency reduction vs. context-adaptation baselines (verified in the prior credit-assignment research pass this session builds on).

---

## 3. Whole trajectory / case-based reasoning — ExpeL, Agent KB

**ExpeL** (arXiv [2308.10144](https://arxiv.org/abs/2308.10144), Zhao et al., AAAI 2024) maintains two memory components: a pool of raw successful trajectories retrieved as few-shot exemplars by similarity, and a separately-maintained list of natural-language **insights** distilled by comparing successful and failed trajectory pairs. The two are reported as synergistic in ablation — insights alone and retrieved trajectories alone both underperform the combination. The structural point that matters for this ticket: **the trajectory-as-case has essentially no creation floor** (a single successful run is immediately a usable case for future few-shot retrieval), but the **insight layer** — the part of ExpeL that is closest to a "lesson" — is deliberately extracted by *pooling comparisons across the whole training task set*, not from any single trajectory. ExpeL's own design therefore already encodes the finding this document leans on: a raw case can be stored at N=1, but turning that case into a trustworthy general insight needs breadth of comparison, not a single instance.

**Agent KB** (arXiv [2507.06229](https://arxiv.org/abs/2507.06229), Tang et al., ICML 2025 Workshop, Oral — verified directly this session) aggregates trajectories into a structured, hierarchical knowledge base with **two-stage hybrid retrieval**: "planning seeds agents with cross-domain workflows, while feedback applies targeted diagnostic fixes." Verified reported gains: **up to 18.7 percentage points at pass@3 on GAIA with smolagents (55.2% → 73.9%)**, and **4.0pp on SWE-bench pass@1 with OpenHands (24.3% → 28.3%)**, with improvements "across all base model families" — genuine evidence for cross-agent transfer of stored experience, independent of MIA's contrastive-storage finding (§5). Agent KB's own materials do not state a minimum-trajectory threshold before an entry becomes trustworthy — the cold-start question this ticket asks about is simply not addressed by the paper, which is itself informative: **case-based systems in the literature are evaluated at scale (many accumulated trajectories) and offer no guidance for the zero-to-few regime pi-evo starts in.**

**Why not the atom for pi-evo, despite real evidence of value.** Three independent findings converge against making the trajectory the stored, retrievable unit at pi-evo's scale:

- **Token cost.** MIA's own baseline comparison (§5) found that raw-trajectory injection methods (RAG, Mem0, A-Mem) *underperform a no-memory baseline* — "long memory contexts introduce noise, leading to performance degradation" — while compressed workflow-summary methods (ReasoningBank, ExpeL, Memento, MIA) do not have this problem. This is a second, independent confirmation of the same point ACE's "brevity bias" names and the memory-taxonomy literature in the prior research document already flagged: raw episodic content degrades retrieval quality when injected directly.
- **False-precedent risk is structurally worse for a whole trajectory than a lesson.** The Databricks finding cited in the prior document (agents reusing notebooks from earlier *incorrect* runs with *increased* confidence) applies with more force to a trajectory than a lesson precisely because a trajectory reads as a complete, successful narrative — there is no explicit hedge in its shape the way `kind='hypothesis'`, `verified=0` is an explicit hedge on a lesson row.
- **The credit-assignment problem does not get easier for a trajectory — it gets harder.** #5 found the only viable v0 mechanism is a deterministic trace heuristic checking whether the agent's next action complied with *one* injected claim. A retrieved trajectory has no single claim to check compliance against; judging "did the agent appropriately adapt this case" is exactly the kind of trajectory-level judgment AgentProp-Bench measured automated judges at κ=0.049 (chance level) on, even with a 3-judge ensemble only reaching κ=0.432 (verified in #5's research). The atom that credit assignment *can* score should be the atom that gets stored and re-injected.

**Where trajectories still belong in pi-evo's design.** The prior document's own trace-archive design (S3, full session JSONL, queried by the curator and eval harness, not injected raw into live turns) is exactly the right home for cases. Nothing here argues against keeping raw sessions; it argues against making them the atom of the *retrieval* store issue #7 sized and shaped.

---

## 4. Induced workflow — Agent Workflow Memory (AWM)

**Mechanism.** AWM (arXiv [2409.07429](https://arxiv.org/abs/2409.07429), Wang, Mao, Fried & Neubig, ICML) induces **workflows** — named, reusable routines with a natural-language description and a sequence of steps abstracted away from task-specific values — and selectively injects them to guide future generations. It "flexibly applies to both offline and online scenarios, where agents induce workflows from training examples beforehand or from test queries on the fly."

**Reported gains (verified from the abstract).** 24.6% relative success-rate improvement on Mind2Web, 51.1% relative improvement on WebArena (plus a reduction in steps taken to solve tasks successfully), and cross-domain generalization "surpassing baselines from 8.9 to 14.0 absolute points."

**Evidence floor.** This is the paper's most load-bearing point for this ticket: **AWM's online mode can and does induce a workflow from a single trajectory**, immediately, with no occurrence-count gate. AWM's own discussion states plainly that this is a real risk, not a hypothetical one: online induction "induces workflows from model-predicted trajectories that are not always correct, thus can lead to incorrect workflows that degrade model performance." This is the *second* independent paper (after MUSE, §6) reporting that single-trajectory induction of a multi-step artifact carries a real regression risk — and it comes from a completely different research group, benchmark, and domain (web navigation, not coding), which is exactly the kind of independent replication that turns "MUSE had one bad case" into "this is a property of the atom, not a fluke of one paper's evaluation."

**Retrieval / updating.** Workflows are retrieved by task similarity and injected as guidance; AWM does not report a distinct supersession mechanism beyond ordinary memory accumulation — a gap the tiering argument in §1 addresses by routing AWM-style workflows through the same lesson-cluster-to-skill promotion path MUSE and CODESKILL already use.

**Why "workflow" collapses into "skill" for pi-evo's purposes.** Structurally, an AWM workflow (description + step sequence, distilled from trajectories, retrieved by similarity) is indistinguishable from a CODESKILL task-level skill or a MUSE skill body once it acquires a routing header. Pi already has a native mechanism for exactly this artifact (`SKILL.md`, progressive disclosure). There is no case, in the sources surveyed, for pi-evo maintaining a fourth, separate "workflow" object type distinct from the skill tier it already has.

---

## 5. Skill — MUSE-Autoskill, CODESKILL, Voyager lineage, Anthropic Agent Skills

**MUSE-Autoskill** (arXiv [2605.27366](https://arxiv.org/abs/2605.27366), Lin et al., ByteDance, verified via full-text fetch this session) is the most directly relevant source, because it reports both the strongest positive result for the skill atom and its most severe documented failure.

**Corrected numbers (the prior document's figures do not match the paper).** On SkillsBench's 75-task common set, self-created MUSE skills reach **85.24%** accuracy on the subset they successfully cover, against **81.17%** for human-authored skills on the same subset (MUSE Table 4) — the prior research document cites **87.94% vs. 68.40%**, which does not match. Cross-agent transfer into Hermes: MUSE-created skills raise Hermes from **37.24% → 51.90%** (a **+14.66pp** delta, MUSE Table 5) — the prior document cites both "+10.51pp" and, separately, Hermes baselines of "47.89% no-skills / 61.21% with human skills," neither of which appears in the paper (the actual Hermes figures are 37.24%/48.02%). The test-bundling statistic is close but not exact: MUSE-created skills ship `tests/` in **8.0%** of packages (Table 7), not the "9%" cited; human-authored skills are correctly cited at 0%. None of these corrections change the direction of the argument — self-created skills genuinely beat human-authored ones on this benchmark — but the specific figures should not be relied on as previously stated.

**Skill format.** MUSE explicitly mirrors Anthropic's Agent Skills format: a `SKILL.md` with YAML frontmatter (`name`, `description`) plus optional `scripts/`, `tests/`, `resources/`, `references/` subdirectories. Catalog routing loads only `name`+`description` eagerly (≈5–10K tokens for a 100-skill bank); the body loads on demand via a `read_skill`-style tool — confirmed both in MUSE's own schema appendix and independently against **Anthropic's own page** ([anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), fetched and verified this session: released Oct 16, 2025, open-standard announcement Dec 18, 2025, `github.com/anthropics/skills`) — this is the exact spec Pi supports natively.

**Evidence floor — the central finding for this ticket.** **Every one of MUSE's 47 covered-task skills was distilled from a single source trajectory** ("Each covered-task skill is distilled from a single source trajectory," MUSE Limitations). MUSE does **not** gate skill creation on an occurrence count at all — contrary to the prior research document's "Rule 2: never promote from one trajectory, require ≥2–3 independent occurrences," which is not what MUSE does and is not attributed to MUSE correctly. What MUSE actually gates on is **validation, not volume**: code-backed skills are blocked from registration until bundled `tests/` pass; text-only or untested code-backed skills fall back to "sandbox execution, source-trajectory checks, and runtime feedback as weaker but still useful validation signals."

**The hvac-control regression, and its general form.** MUSE's largest self-created regression — **80% → 20%** on a PI-control task — is explained precisely: "the source trajectory used a calibration window and gain-estimation routine specific to that simulator's noise profile; when re-applied in fresh runs, the variance in calibration data occasionally produces tuned gains outside the verifier's stability margin." This happened specifically in the weak-gate path (no generated tests for this skill), which is the only path available for a numerically-tuned procedure that can't easily be unit-tested for correctness across the input distribution.

Taken together with AWM's independent finding (§4) and CODESKILL's split between event-driven and task-level granularity (§1, §6), the **general form** of this result is:

> **A skill or workflow distilled from a single successful trajectory encodes whatever incidental values, environmental assumptions, or tuned parameters happened to hold in that one run. When the claim is discrete and verifiable — a command, a fixed instruction, a structural fact — a single trajectory plus an executable check (a passing test, a reproduced fix) is sufficient evidence, because the check substitutes for the missing repetition. When the claim is continuous or environment-specific — a tuned constant, a calibration routine, an assumption about scale or noise — a single trajectory cannot distinguish "this generalizes" from "this happened to work once," and no amount of code-review-style inspection catches it, because the failure only appears when the untested dimension shifts.**

This is not a hypothesis this document is proposing in isolation — it is the pattern three separate research groups converged on independently: MUSE's test-gate vs. sandbox-gate split, AWM's explicit warning about online single-trajectory induction, and CODESKILL's structural decision to require "a small set of related trajectories" specifically for its broader, more claim-heavy task-level granularity while allowing single-trajectory extraction for its narrower, more mechanical event-driven granularity.

**CODESKILL** (arXiv [2605.25430](https://arxiv.org/abs/2605.25430), Li et al., NTU/Zhejiang, verified via full-text fetch) confirms the reported headline numbers exactly as the prior document cited them: **+9.69** average pass rate over the no-skill baseline, **+4.01** over the strongest prompt-based or memory baseline, across EnvBench, SWE-Bench Verified, and Terminal-Bench 2. It formalizes skill-bank maintenance as a **learned** `add`/`merge`/`drop` policy trained with GRPO, using a hybrid reward of rubric-based LLM-judge quality plus verifiable execution improvement on a frozen downstream policy. Its own ablation: full lifecycle maintenance "slightly reduces the average pass rate by about 2%, but shrinks the skill bank from 1252 to 676 skills, nearly halving its size" — confirming the prior document's "bank growth is a bug" framing, but the mechanism behind it (a trained RL policy requiring 12,856 supervised examples plus 500 GRPO steps against a frozen downstream agent, ~210 GPU-hours per the paper's own Table 4) is **infrastructure pi-evo v0 cannot build.** This is the clearest illustration in the whole survey of the gap between "what wins at scale" and "what a solo operator starting from zero can run": CODESKILL's learned management policy is very likely the better long-run mechanism, and it is completely out of reach until there is a large, curated corpus of prior skill-management decisions to warm-start from — which does not exist at t=0.

**Voyager** (Wang et al., TMLR 2024, arXiv 2305.16291 — cross-confirmed via both MUSE's and CODESKILL's own bibliographies rather than independently re-fetched this session) is the lineage both papers cite as the seminal example: an ever-growing library of executable-code skills in Minecraft, with the same LLM authoring and refining skills against environment feedback. It is foundational to the "skill" framing but predates the modern lifecycle formalization (creation/memory/management/evaluation/refinement) that MUSE and CODESKILL both build on.

---

## 6. Contrastive storage — does it matter, and does it need a different atom?

**MIA** (arXiv [2604.04503](https://arxiv.org/abs/2604.04503), Qiao et al., verified via full-text fetch) confirms the claim precisely: with a Qwen2.5-VL-7B Executor, MIA achieves "an average gain of 31% across seven diverse datasets, notably outperforming its much larger counterpart, Qwen2.5-VL-32B, by **18%**." Its Memory Manager retrieves **both** "successful trajectories (positive paradigms) and failed trajectories (negative constraints)" for explicit contrastive context, scored by a hybrid of semantic similarity, a **value reward** (empirical success ratio $V\!al_i = s_i/(u_i+1)$) and a **frequency reward** ($Freq_i = 1/(u_i+1)$, deliberately favoring long-tail, less-used entries to avoid over-concentration). MIA also independently reproduces the finding that raw long-context memory hurts: baselines that inject retrieved content directly (RAG, Mem0, A-Mem) *underperform the no-memory baseline*, while methods that compress into structured summaries (ReasoningBank, ExpeL, Memento, MIA itself) do not have this problem — a second, independent confirmation of ACE's "brevity bias" and the "enlarging context made things worse" claim the prior document made.

**The honest caveat.** The 18% figure is for the *full* MIA system — a trained Planner-Executor architecture with two-stage RL (GRPO) plus test-time learning, not for contrastive storage in isolation. MIA's own ablation (Table 5/6 in the paper) shows contrastive memory alone ("Only Memory") actually *reduces* average multimodal accuracy by 0.4 points relative to the base agent; the gain only materializes once memory feeds a **trained** Planner (+3.5–4.15pp) and is compounded by RL training (+2.37/1.72pp) and test-time learning (+3.23/2.64pp). **Contrastive retrieval is a real, independently-replicated principle — retrieving both what worked and what didn't produces a materially better prior than success-only retrieval — but the specific 18% number is entangled with RL training infrastructure pi-evo does not have.** Attribute the number to the whole system, not to contrastive storage alone.

**Does this need a new atom?** No. The mechanism MIA describes — a single memory table with a success/failure label per row, both retrievable, scored contrastively — maps directly onto the lesson schema already in the design: a lesson with `kind='anti_pattern'` (or a `polarity` field) is a negative-constraint row exactly as MIA's negative trajectories are, at a fraction of the token cost, because a lesson is already compressed to `trigger`/`guidance` rather than a full trajectory. The change this finding requires is one column and one retrieval-query clause (surface both polarities, weighted), not a new table.

---

## 7. The evidence floor per atom type

| Atom | Minimum evidence to *create* (hypothesis status) | What raises confidence | Safe from N=1? | Primary source |
|---|---|---|---|---|
| **Distilled lesson** | 1 trajectory + one deterministic verification signal (exit code flips, error signature disappears, an explicit user correction) | A second independent occurrence, ideally a different session; an executable check | **Yes, if the claim is discrete/verifiable.** No, if it encodes a tuned numeric/environmental value — flag and require a second occurrence or a test | ACE ([2510.04618](https://arxiv.org/abs/2510.04618)) — lessons extracted per-trajectory from natural execution feedback, no labeled supervision required |
| **Trajectory/case** | 1 successful run (immediately retrievable as-is) | Nothing raises confidence in the stored case itself; only the *insight layer* distilled from many cases improves | Yes for storage; **no** for trustworthy *distillation* — insight extraction needs pooling across the full task set, not one run | ExpeL ([2308.10144](https://arxiv.org/abs/2308.10144)) — two-part memory, insights require cross-trajectory comparison |
| **Induced workflow** | 1 trajectory (AWM's online mode explicitly supports this) | More trajectories per workflow reduce the chance of an incorrect induction | **No** — AWM's own discussion states online single-trajectory induction "can lead to incorrect workflows that degrade model performance" | AWM ([2409.07429](https://arxiv.org/abs/2409.07429)) |
| **Skill (task-level / broad)** | "A single trajectory to a small set of related trajectories" (CODESKILL); MUSE uses N=1 in practice | An executable test suite gating registration (MUSE); a learned add/merge/drop policy (CODESKILL, requires 12,856 SFT examples + 500 GRPO steps + a frozen downstream policy) | **No** without a test gate — MUSE's hvac-control regressed 80%→20% from exactly this path (single trajectory, no generated tests, sandbox-check only) | MUSE ([2605.27366](https://arxiv.org/abs/2605.27366)), CODESKILL ([2605.25430](https://arxiv.org/abs/2605.25430)) |
| **Skill (event-driven / narrow)** | 1 trajectory | An executable check when the guidance is code-backed | **Yes** — narrow, reactive, mechanically checkable guidance is structurally closer to a lesson and doesn't carry the continuous-parameter risk | CODESKILL ([2605.25430](https://arxiv.org/abs/2605.25430)) |

**The v0-viability line, stated plainly.** Every atom in this table *can* be created from a single trajectory. What differs is what makes that single-trajectory creation *safe*, and three of the four candidates need machinery pi-evo v0 does not have to make it safe at any reasonable confidence: a benchmark-grade test harness (MUSE), a trained management policy (CODESKILL), or a large pooled comparison set (ExpeL's insights, AWM's workflow robustness). The lesson atom is the only one whose safety condition — a deterministic verification signal on the trace, already specified as the sole viable credit-assignment mechanism in issue #5 — is something pi-evo v0 already has a plan to build regardless of this ticket's outcome. That is the concrete form of "an atom needing hundreds of trajectories before producing anything useful is not viable for v0."

---

## 8. Schema sketch (SQLite-shaped, per issue #7)

This keeps the shape of the prior document's Postgres `lessons` table (the atom choice there was right even though the argument for it was missing) but ports it onto issue #7's conclusion — embedded SQLite + `sqlite-vec` + FTS5, no service — and folds in the corrections above: an explicit `contains_tunable_value` evidence-floor flag, a `polarity`/`kind` split for contrastive storage (§6), and `verified` tiers whose meaning is now argued from primary sources rather than asserted.

```sql
-- sqlite3 3.51+, better-sqlite3, sqlite-vec loaded via sqliteVec.load(db)

CREATE TABLE lessons (
  id                     TEXT PRIMARY KEY,                 -- uuid
  kind                   TEXT NOT NULL CHECK (kind IN
                            ('fix','anti_pattern','convention','tool_quirk','env','perf','correction')),
  polarity               TEXT NOT NULL DEFAULT 'positive'
                            CHECK (polarity IN ('positive','negative')),  -- MIA-style contrastive leg, §6

  -- CONTENT: the atom itself — one claim, one trigger, one action
  trigger                TEXT NOT NULL,     -- the symptom a future agent will observe
  guidance               TEXT NOT NULL,     -- the corrective action (positive) or the thing to avoid (negative)
  error_sig              TEXT,              -- normalized signature, exact-match retrieval leg

  -- SCOPE (kept flat; pi-evo v0 is 1-3 machines, not a multi-team fleet)
  scope_repo             TEXT,
  scope_lang             TEXT,
  scope_tool             TEXT,

  -- EVIDENCE FLOOR, made an explicit, queryable fact rather than an implicit assumption (§5, §7)
  contains_tunable_value INTEGER NOT NULL DEFAULT 0,   -- 1 if guidance embeds a numeric/env-specific constant
                                                        -- forces verified >= 1 before injection when set
  verified               INTEGER NOT NULL DEFAULT 0,   -- 0 hypothesis (N=1 + trace-verified)
                                                        -- 1 reproduced (>=2 independent sessions)
                                                        -- 2 test-gated (a CI check or unit test backs it)
  occurrences            INTEGER NOT NULL DEFAULT 1,
  distinct_sessions      INTEGER NOT NULL DEFAULT 1,

  -- PROVENANCE
  source_session         TEXT NOT NULL,
  source_git_sha         TEXT,
  derived_from           TEXT REFERENCES lessons(id),   -- supersession chain

  -- OUTCOME COUNTERS — set by the deterministic trace heuristic (issue #5's rank-1 mechanism),
  -- never by a statistical test. Read as a sequential/asymmetric decision rule, not a p-value.
  helpful_count          INTEGER NOT NULL DEFAULT 0,
  harmful_count          INTEGER NOT NULL DEFAULT 0,

  -- LIFECYCLE
  status                 TEXT NOT NULL DEFAULT 'active'
                            CHECK (status IN ('active','superseded','quarantined','retired')),
  superseded_by          TEXT REFERENCES lessons(id),
  promoted_to            TEXT,              -- 'skill:<slug>' | 'rule' | 'artifact:<pr-url>' | NULL — the T2/T4 exit

  created_at             TEXT NOT NULL DEFAULT (datetime('now')),
  last_seen_at           TEXT NOT NULL DEFAULT (datetime('now')),
  last_used_at           TEXT
);

-- lexical leg (BM25, native FTS5)
CREATE VIRTUAL TABLE lessons_fts USING fts5(
  trigger, guidance, scope_tool,
  content='lessons', content_rowid='rowid'
);

-- vector leg (sqlite-vec, exact brute-force at this row count — no ANN needed per issue #7's findings)
CREATE VIRTUAL TABLE lessons_vec USING vec0(
  lesson_rowid INTEGER PRIMARY KEY,
  embedding    FLOAT[768]
);

-- cheapest possible contradiction guard at 1-3-machine scale: one active claim per (repo, trigger)
CREATE UNIQUE INDEX one_active_claim
  ON lessons(scope_repo, trigger) WHERE status = 'active';

CREATE INDEX idx_lessons_error_sig ON lessons(error_sig) WHERE status = 'active';
CREATE INDEX idx_lessons_status    ON lessons(status, scope_repo);

-- append-only, written exclusively by the trace-derived heuristic from issue #5 — never a batch stat job
CREATE TABLE lesson_feedback (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  lesson_id   TEXT NOT NULL REFERENCES lessons(id),
  session_id  TEXT NOT NULL,
  signal      TEXT NOT NULL CHECK (signal IN ('injected','used','helpful','harmful')),
  evidence    TEXT,                          -- JSON: the matched command/error_sig pair
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);
CREATE INDEX idx_feedback_lesson ON lesson_feedback(lesson_id, created_at);

-- retrieval log for provenance / "which lesson caused this" debugging — cheap, per issue #7's ops findings
CREATE TABLE retrieval_log (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  session_id   TEXT NOT NULL,
  query_hash   TEXT NOT NULL,
  returned_ids TEXT NOT NULL,                -- JSON array of lesson ids
  created_at   TEXT NOT NULL DEFAULT (datetime('now'))
);
```

**What is deliberately *not* in this store, and why:**

- **No `trajectories` table.** Raw sessions stay in `~/.pi/agent/sessions/*.jsonl` and whatever trace archive the curator writes to; they are read by the curator to *produce* lesson rows, never injected raw into a live turn (§3).
- **No `skills` table with a body column.** Skills are `SKILL.md` files on disk, using Pi's native package/`resources_discover` mechanism — the format MUSE and CODESKILL both converged on independently, and the one Anthropic's spec already standardizes. A thin `promoted_to` pointer on the lesson row is enough to trace which lessons fed a given skill; the skill body itself doesn't belong in this database (§5, §9).
- **No `subject`/`predicate`/`object_value` triple decomposition.** That machinery in the prior Postgres schema was built for a multi-writer fleet's contradiction-detection problem (§1.7 of the prior document). At 1–3 machines with one writer, `one_active_claim` on `(scope_repo, trigger)` gets the same practical protection at a fraction of the schema complexity.

---

## 9. Citation verification report

Every arXiv ID this document relies on was fetched directly (abstract and, where the specific claim demanded it, full text) this session.

| ID | Claimed as | Resolves? | Notes |
|---|---|---|---|
| [2510.04618](https://arxiv.org/abs/2510.04618) | ACE | **Yes** | Zhang et al., Oct 2025 (rev. through Mar 2026). Generator/Reflector/Curator, brevity bias, context collapse, +10.6%/+8.6% figures — previously verified in this repo's credit-assignment research pass, re-relied-on here without re-fetching. |
| [2605.27366](https://arxiv.org/abs/2605.27366) | MUSE-Autoskill | **Yes, but the prior document's numbers do not match.** | Full text fetched. Correct figures: 85.24% self-created vs. 81.17% human on the covered subset (not 87.94%/68.40%); Hermes transfer 37.24%→51.90%, **+14.66pp** (not "+10.51pp," and the separately-cited "47.89%/61.21%" Hermes baselines do not appear anywhere in the paper — actual Hermes baselines are 37.24%/48.02%); test-bundling share is 8.0% for MUSE-created skills (not 9%), 0% for human-authored (correct). hvac-control 80%→20% regression, its explanation, and MUSE's actual gating mechanism (unit tests when code-backed, sandbox/runtime checks otherwise — **not** an occurrence-count threshold) all confirmed directly from the paper's text. |
| [2605.25430](https://arxiv.org/abs/2605.25430) | CODESKILL | **Yes, matches exactly.** | Full text fetched. +9.69 / +4.01 pass-rate figures, "small set of related trajectories" for task-level extraction vs. single-trajectory event-driven extraction, learned RL add/merge/drop policy (12,856 SFT examples, 500 GRPO steps, ~210 GPU-hours per Table 4), and the 1252→676 skill-bank-halving ablation all confirmed directly from the paper's text. |
| [2604.04503](https://arxiv.org/abs/2604.04503) | MIA | **Yes, matches exactly.** | Full text fetched. "Outperforming its much larger counterpart, Qwen2.5-VL-32B, by 18%" is a direct quote. Contrastive success/failure retrieval, the value/frequency reward formulas, and the "Only Memory" ablation showing a −0.4pp effect in isolation (i.e., the 18% figure belongs to the full trained system, not contrastive storage alone) all confirmed directly. |
| [2507.06229](https://arxiv.org/abs/2507.06229) | Agent KB | **Yes.** | Tang et al., ICML 2025 Workshop, Oral. Abstract-level fetch confirmed 18.7pp GAIA gain (55.2%→73.9%, smolagents, pass@3) and 4.0pp SWE-bench gain (24.3%→28.3%, OpenHands, pass@1), and the two-stage planning/feedback retrieval architecture. The paper does not state a minimum-trajectory threshold for entry trustworthiness — noted as a gap, not filled in with an invented number. |
| [2409.07429](https://arxiv.org/abs/2409.07429) | Agent Workflow Memory (AWM) | **Yes.** | Wang, Mao, Fried & Neubig, ICML. Abstract-level fetch confirmed 24.6% relative (Mind2Web) / 51.1% relative (WebArena) success-rate gains and 8.9–14.0pp absolute cross-domain generalization gains, plus offline/online induction modes and the explicit statement that online single-trajectory induction "can lead to incorrect workflows that degrade model performance." |
| [2308.10144](https://arxiv.org/abs/2308.10144) | ExpeL | **Yes.** | Zhao et al., AAAI 2024. Two-component memory (insights + trajectory pool) and the insights-require-cross-trajectory-comparison structure are corroborated both directly and via consistent characterization in three independently-fetched papers that cite it (MIA's, MUSE's, and CODESKILL's own related-work sections). No specific accuracy numbers from ExpeL are relied on in this document, precisely because this session's direct extraction of them could not be independently re-confirmed with full confidence — a deliberate abstention rather than an unverified claim. |
| — | Anthropic Agent Skills spec | **Yes.** | [anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills), fetched directly this session. Confirms `SKILL.md` + YAML frontmatter (`name`/`description`), the three-tier progressive-disclosure mechanism, and release dates (Oct 16, 2025 blog post; Dec 18, 2025 open-standard announcement) — consistent with both MUSE's and CODESKILL's own citations of it. |
| — | Voyager | **Yes (not independently re-fetched this session).** | Wang et al., TMLR 2024, arXiv 2305.16291. Cross-confirmed identically by two independently-fetched primary sources (MUSE's and CODESKILL's own bibliographies), both describing it the same way: an ever-growing executable-code skill library in Minecraft with self-verification. Treated as reliable on that basis rather than re-fetched directly, given the effort budget for this pass. |

**No citation relied upon above failed to resolve.** All four arXiv IDs the issue named explicitly (`2605.27366` MUSE, `2605.25430` CODESKILL, `2604.04503` MIA, `2507.06229` Agent KB) exist, are genuine, and were checked against their full text or abstract this session — the caution about `26xx`-prefixed 2026 dates in the issue was warranted as a general practice, but every one of these specific IDs is real. The material finding is not a fake citation; it is that **the prior research document's specific numbers for MUSE were transcribed incorrectly** in three places (the self-vs-human percentages, the Hermes transfer delta, and the Hermes absolute baselines), and that its "Rule 2" (requiring ≥2–3 occurrences before promoting a skill) was presented as if it were MUSE's own practice when MUSE's actual practice is single-trajectory creation gated by validation, not volume.

---

## 10. What this recommendation gives up

- **A richer per-entry narrative.** A lesson row is a compressed claim; it cannot carry the texture of a full trajectory the way ExpeL's case retrieval can. This is deliberate — that texture is exactly what the token-cost and false-precedent evidence in §3 argues against re-injecting live, and it remains available in the trace archive for the curator and for audits.
- **Automatic multi-step procedure capture.** A single lesson can't represent a workflow. This is why the promotion path to the skill tier (§1) has to exist and has to work, not just remain a diagram — pi-evo's curator needs to actually notice when a cluster of related lessons should become a `SKILL.md`, or the system will accumulate narrow fixes and never capture the procedural knowledge MUSE and CODESKILL both show is worth more per token.
- **The performance ceiling CODESKILL's learned management policy demonstrates.** At scale, a trained add/merge/drop policy beats fixed-prompt heuristics by a real margin (+4.01pp over the best prompt-based baseline). Pi-evo v0 cannot build this from zero sessions, and this document does not claim it never should — only that it is a Phase-2-or-later optimization, not a v0 requirement, exactly as the ticket's framing anticipated.

---

## 11. The strongest argument against this recommendation

The lesson atom's biggest structural weakness, argued honestly: **it is the atom that generalizes worst per token spent creating it.** MUSE's own headline result is that *skills* — not lessons — beat human-authored baselines by double digits, and CODESKILL shows a learned skill-management policy beating fixed-prompt lesson-style extraction by 4pp. If pi-evo's real bottleneck turns out to be *procedural* knowledge (multi-step repo conventions, not single-command fixes), betting the whole schema on the atom with the lowest per-item value could mean building a system that's cheap to validate but caps out well below what a skill-centric design would reach once there's enough data to support one safely. The mitigating case is that this is exactly why the promotion path matters as much as the atom choice — but if the curator's promotion logic never actually gets built (a real risk; it's the part of every one of these systems that needs the most judgment and the least automation), pi-evo ends up with a pile of narrow fixes and none of the procedural leverage the primary sources show is where most of the value actually is.
