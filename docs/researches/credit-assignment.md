# Credit Assignment: Proving an Injected Lesson Helped

_Research findings · answers GitHub issue [#5](https://github.com/SamWongML/pi-evo/issues/5) · July 2026_

> Scope note: issue #5 fixes the operating point at **1–3 machines, single operator** — a much
> smaller volume than the fleet-of-many-agents design in `docs/researches/self-evolving-pi-agents.md`
> (v2), which put `helpful_count`/`harmful_count` on the `lessons` table and used them in both the
> retrieval ranking function (§5.1) and the pruning query (§7.3: `UPDATE lessons SET
> status='quarantined' WHERE harmful_count >= 3`) without ever specifying what increments them.
> `lesson_feedback.signal` in that schema already anticipates the values `injected | used | helpful |
> harmful | contradicted` — this document is about what actually writes `helpful` and `harmful`,
> and whether it can be trusted at this volume.

## 0. The volume assumption, stated up front

Every mechanism below is judged against a concrete number, not a vague "not enough data." At 1–3
machines run by one operator, a generous estimate is **50–300 agent sessions/week** fleet-wide
(the high end assumes heavy batch/headless use across all three machines; the low end assumes
mostly interactive, one-machine-at-a-time use). That is the *ceiling* on total observations.

It is **not** the per-lesson number. The retrieval gateway design in the prior document targets a
15–40% turn-level hit rate spread across what will realistically be hundreds of distinct lesson
rows once the store has run for a few months (its own pruning logic caps the active bank at 1,500
rows). Divide 50–300 sessions/week across a few hundred lessons and most individual lessons get
**0–5 injections a week**; only the handful of lessons tied to your single most common recurring
error will see double digits. This split — cheap in aggregate, starved per-item — is the whole
shape of the problem and is why every mechanism below is scored twice: once as "how many *total*
observations does the method need" and once as "how long until *this one lesson* has enough."

## 1. Mechanisms surveyed

### 1.1 Turn-level behavioural attribution (compliance + adjacent outcome)

**Mechanism.** No LLM call. At `before_agent_start` Pi's recall hook already knows which lesson IDs
were injected this turn (this is exactly the `retrieval_log.returned_ids` table already in the
prior schema). A capture-side extension then watches the next `tool_call`/`tool_result` events in
the same session and does a deterministic string/structural match: did `event.input` contain the
`blocked_command` or a normalized form of the `guidance` field of an injected lesson? Did the
`tool_result` that followed have `isError: false` / `exitCode: 0` where the prior attempt on the
same target had failed? This is pure trace parsing over fields the prior document's own
`evolve-capture.ts` (§7.1) already extracts (`isError`, `details.exitCode`, `event.input`).

**Cost.** Effectively zero — no model call, no network round-trip, runs synchronously inside the
extension that already exists for capture.

**Runs needed to be trustworthy.** N=1 per signal — each injected-lesson event immediately produces
one (used / not-used, then succeeded / then failed) data point. There is no waiting for a batch.
What it cannot do at N=1 is distinguish causation from coincidence — the agent might have fixed the
bug the same way regardless of the injected lesson. Treat it as a **compliance heuristic feeding a
threshold rule**, not a p-value: e.g., 2–3 consecutive `harmful` signals on one lesson triggers
quarantine (this is literally the `harmful_count >= 3` clause already in the prior document's
pruning SQL, §7.3) — a fixed small-count rule is defensible here precisely because false positives
are cheap to reverse (a quarantined lesson can be reinstated) and false negatives compound (an
active harmful lesson keeps getting retrieved).

**Confidence supported.** Directional only. Good enough to drive a binary keep/quarantine decision.
Not sufficient to claim "this lesson improved success rate by X points" — that claim requires one
of the counterfactual mechanisms below.

**Relation to LRAT.** This is the non-parametric analogue of the signal LRAT (arXiv
[2604.04949](https://arxiv.org/abs/2604.04949)) exploits — see §2 for why LRAT's actual machinery
does not transfer but its underlying idea (behaviour-after-seeing-the-candidate is a better signal
than a static judgment) does, and this is how to capture it without a training pipeline.

### 1.2 Full task-level counterfactual A/B

**Mechanism.** Replay the same task with the lesson present vs. absent (Pi already supports this:
`--no-extensions`/current-store/candidate-store conditions per the prior document's own eval
harness, §9.1), repeated across several task instances that are known to trip the lesson's trigger
condition, comparing pass/fail rates.

**Cost.** High. Each "test" of one lesson is not one run but a repeated experiment.

**Runs needed to be trustworthy — computed, not assumed.** For a binary (pass/fail) outcome, the
standard two-proportion power calculation (z_{α/2}=1.96, z_β=0.84, α=0.05, power=0.80) gives:

| True effect of the lesson | Runs needed **per arm** | Total runs for one lesson |
| --- | --- | --- |
| 30 pp lift (e.g. 50%→80%) | ≈36 | ≈72 |
| 20 pp lift (e.g. 50%→70%) | ≈91 | ≈182 |
| 10 pp lift (e.g. 50%→60%) | ≈384 | ≈768 |
| 5 pp lift (e.g. 50%→55%) | ≈1,479 | ≈2,958 |

The prior document itself already surfaced the small-N end of this problem: it notes MUSE-Autoskill
(arXiv [2605.27366](https://arxiv.org/abs/2605.27366), verified to exist) ran evaluation cases at
**N=5** per condition and still reported wide confidence intervals on a binary-reward task — fully
consistent with the table above, where even the largest, easiest-to-detect 30pp effect needs 36 per
arm, not 5.

Against the §0 budget (50–300 sessions/week, **fleet-wide**, across every lesson and every ordinary
work session, not dedicated to testing), spending 72–2,958 runs testing one single lesson is not
something a solo operator can do routinely. It is affordable only as an **audit**, not a per-entry
mechanism: pick the 3–5 most-used or most-suspect (`harmful_count > 0`) lessons once a month and
spend a real compute budget confirming or refuting them, accepting that only gross (≥20–30pp)
effects are catchable and a "no significant difference" result is inconclusive, not exonerating.

**Confidence supported.** The only mechanism here that can support an actual causal claim
("this lesson caused a Y-point change"). Reserve it for exactly that reason — spend it on the few
lessons where a wrong call is expensive.

### 1.3 LLM-as-judge over the trajectory

**Mechanism.** During the nightly Reflect pass (already an LLM call per session in the prior
document's `pi -p` reflector, §7.4), ask the judge model, given the full trajectory and the
specific injected lesson content, to label each lesson `helpful | harmful | neutral | not_used`
with a short justification.

**Cost.** Near-zero marginal cost if piggybacked on the Reflect step that already runs; one extra
structured output field, not an extra pass.

**Runs needed to be trustworthy.** N=1 — it operates per-session, no repetition needed. The caveat
is in *reliability*, not volume. General chat-preference judging is well calibrated: Zheng et al.,
"Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena," arXiv
[2306.05685](https://arxiv.org/abs/2306.05685), report GPT-4-judge agreement with humans around
85%, exceeding human-human agreement (~81%), but that number is for open-ended chat preference, not
tool-using agent trajectories. For the latter, AgentProp-Bench (arXiv
[2604.16706](https://arxiv.org/abs/2604.16706), Gurram, 2026) is directly on point: substring-based
automated judging of agent trajectories scored **κ=0.049 — chance level** — against human labels,
and even a **3-judge ensemble only reached κ=0.432**, described in the paper as moderate agreement
at best. That is the realistic ceiling for "did this trajectory use the lesson well," not the
85%-agreement number from chat evaluation.

**Confidence supported.** Moderate — useful as a corroborating vote combined with §1.1's
deterministic signal, not as a standalone ground truth. A sensible rule: escalate to the more
expensive mechanisms (§1.2 or §1.4) only when the judge and the trace-derived heuristic disagree.

### 1.4 Local/targeted counterfactual ablation (not in the original issue list, added because it changes the cost picture)

**Mechanism.** Rather than re-running the whole task, mask *just the one injected lesson* out of the
context at the single decision point where it mattered and regenerate only that step, comparing the
result with vs. without. This is exactly the method in ContextCite (Cohen-Wang, Shah, Georgiev,
Madry, "ContextCite: Attributing Model Generation to Context," arXiv
[2409.00729](https://arxiv.org/abs/2409.00729), NeurIPS 2024): it fits a lightweight surrogate model
over a budget of context ablations to attribute a specific generated statement to specific context
sources, and the paper explicitly lists **detecting poisoning attacks** as one of its three worked
applications. A closely related idea purpose-built for RAG poisoning is RAGCharacter / Needle-in-RAG
(Cui & Liu, arXiv [2605.01782](https://arxiv.org/abs/2605.01782), 2026), which re-enters a *flagged*
trace and performs "budgeted counterfactual masking and replay" to localize which retrieved span
caused one concrete bad generation, at the character level rather than the whole-passage level.

**Cost.** Cheap relative to §1.2 — a handful of extra model calls at one decision point, not dozens
of full-task reruns — but it needs harness plumbing ContextCite's own implementation relies on
(regeneration under controlled ablation, and ideally access to output log-probabilities to fit the
attribution surrogate well) that Pi does not currently expose, and neither paper has been validated
on a tool-using coding-agent trajectory — both target retrieval-augmented *question answering*. Rank
this as a **v1, not v0** mechanism: promising, cheap enough to be worth building once the harness
plumbing exists, but currently unverified in this exact setting.

**Confidence supported.** Tells you the lesson was causally load-bearing for one specific decision —
it does not by itself say whether that decision was good or bad. Pair it with §1.1/§1.3 for the
outcome judgment. Best used selectively — triggered by a disagreement or a flagged bad outcome, not
run on every session.

### 1.5 Agent self-report

**Mechanism.** Have the agent state, at `agent_settled`, which retrieved lessons it used and whether
they helped.

**Cost.** Near-zero — no extra call if folded into the existing closing turn.

**Runs needed.** N=1, immediately available. The problem is not volume, it is trust: Turpin,
Michael, Perez & Bowman, "Language Models Don't Always Say What They Think: Unfaithful Explanations
in Chain-of-Thought Prompting," arXiv [2305.04388](https://arxiv.org/abs/2305.04388), show model
self-explanations can be systematically unfaithful to the actual causal driver of a decision —
accuracy on stated reasoning dropped by up to 36 points on a 13-task suite when a known biasing
feature was present but never mentioned in the explanation. A model reporting "lesson X helped" is
not good evidence that it did; self-report also has the ordinary sycophancy risk of rating whatever
was shown to it as helpful by default.

**Confidence supported.** Low, standalone. Use only as a cheap prefilter (skip the LLM-judge call
when self-report and the deterministic trace heuristic already agree; only pay for §1.3 or §1.4 when
they disagree).

### 1.6 Run manifests + after-the-fact statistical attribution

**Mechanism.** Log, for every turn, exactly which lesson IDs were in context (the prior document's
`retrieval_log` table already does this) and the eventual outcome of the session/task. Compute,
per lesson, something like "success rate of sessions where lesson X was present" vs. baseline —
a co-occurrence counter, which is precisely what `helpful_count`/`harmful_count` already imply.

**This exact mechanism has been tried and its sample complexity measured.** "When to Forget: A
Memory Governance Primitive" (Simsek, arXiv [2604.12007](https://arxiv.org/abs/2604.12007), 2026)
proposes "Memory Worth" (MW) — a two-counter, success/failure co-occurrence tracker per memory item
that provably converges to `p+(m)`, the probability of task success given that memory `m` is
retrieved. The paper's own empirical validation needed **10,000 episodes** in a controlled setting
to reach ρ=0.89 correlation with ground truth, and **3,000 episodes** in a text-retrieval setting
before stale memories reliably crossed its low-value threshold. And even at that volume, the paper
is explicit about the ceiling: MW "measures outcome co-occurrence rather than causal contribution,"
i.e. it cannot on its own distinguish a memory that *caused* success from one that merely
*correlated* with it (e.g. a lesson that only ever gets retrieved on already-easy tasks will look
falsely helpful; one retrieved mostly on hard tasks will look falsely harmful).

Set that 3,000–10,000-episode convergence bar against §0's 50–300 sessions/week ceiling: **even
treating the entire lesson store as one aggregate counter — not per lesson — takes roughly 10–60
weeks (2.5–14 months) just to reach the volume this specific published mechanism needed to
converge**, and that is before per-lesson granularity, where the true number is worse by roughly the
lesson-count factor (§0).

**The confounding problem compounds this.** Retrieval is not random — the ranking function in the
prior document's §5.1 query deliberately biases which lessons get shown when (recency, occurrence
count, verified tier), so raw co-occurrence rates are exactly the setting classical observational-
causal-inference methods were built for: Rosenbaum & Rubin, "The Central Role of the Propensity
Score in Observational Studies for Causal Effects," Biometrika 70(1):41-55, 1983. Naive
`helpful_count`/`harmful_count` increments without adjusting for what kind of task/context the
lesson tends to be retrieved into will misattribute task difficulty to lesson quality.

**Mitigation that actually fits the volume.** Do not run a per-lesson frequentist significance test
— it will not fire for the overwhelming majority of a few-hundred-row store within any reasonable
timeframe. Instead pool across the *whole population* of lessons with an empirical-Bayes/shrinkage
estimator (the classical form: Efron & Morris, "Data Analysis Using Stein's Estimator and its
Generalizations," JASA 70(350):311-319, 1975 — Beta-Binomial shrinkage is the direct
descendant applied to helpful/harmful counts). A lesson with only 4 observations gets pulled toward
the population's average win rate rather than reported at its noisy raw ratio; lessons that still
look bad *after* shrinkage, even with few observations, are the ones worth a manual look or a
counterfactual audit (§1.2). This does not make any single lesson's number trustworthy in isolation,
but it produces a serviceable **relative ranking for pruning** using only the tens-to-low-hundreds
of total weekly observations that are actually available at this volume — which is the honest
claim it can support.

**Runs needed to be trustworthy.** Per-lesson: multiple months to multiple years for most of the
store (see above). Store-wide relative ranking via shrinkage: usable within weeks, with the caveat
that confidence per individual lesson stays low until that lesson personally accumulates enough
occurrences — shrinkage produces a better-calibrated ranking, not a shortcut around the underlying
data scarcity.

**Confidence supported.** Relative ranking for pruning/promotion triage. Not absolute
probability-of-help claims for any single entry at low N.

### 1.7 Implicit signals — LRAT, verified, and why it does not transfer as-is

The issue asks specifically about LRAT (arXiv [2604.04949](https://arxiv.org/abs/2604.04949)).
**It exists and the reported claims are accurate**: "Learning to Retrieve from Agent Trajectories,"
Zhou, Dai, Qu, Pang, Xu & Wen, submitted March 2026. It mines retrieval supervision from agent
browsing behaviour — browsed documents become positives, unbrowsed candidates from the same
retrieval batch become reliable negatives (the paper shows agent browsing lacks the position bias
that makes human clicks unreliable negatives), an LLM judge filters out browsed-but-unhelpful
false positives, and reasoning-token length is used to weight a modified InfoNCE contrastive loss.
**The paper does confirm** the failed-run claim from the issue: "retrievers trained with both
correct and incorrect trajectories consistently outperform the base retriever, although incorrect
trajectories yield smaller gains" — it degrades gracefully on failures, it does not need only wins.

**But the mechanism itself does not generalise to this system, for two independent reasons:**

1. **Volume.** LRAT is a gradient-based fine-tune of a dense retriever. Its own experiments used
   **26,482 complete trajectories from 10,000 queries, yielding 91,713 training pairs**, across four
   retriever backbones. At 50–300 sessions/week fleet-wide, reaching 26,482 trajectories takes
   roughly **2–10 years**. This is three to four orders of magnitude past the volume this operator
   will produce, and it is *training data* in the classical ML sense (needed once, up front, to
   produce a usable model), not an online per-item counter — there is no version of "wait longer"
   that fixes this at solo scale; it needs a fleet.
2. **Structural mismatch.** LRAT's signal is a **choice**: the agent is shown a ranked list of
   candidate documents and decides which to open. Pi's recall design (per the prior document's
   §2.1/§5) injects lesson content **directly into context** — there is no analogous "browse vs.
   skip" action to harvest, because the agent never gets to decline to see it. Retrofitting this
   would require redesigning recall as a two-stage "show titles, let the agent request bodies"
   interaction, which trades away the whole point of low-latency, always-injected context recall.

**Verdict:** do not adopt LRAT's training pipeline. Its underlying insight — that what the agent
*does* after seeing a candidate is a better supervision signal than a static judgment made before
seeing it — is exactly what §1.1's turn-level compliance heuristic already captures, for free, as a
non-parametric heuristic rather than a trained model. Reuse the idea; skip the machinery and the
26K-trajectory bill.

## 2. Detecting harm, specifically

The issue is right to weight this asymmetrically: a harmful entry keeps getting retrieved and keeps
poisoning turns until someone notices, while a merely-unhelpful-but-harmless entry just wastes
tokens. Two further findings bear directly on "can harm be detected without a counterfactual":

**Narrow, hand-specified behavioural invariants can work without any counterfactual, but need
large labeled volumes to validate and have serious false-positive ceilings.** "Forensic Trajectory
Signatures for Agent Memory Poisoning Detection" (Leong, arXiv
[2606.30566](https://arxiv.org/html/2606.30566), 2026) shows that a single well-chosen mechanistic
invariant — in their case, a memory-exfiltration attack must call `memory_recall_fact` before
`email_send_email`, because the attacker's payload is a stored *value*, not a key — achieves
AUC=0.9563 and Recall=0.9792 with **no re-run required**, purely from tool-call-log structure. But:
validating and calibrating this needed **N=2,520 labeled runs** (9 models × 7 defenses × 40 runs),
a cross-model holdout of N=280 per model, and at N=20 the paper's own reported Wilson 95% CI on
recall is **[0.839, 1.000]** — their words: "N=20 yields this wide interval; larger samples are
needed for production safety guarantees." Worse, a follow-up N=4,360 study found the same signature
fires on **100% of benign memory-grounded sends** under some conditions (`P(FP | signature)=100%`)
— it captures a valid attack *precondition*, not a maliciousness *predicate*, and needs semantic
gating to be usable. The lesson for this system: a cheap, hand-authored behavioural rule per
lesson-*kind* (e.g. "guidance recommended running command X; agent instead hit a *new* error
signature within 2 turns it hadn't seen before") is affordable and immediate, but do not expect to
*learn* such a rule from your own trace data — you will never accumulate the thousands of labeled
runs the published version needed to calibrate it, and a learned classifier at your volume will
badly overfit. Hand-specify the rule; do not train it.

**Adding statistical sophistication to credit assignment can actively hurt at exactly this scale.**
"AEL: Agent Evolving Learning for Open-Ended Environments" (Xu et al., arXiv
[2604.21725](https://arxiv.org/abs/2604.21725), 2026) is the most directly relevant negative result
found in this search: run on a **208-episode** benchmark — the same order of magnitude as this
system's monthly volume — the authors report that "every additional mechanism we test (planner
evolution, per-tool selection, cold-start initialization, skill extraction, and **three credit
assignment methods**) *degrades* performance," concluding the bottleneck at low episode counts is
"self-diagnosing how to use experience rather than adding architectural complexity." This is direct
empirical support, at solo-relevant N, for preferring the simplest mechanism (§1.1) over anything
that tries to be statistically clever before there is data to be clever with.

**What this means concretely for harm:** the two mechanisms that actually work without a
counterfactual at this volume are (a) §1.1's immediate next-error correlation, tuned toward
over-flagging rather than under-flagging (a false quarantine costs a demotion that can be reversed;
a missed harmful entry compounds), and (b) a small number of hand-written, lesson-kind-specific
behavioural invariants in the spirit of Forensic Trajectory Signatures, calibrated by inspection
rather than learned. Treat the harmful side of the counter as a **sequential test with an
asymmetric stopping rule**, not a batch statistic — this is the classical framing of Wald's
Sequential Probability Ratio Test (A. Wald, *Sequential Analysis*, Wiley, 1947): decide after each
new observation whether the evidence already clears a (deliberately low, asymmetric) bar, rather
than waiting to accumulate a fixed large N before deciding anything. The prior document's
`harmful_count >= 3` quarantine threshold is already, in effect, a crude fixed-count SPRT; the
recommendation here is to keep that shape rather than replace it with a proper significance test
that will essentially never fire at this volume.

## 3. Citation resolution report

Every arXiv ID cited above, plus every one flagged by name in the issue, was fetched and checked
against its claimed content this session:

| ID | Claimed as | Resolves? | Notes |
| --- | --- | --- | --- |
| [2604.04949](https://arxiv.org/abs/2604.04949) | LRAT | **Yes** | "Learning to Retrieve from Agent Trajectories," Zhou et al., March 2026. Claims confirmed (browse/skip signal, works on failed runs) but see §1.7 for why the mechanism itself doesn't transfer — it's a gradient-trained retriever needing ~26K trajectories, not a heuristic. |
| [2604.04503](https://arxiv.org/abs/2604.04503) | MIA | **Yes** | "Memory Intelligence Agent," Qiao et al., April 2026. Manager/Planner/Executor, contrastive success+failure retrieval — content matches the prior document's citation. |
| [2510.04618](https://arxiv.org/abs/2510.04618) | ACE | **Yes** | "Agentic Context Engineering," Zhang et al., Oct 2025 (rev. through Mar 2026). Generator/Reflector/Curator, brevity bias, context collapse, +10.6%/+8.6% figures — content matches. |
| [2605.27366](https://arxiv.org/abs/2605.27366) | MUSE-Autoskill | **Yes** | "MUSE-Autoskill," Lin et al., May 2026. Paper confirmed to exist with matching 85.24%/81.17% self-vs-human figures; the specific N=5/wide-CI claim (repeated from the prior document) was not independently re-confirmed in this session's abstract-level fetch and should be treated as attributed, not independently verified. |
| [2603.15125](https://arxiv.org/abs/2603.15125) | MCFA memory poisoning | **Yes** | "From Storage to Steering: Memory Control Flow Attacks on LLM Agents," Xu et al., March 2026. ">90% of trials vulnerable" confirmed; the specific "100% relapse rate when fixed conversationally" detail from the prior document was not independently re-confirmed at abstract level in this session. |
| [2409.00729](https://arxiv.org/abs/2409.00729) | ContextCite | **Yes** | Cohen-Wang et al., NeurIPS 2024. Context attribution via ablation surrogate; poisoning-detection application confirmed at abstract level. |
| [2306.05685](https://arxiv.org/abs/2306.05685) | LLM-as-judge (MT-Bench) | **Yes** | Zheng et al. 2023. ~85% GPT-4/human agreement confirmed. |
| [2305.04388](https://arxiv.org/abs/2305.04388) | CoT unfaithfulness | **Yes** | Turpin et al. 2023. Up to 36-point accuracy drop from unfaithful explanations confirmed. |
| [2604.12007](https://arxiv.org/abs/2604.12007) | Memory Worth / "When to Forget" | **Yes** | Simsek, April 2026. 3,000–10,000-episode convergence figures confirmed directly from the paper's own PDF. |
| [2605.04811](https://arxiv.org/abs/2605.04811) | TreeMem | **Yes** | Mao et al., May 2026. Monte-Carlo tree-based multi-agent credit assignment confirmed to exist; full methodology (reward signal, rollout count) was not extractable from the fetched PDF in this session and is not relied on for any quantitative claim above. |
| [2605.08374](https://arxiv.org/abs/2605.08374) | MemQ | **Yes** | Liao et al., May 2026. TD(λ) credit propagation over a provenance DAG confirmed; reward-signal and data-volume specifics were not extractable and are not relied on above. |
| [2604.16706](https://arxiv.org/abs/2604.16706) | AgentProp-Bench | **Yes** | Gurram, April 2026. κ=0.049 (substring judge) and κ=0.432 (3-judge ensemble) confirmed directly. |
| [2604.21725](https://arxiv.org/abs/2604.21725) | AEL | **Yes** | Xu et al., April 2026. The "every additional mechanism...degrades performance" finding, including "three credit assignment methods," confirmed directly. |
| [2606.30566](https://arxiv.org/html/2606.30566) | Forensic Trajectory Signatures | **Yes** | Leong, June–July 2026. AUC/recall/CI figures confirmed directly, including the N=20 wide-CI admission and the 100%-benign-false-positive limitation. |
| [2605.01782](https://arxiv.org/abs/2605.01782) | Needle-in-RAG / RAGCharacter | **Yes** | Cui & Liu, May 2026. Exists, "budgeted counterfactual masking and replay" confirmed at abstract level; exact replay-count/cost figures were not extractable from the fetched content and are not relied on above beyond the qualitative mechanism description. |
| 1707.02038 | Thompson Sampling tutorial | **Yes** | Russo et al., 2017/2018, Foundations and Trends in ML — pre-2026, standard reference, used only for the bandit-pooling framing in §1.6/ranked list, not for a specific number. |
| Rosenbaum & Rubin 1983, Biometrika 70(1):41-55 | Propensity score | **Yes** | Confirmed via search; foundational, pre-dates any recency concern. |
| Efron & Morris 1975, JASA 70(350):311-319 | James-Stein / empirical Bayes | **Yes** | Confirmed via search; foundational, pre-dates any recency concern. |

**No citation relied upon above failed to resolve.** Every arXiv ID named in the issue
(2604.04949, 2604.04503, and the ACE/2510.04618 reference from the prior document) exists and says
what it was claimed to say, with the one important qualification on LRAT's actual mechanism spelled
out in §1.7. A handful of tangentially-related papers surfaced during search (TreeMem, MemQ) exist
but their full methodology was not extractable through the available fetch tooling in this session
— they are listed for completeness and are explicitly **not** used to support any quantitative claim
in this document.

## 4. Ranked shortlist — mechanisms viable at solo volume

Ranked by (usefulness × affordability) at 50–300 sessions/week fleet-wide. "Trustworthy" below means
"produces a result worth acting on at the stated confidence level," not "reaches p<0.05."

| Rank | Mechanism | Cost per signal | Runs needed before trustworthy | Confidence it supports | Pi implementation |
| --- | --- | --- | --- | --- | --- |
| **1** | Turn-level behavioural compliance + immediate next-outcome correlation (§1.1) | ~$0, no LLM call, pure trace parsing | N=1 per signal; use a fixed small-count threshold (e.g. 2–3 consecutive harmful signals), not a significance test | Binary keep/quarantine decision. Not a magnitude claim. | Extend `evolve-capture.ts`'s existing `tool_result` handler: on `before_agent_start`, record injected lesson IDs (already `retrieval_log.returned_ids`); on the next `tool_call`, string/structural-match `event.input` against each injected lesson's `blocked_command`/`guidance`; on the paired `tool_result`, check `isError`/`exitCode` and whether a *new* `error_sig` appeared. Emit `lesson_feedback` rows with `signal='used'`/`helpful'`/`'harmful'` directly — no new hook needed, only new logic inside hooks the prior document already specified. |
| **2** | LLM-as-judge on the trajectory, folded into the existing nightly Reflect pass (§1.3) | ~free marginal cost (Reflect already runs an LLM pass per session) | N=1 per session; reliability capped at κ≈0.43 for agent-trajectory judgments (not the ~85% figure from chat evaluation) | Moderate corroborating vote; combine with rank 1, escalate on disagreement | Extend the existing Reflector prompt (Appendix A of the prior document, run via `pi -p --mode json`) to require a per-injected-lesson `{helpful\|harmful\|neutral\|not_used}` verdict with justification, alongside its existing lesson-extraction job. |
| **3** | Self-report at `agent_settled` (§1.5) | ~$0, folded into existing generation | N=1, immediate | Low standalone (CoT-unfaithfulness literature); usable only as a prefilter | One additional required field in the closing-turn schema; use only to decide whether ranks 2/4 are worth invoking (skip the judge call when self-report and rank-1 already agree). |
| **4** | Run manifests + population-level empirical-Bayes shrinkage, *not* per-lesson significance testing (§1.6) | Cheap engineering; the `retrieval_log`/`lesson_feedback` tables already planned are sufficient | Per-lesson: months to years for most of the store (see §1.6's 3,000–10,000-episode anchor). Store-wide shrinkage ranking: usable within weeks on tens-to-low-hundreds of total observations | Relative pruning-candidate ranking across the store. **Not** an absolute per-lesson probability claim | Nightly curator job: fit a Beta-Binomial (or similar) hierarchical model over all lessons' `helpful_count`/`harmful_count`, output a shrunk win-rate per lesson, flag the bottom percentile for manual review or for the audit budget in rank 5. |
| **5** | Full task-level counterfactual A/B, budgeted as a monthly audit of 3–5 hand-picked lessons only (§1.2) | High — 36 to 1,479+ runs per arm depending on the effect size you need to detect (table in §1.2) | Only ever affordable for gross (≥20–30pp) effects, on a handful of lessons per month, never as a per-entry default | The only mechanism that supports an actual causal magnitude claim — spend it deliberately | Reuse the prior document's existing eval harness (`--no-extensions`/current-store/candidate-store conditions, §9.1) against archived failing sessions replayed via `--session-dir`/`--append-system-prompt`; reserve for the top of the rank-4 flag list. |
| **6** | Local/targeted counterfactual ablation, ContextCite/RAGCharacter-style (§1.4) | Cheap in raw compute (a few extra calls at one decision point) but needs harness plumbing Pi doesn't currently expose, and is unvalidated on tool-using coding trajectories | N=1 per contested decision — no repeated task runs | Tells you the lesson was causally load-bearing for one step, not whether that step's outcome was good | Not v0. Worth prototyping once the harness can force controlled regeneration at a single decision point; trigger only on rank-1/rank-2 disagreement or a flagged bad outcome. |
| **7** | LRAT-style learned retriever fine-tuning from browse/skip behaviour (§1.7) | Requires building a contrastive-training pipeline | ~26,482 trajectories / ~91,713 pairs in the published result — 2–10 years at this system's volume | Not viable at all at this scale | Do not build. Reuse the underlying idea via rank 1 instead. |

**Bottom line.** Nothing in this survey lets a solo operator run a classical statistical test
per lesson and get a trustworthy answer within a useful timeframe — every mechanism that was
actually measured in the literature (Memory Worth's 3,000–10,000 episodes, the two-proportion power
table, LRAT's 26K trajectories, the Forensic Trajectory Signatures' 2,520–4,360-run validation sets)
needs volumes one to four orders of magnitude past 50–300 sessions/week. The honest v0 answer is:
**use the deterministic trace heuristic (rank 1) as the mechanism that actually sets
`helpful_count`/`harmful_count`, treat it as a sequential/asymmetric decision rule rather than a
statistic, corroborate it cheaply with a judge call already being paid for (rank 2) and a self-report
prefilter (rank 3), pool across the whole store with shrinkage for *ranking* rather than per-item
truth (rank 4), and spend a small, deliberate counterfactual budget only on the few entries that
keep surfacing as suspect (rank 5).** This is also consistent with the one directly on-point
negative result found in this search: AEL (arXiv 2604.21725) found that adding statistical
sophistication to credit assignment *degraded* performance at a 208-episode scale — almost exactly
this system's expected monthly volume.
