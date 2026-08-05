# LLM Evaluation Frameworks

Reference material for evaluating prompts, RAG pipelines, and agent outputs. This complements `scripts/rag_evaluator.py` (which implements a subset of the metrics below) — use this file when the user needs the reasoning behind a metric, wants to reach for a named open-source framework, or is designing an eval set from scratch.

## 1. Golden Datasets: How to Build One That Doesn't Lie to You

An eval set is only as good as its coverage and its labels. Most "our eval says it's fine" failures trace back to the dataset, not the metric.

**Minimum viable golden set (per Operating Rule 2 in SKILL.md): 10–20 cases with expected outputs.** Below that, a single quirky case can swing pass-rate by 5–10 points and you're optimizing noise.

### Sourcing cases
- **Production logs, stratified.** Pull real inputs, not invented ones — invented cases are biased toward what the prompt author already thought of. Stratify by known segments (user type, input length, language, difficulty) so rare-but-important cases aren't averaged away.
- **Failure-driven additions.** Every bug a user reports or you find in code review becomes a permanent eval case. This is the single highest-leverage habit in eval maintenance — it's how a golden set becomes a regression suite instead of a static snapshot.
- **Adversarial/edge cases deliberately authored.** Empty input, max-length input, conflicting instructions, prompt-injection attempts embedded in retrieved content, ambiguous phrasing. These rarely occur in organic logs but cause the worst failures.
- **Near-duplicates for consistency checks.** Two phrasings of the same underlying request should get the same answer. If they don't, you have a stability problem, not a correctness problem — a different fix.

### Labeling
- Prefer **reference-based** labels (an expected output, or a decision + key facts) over vague rubrics — "the response should state X, Y, and NOT claim Z" is checkable; "should be helpful" is not.
- For open-ended generation, label with **decomposed criteria** (a checklist) rather than one holistic score. A single 1–5 "quality" score conflates correctness, tone, and format, and gives no diagnostic signal when it drops.
- **Version your golden set.** Treat it like code — diffs, review, changelog. A silently-edited eval set invalidates every historical comparison against it.
- **Held-out slice.** Keep 20–30% of cases never used during prompt iteration, only checked before ship. Otherwise you overfit the prompt to the visible eval cases (this is explicitly called out in SKILL.md's few-shot workflow: "if a case passes only because it resembles an example, add a held-out variant").

### Sizing and refresh
- 10–20 cases catches gross regressions; 50+ is needed before pass-rate deltas of a few points are meaningful (see Section 4). Scale the set to the decision you're making — ship/no-ship on a small prompt tweak needs less than a quarterly model-migration decision.
- Refresh when: the underlying task distribution shifts, a new failure mode is found, or the eval set starts scoring 95%+ across the board (ceiling effect — it's no longer discriminating between good and bad prompts).

## 2. LLM-as-Judge: Pitfalls and Mitigations

Using a model to grade another model's output is the only scalable option for open-ended tasks, but it has well-documented, reproducible biases. Treat judge scores as a proxy metric, never ground truth.

### Documented failure modes

| Bias | What happens | Mitigation |
|---|---|---|
| **Position bias** | In pairwise comparisons, the judge favors whichever response is shown first (sometimes by 10-20+ points of win rate) regardless of quality | Score every pair twice with order swapped; discard or flag disagreements as ties rather than averaging blindly |
| **Verbosity bias** | Longer, more elaborated answers score higher independent of correctness | Add an explicit rubric line: "length and elaborateness are not virtues; penalize padding and unsupported claims" |
| **Self-preference bias** | A judge model rates outputs from its own model family more favorably | Use a different model family as judge than the one being evaluated, or triangulate with a second judge model |
| **Anchoring/scale compression** | Numeric scores cluster in a narrow band (e.g., everything is "7 or 8 out of 10") | Prefer pairwise or binary decisions over 1-10 scales; if a scale is required, force distribution with per-criterion binary checks summed rather than one holistic number |
| **Sycophancy toward the prompt's framing** | If the grading prompt hints at the "expected" answer, the judge rationalizes agreement | Grade blind to which system produced which output; don't include the prompt author's own justification in the judge context |
| **Format/style sensitivity** | Markdown formatting, confident tone, or citations move scores even when content is equivalent | Normalize formatting before grading, or explicitly instruct the judge to ignore style and focus on the checklist |
| **Low reliability on subtle correctness** | Judges are good at catching gross issues (off-topic, malformed, unsafe) and much weaker at catching subtle factual errors or reasoning gaps — this is the actual failure mode that matters most | Use judges as a first-pass filter, not final say, for anything safety- or correctness-critical; route ambiguous or high-stakes cases to a human reviewer |

### Concrete mitigations to build into the eval harness
1. **Decompose into a rubric with discrete, checkable criteria** ("cites at least one source," "does not contradict the provided context," "answers the literal question asked") and have the judge output pass/fail per criterion plus a one-line justification per criterion. This is more reliable than one holistic score and gives you a diagnosable report, matching the "decomposed criteria" labeling approach in Section 1.
2. **Calibrate the judge against human labels before trusting it.** Run the judge on 30-50 cases that also have human labels; report agreement (see Section 3, treat this like inter-rater reliability). If agreement is below ~80% on a binary criterion, the rubric is underspecified — tighten it, don't just accept the judge.
3. **Use structured output for the judge's response** (the API's native JSON schema / tool-call enforcement — per SKILL.md Operating Rule 3, prefer platform features over prompt hacks) so scores are always parseable and you never lose data to a malformed judge response.
4. **Chain-of-thought before the verdict, verdict last.** Ask the judge to reason first, output the score/verdict as the final token(s). Verdict-first framing anchors the reasoning to justify a score already committed to.
5. **Re-run judge scoring with temperature 0 (or the lowest available) and check for run-to-run flips** on a sample — judge noise is itself a source of false eval deltas.

## 3. Pairwise vs. Pointwise Scoring

**Pointwise (absolute) scoring**: judge or rubric assigns a score to one output in isolation ("rate this response 1-5" or "pass/fail against this checklist").
- Pros: scales linearly (N outputs = N judge calls), scores are comparable across different eval runs and time, works for regression tracking (SKILL.md's eval-gate workflow assumes pointwise pass/fail per case).
- Cons: absolute scales are where anchoring/scale-compression bias (Section 2) is worst; a model's "7/10" today may not mean the same thing as its "7/10" next week if grading drifts.
- Best for: regression gates, per-case pass/fail tracked over time, anything used as an automated CI gate — which is exactly how it's used in the prompt-optimization eval gate in SKILL.md.

**Pairwise (relative) scoring**: judge is shown two (or more) outputs for the same input and picks the better one, or ranks them.
- Pros: humans and LLM judges are both measurably more consistent at relative comparison than absolute scoring — "which is better" is an easier, more reliable judgment than "how good is this on a scale of 10." Directly produces a preference signal usable for prompt A/B decisions.
- Cons: O(N²) comparisons if you want full ranking of N variants (mitigate with Elo/Bradley-Terry style incremental ranking instead of all pairs); position bias (Section 2) must be controlled by swapping order; doesn't give an absolute quality bar — "better than variant B" says nothing about whether either is good enough to ship.
- Best for: A/B prompt comparisons (revised vs. baseline — exactly the `--compare baseline.json` pattern in the prompt optimizer), model migration decisions, picking a winner among several candidate prompts.

**Practical rule**: use pairwise when the question is "which of these two prompts is better," use pointwise when the question is "is this good enough to ship" or "did this regress." Many mature eval setups run both — pointwise checklist scoring for the CI gate, pairwise preference sampling for qualitative prompt-iteration decisions that pointwise scores don't resolve.

## 4. Statistical Significance for Eval Deltas

The most common eval mistake: treating a 2-point pass-rate change on a 15-case eval set as a real signal. It usually isn't.

### Why this matters
With n=15 binary outcomes, the standard error on a pass rate near 80% is roughly sqrt(0.8×0.2/15) ≈ 0.10 — a ±10-point swing is within noise. A prompt change that moves pass rate from 80% to 87% on 15 cases is not distinguishable from chance.

### What to do about it
1. **Report confidence intervals, not point estimates.** For a binary pass/fail metric, a Wilson score interval (more reliable than the normal approximation at small n) around the observed pass rate tells you the plausible range. If baseline's interval and revision's interval overlap substantially, don't ship on the strength of that number alone — go get more cases or a paired test.
2. **Use a paired test, not two independent proportions.** Since both prompt variants are run on the *same* eval cases, use McNemar's test (for binary pass/fail) or a paired t-test / Wilcoxon signed-rank (for continuous scores like judge ratings) — pairing removes the "some cases are just harder" noise that an unpaired comparison wastes power on.
3. **Bootstrap resampling** when the metric is a composite (e.g., an average of several rubric criteria): resample eval cases with replacement 1,000+ times, recompute the delta each time, and report the 90% or 95% interval of the resulting distribution. If the interval crosses zero, the delta is not distinguishable from noise.
4. **Pre-register the regression threshold before running the comparison**, not after seeing the number — this is precisely what SKILL.md's eval gate does mechanically (`clarity_score >= baseline` and `token_count <= baseline * 1.10`, checked programmatically, exit 0/1). A hard-coded, pre-committed gate removes the temptation to rationalize a marginal drop as "probably fine."
5. **No-regression is a per-case rule, not an aggregate one.** An aggregate score can stay flat while individual previously-passing cases silently start failing (canceled out by other cases newly passing). Always diff the *set* of passing case IDs between baseline and revision, not just the count — this is the concrete meaning of "the revision must not lose any previously-passing eval case" in SKILL.md's prompt-optimization workflow.
6. **Minimum n for a trustworthy delta**: as a rule of thumb, don't claim a pass-rate difference smaller than ~10 points is real below n≈30, and don't claim a difference smaller than ~5 points is real below n≈100. Below that, only report and act on binary regressions (a case that passed now fails) — those are meaningful at any n, since they're a paired, per-case signal rather than an aggregate one.

## 5. Named Frameworks

### promptfoo
An open-source CLI/config-driven eval runner for prompts. Config is a YAML file listing prompts (or prompt templates with variables), providers (models to run them against), and test cases with assertions.
- **Assertion types worth knowing**: `equals`/`contains` (exact/substring), `icontains`, regex, `javascript`/`python` (custom scoring function), `llm-rubric` (LLM-as-judge against a natural-language rubric), `similar` (embedding cosine similarity to a reference), `cost`/`latency` thresholds, `is-json`/`is-valid-openai-function-call` for structured-output validation.
- **Matrix evaluation**: runs every prompt × every provider × every test case, producing a pass-rate grid — this is the natural tool for "which of these 3 prompt variants works best across these 2 model providers" questions, i.e., the A/B comparisons from Section 3.
- **Where it fits alongside this skill's tools**: `prompt_optimizer.py` does static analysis (structure, tokens, clarity) on a single prompt file offline; promptfoo actually executes prompts against live model APIs and grades the outputs. Use the optimizer for the "is this prompt well-formed" pass, then promptfoo (or an equivalent harness) for the "does it actually produce correct outputs" pass — the eval gate step in SKILL.md's workflow is exactly where a promptfoo run belongs, feeding into `eval_results.json`.
- **Red team mode**: promptfoo also generates adversarial test cases (prompt injection, jailbreak attempts, PII leakage probes) automatically — useful for the "adversarial cases deliberately authored" bullet in Section 1 when you don't want to hand-write every attack.

### DeepEval
A Python-native eval framework (pytest-style) with a large library of pre-built metrics, several backed by academic research rather than ad hoc heuristics.
- **Notable metrics**: `AnswerRelevancyMetric`, `FaithfulnessMetric`, `ContextualPrecisionMetric`, `ContextualRecallMetric` (the RAG metrics — same family as ragas below, and as `rag_evaluator.py`'s relevance/precision@k/faithfulness), `HallucinationMetric`, `ToxicityMetric`, `BiasMetric`, `SummarizationMetric`, `GEval` (a general-purpose configurable LLM-judge metric that takes a natural-language criteria description and produces a 0-1 score with chain-of-thought — DeepEval's answer to the "decompose into rubric criteria" guidance in Section 2).
- **Integrates as pytest assertions**: `assert_test(test_case, [metric1, metric2])` — this makes eval-gating a literal CI step (`pytest test_eval.py`), which is the most direct way to enforce the "must pass before shipping" rule from SKILL.md's eval gate.
- **Synthetic data generation (`Synthesizer`)**: generates golden-set candidate cases from source documents — useful for bootstrapping Section 1's golden dataset from an existing knowledge base, though synthetic cases should be reviewed/edited by a human before being trusted as ground truth (they inherit whatever gaps the generating model has).
- **Confident AI (DeepEval's hosted companion)**: gives dashboarding, historical eval-run comparison, and regression detection across runs — useful once eval gating in CI is producing enough history to want a trend view instead of a single JSON diff.

### ragas
The most widely used framework specifically for RAG pipeline evaluation — its metric set is the reference definition most other RAG evaluators (including `rag_evaluator.py`'s report) are implicitly benchmarked against.
- **Core metrics**:
  - `Faithfulness` — what fraction of claims in the generated answer are supported by the retrieved context, computed by extracting atomic claims from the answer via an LLM and verifying each against context. This is the direct analog of the "faithfulness" and "groundedness" figures in `rag_evaluator.py`'s report.
  - `Answer Relevancy` — generates hypothetical questions from the answer and measures embedding similarity back to the original question, penalizing answers that are grounded but non-responsive (correct facts, wrong question).
  - `Context Precision` — of the retrieved chunks, what fraction are actually relevant to answering the question (this is precision@k from `rag_evaluator.py`).
  - `Context Recall` — of the facts needed to answer (usually derived from a reference answer), what fraction appear somewhere in the retrieved context — this is the "coverage" metric in `rag_evaluator.py`'s report.
  - `Context Entities Recall`, `Noise Sensitivity` — finer-grained variants for entity-heavy domains and for measuring how much irrelevant-but-plausible context degrades answer quality.
- **Distinguishing design choice**: most ragas metrics are themselves LLM-judged (claim extraction, verification, question generation) rather than purely embedding-similarity-based — which means ragas scores inherit every LLM-as-judge pitfall in Section 2, and should be spot-checked against human judgment on a sample before being trusted as an unattended gate.
- **Where it fits**: use ragas (or an equivalent) as the deeper diagnostic when `rag_evaluator.py`'s cheaper heuristic metrics flag a problem — e.g., relevance < 0.80 per SKILL.md's rule ("fix retrieval before rewriting the generation prompt") — ragas's claim-level faithfulness breakdown tells you *which* claims are unsupported, which a scalar score doesn't.

### Choosing among them
- **promptfoo**: fastest to stand up for prompt-vs-prompt or model-vs-model comparison with a config file, minimal code; best fit for the prompt-optimization eval gate.
- **DeepEval**: best fit when the team already tests in Python/pytest and wants eval assertions living next to unit tests in CI, or wants the broadest single library of pre-built metrics (safety, bias, summarization, RAG) under one API.
- **ragas**: best fit specifically for RAG faithfulness/relevance/coverage diagnostics when `rag_evaluator.py`'s built-in metrics flag a problem and you need claim-level detail to root-cause it.
- These are not mutually exclusive — a mature setup often uses promptfoo for prompt CI gating, ragas for RAG-specific deep dives, and DeepEval (or hand-rolled judges per Section 2) for anything bespoke none of the above covers.
