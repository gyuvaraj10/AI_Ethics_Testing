# How Eval Frameworks Actually Calculate These Six Metrics
### Worked through a Loan Processing Agent scenario

**Scenario:** A loan processing agent handles Maria Chen's mortgage application — it retrieves her submitted documents, computes DTI, runs a credit check, renders a decision, and reports back to a loan officer. Each metric below is evaluated against a slightly different snapshot of that interaction, since in production each metric typically runs against the specific output/context pairing it's designed for.

A note before the numbers: the framework class names below (`HallucinationMetric`, `Faithfulness`, `AnswerCorrectness`, `TaskCompletionMetric`, `ResponseRelevancy`) reflect my training knowledge of DeepEval and RAGAS as of mid-2025. Both libraries ship frequent point releases that rename or restructure metrics, so verify exact signatures against docs.confident-ai.com and docs.ragas.io before wiring these into a production harness.

---

## 1. Hallucination Detection — DeepEval `HallucinationMetric`

This is a native, directly-applicable metric. The algorithm doesn't decompose the output into claims — it checks the **entire output** against **each context document independently**, using an NLI-style judgment (entailment / neutral / contradiction). The score is the fraction of context documents the output contradicts.

`Hallucination Score = (# contexts contradicted by output) / (total # contexts)`

**Context retrieved:**

| Doc | Content |
|---|---|
| C1 — pay stub | Gross monthly income: $8,200 |
| C2 — bank statement | Average monthly deposits: $8,500 |
| C3 — credit policy | Minimum income for this loan size: $7,000/month |
| C4 — credit bureau report | Score: 712. One late payment recorded, month 14 |

**Agent output:** *"Maria's monthly income is $8,500, comfortably above the $7,000 minimum required for this loan size. Her credit score of 720 reflects a strong repayment history, with no late payments in the past 24 months."*

| Context | Output checked against it | Verdict |
|---|---|---|
| C1 ($8,200 income) | Output claims $8,500 | **Contradicted** |
| C2 ($8,500 deposits) | Output claims $8,500 | Entailed |
| C3 ($7,000 threshold) | Output claims "comfortably above $7,000" | Entailed |
| C4 (712, one late payment) | Output claims 720, zero late payments | **Contradicted** |

`Score = 2/4 = 0.50`

Against the rubric (PASS < .5, WARN .5–.7, FAIL > .7), 0.50 lands right at the PASS/WARN boundary — it fails PASS because PASS requires strictly less than .5. This is a realistic edge case: the agent conflated bank-deposit income with pay-stub income (a real and common hallucination pattern), plus a transposed credit score.

---

## 2. Unsupported Claim Ratio — RAGAS `Faithfulness` (inverted)

No framework has a metric literally called "Unsupported Claim Ratio," but it's exactly `1 − Faithfulness` from RAGAS. Unlike Hallucination Detection above, Faithfulness decomposes the output into **atomic claims first**, then checks each claim individually against the context.

`Faithfulness = (# claims supported by context) / (total # claims)`
`Unsupported Claim Ratio = 1 − Faithfulness`

**Agent output:** *"Maria's monthly income is $8,200. She has been with her current employer for 3.5 years. Her debt-to-income ratio is 28%. Based on this profile, we recommend a 30-year fixed mortgage at 6.5% APR."*

**Context:** pay stub ($8,200), employment letter ("employed since March 2022"; session date is September 2024, so context implies ~2.5 years tenure), DTI worksheet ($2,296 monthly obligations / $8,200 income = 28.0%). No document in context addresses loan product or rate.

| Claim | Supported by context? |
|---|---|
| Income $8,200/month | Yes — matches pay stub |
| Employer tenure 3.5 years | **No** — context implies ~2.5 years |
| DTI 28% | Yes — matches worksheet |
| Recommend 30-yr fixed @ 6.5% APR | **No** — no source document covers product/rate |

`Faithfulness = 2/4 = 0.50`
`Unsupported Claim Ratio = 1 − 0.50 = 0.50`

Rubric: PASS ≤ .3, WARN .3–.5, FAIL > .5. At exactly 0.50 this sits at the top edge of WARN — one tick over and it fails. Note the distinct failure mode here versus Hallucination Detection: the tenure claim is actively wrong, but the rate/term claim isn't necessarily *false*, it's just **unattributed** — which is the precise thing this metric is designed to catch and Hallucination Detection would miss (since "unsupported" and "contradicted" aren't the same thing).

---

## 3. Factual Accuracy Score — RAGAS `AnswerCorrectness`

This one's a TP/FP/FN classification against a ground-truth record, combined with embedding similarity. RAGAS decomposes both the ground truth and the generated answer into statements, classifies each generated statement, then blends an F1-style factuality score with semantic similarity (default weighting: 0.75 factuality / 0.25 similarity).

`F1 = TP / (TP + 0.5 × (FP + FN))`
`AnswerCorrectness = 0.75 × F1 + 0.25 × semantic_similarity`

**Ground truth (underwriting system of record):** income $8,200/mo, DTI 28%, credit score 712, decision: approve, standard terms.

**Agent output (summary report):** *"Income $8,200/month, DTI ratio 28%. Credit score 705. Recommend approval with standard terms."*

| Statement | Classification |
|---|---|
| Income $8,200 | TP — matches ground truth |
| DTI 28% | TP — matches ground truth |
| Credit score 705 | **FP** (wrong value) and ground truth's 712 is **FN** (not correctly conveyed) |
| Approve, standard terms | TP — matches ground truth |

`TP = 3, FP = 1, FN = 1`
`F1 = 3 / (3 + 0.5×(1+1)) = 3/4 = 0.75`

Semantic similarity between the full ground-truth text and the full output text (embedding cosine) ≈ 0.93 — high, since most of the wording overlaps.

`AnswerCorrectness = 0.75 × 0.75 + 0.25 × 0.93 = 0.5625 + 0.2325 = 0.795 ≈ 0.80`

Rubric: PASS > .9, WARN .75–.9, FAIL < .75. A single wrong digit (705 vs. 712) drops an otherwise near-perfect report from PASS to WARN — illustrating why F1's harsh penalty on numeric mismatches matters more than the semantic-similarity term, which stays high even when a number is wrong because the surrounding sentence structure is nearly identical.

---

## 4. Task Success Rate — DeepEval `TaskCompletionMetric` + aggregate pass rate

This is the one metric that needs two layers: DeepEval's `TaskCompletionMetric` scores a single agent run (0–1, via LLM judge reasoning over the agent's tool-call trajectory against the inferred goal), and the production "Task Success Rate" is a separate aggregate computed by thresholding many of those per-run scores.

**Goal:** "Process Maria Chen's mortgage application end-to-end: verify documents, compute DTI, run credit check, render decision, and notify the applicant within the 24-hour SLA."

**Agent trajectory:** `fetch_documents()` ✓ → `calculate_dti()` ✓ (28%) → `check_credit_score()` ✓ (712) → `make_decision()` ✓ ("Approve") → `send_notification()` — API call timed out, no retry logged, applicant never notified.

DeepEval's judge compares the inferred intended outcome (decision rendered *and* applicant notified within SLA) against what actually happened, and might score this **0.6/1.0** with reasoning along the lines of: "core financial decision logic completed accurately, but the explicit SLA requirement to notify the applicant was not fulfilled."

For the aggregate production metric, set a per-case pass threshold (e.g., ≥ 0.7 counts as "success"). Maria's case, at 0.6, counts as a fail at the aggregate level even though the underwriting math was entirely correct — which is the point: task success has to include the operational requirements, not just the decision logic.

Across 50 applications processed that week, 41 cleared the ≥0.7 threshold; 9 fell short (notification failures, logging gaps, SLA breaches similar to Maria's case).

`Task Success Rate = 41/50 = 0.82`

Rubric: PASS ≥ .85, WARN .7–.85, FAIL < .7 → **WARN**.

Worth flagging given n=50 is a modest sample: a raw proportion like 0.82 deserves a confidence interval, not just a point estimate. Wilson score interval at 95% confidence (z = 1.96):

```
center = (p + z²/2n) / (1 + z²/n) = (0.82 + 0.0384) / 1.0768 ≈ 0.797
margin = [z / (1+z²/n)] × √(p(1−p)/n + z²/4n²) ≈ 1.821 × 0.0578 ≈ 0.105
95% CI ≈ [0.69, 0.90]
```

The true success rate could plausibly be anywhere from solid WARN territory to PASS — at this sample size, the rubric verdict itself carries real uncertainty, which is worth surfacing alongside the point estimate rather than reporting 0.82 as if it were exact.

---

## 5. Grounding Score — no native framework metric; custom embedding-cosine approach

This is the one that doesn't map cleanly onto DeepEval or RAGAS. Both of those libraries (and TruLens's `groundedness_measure_with_cot_reasons`) implement "groundedness" the same way Hallucination Detection and Faithfulness above already do — LLM/NLI claim verification, not raw embedding distance. None of the major frameworks compute grounding as literal cosine similarity between a full response and retrieved context chunks.

The slide's definition ("semantic overlap... embedding cosine similarity" is actually attached to Relevance Score, but Grounding Score's "semantic overlap between response and retrieved context" reads the same way) matches a lighter-weight pattern from custom RAG observability pipelines: embed the response, embed each retrieved chunk, take cosine similarity per chunk, then aggregate (commonly weighted by retrieval rank). It's a cheap, fast complementary signal to run alongside the heavier LLM-judge metrics — not a replacement for them, since it can't catch a fluent-but-wrong paraphrase the way NLI-based faithfulness checking can.

`cosine(A, B) = (A · B) / (||A|| × ||B||)`

Using illustrative low-dimension vectors (real systems would use 1536+ dim embeddings from a model like text-embedding-3 — same formula, same arithmetic, just bigger vectors):

Response embedding `R = [0.8, 0.6, 0.1, 0.3]`
C1 (income policy) `= [0.75, 0.55, 0.05, 0.35]` — closely on-topic
C2 (credit policy) `= [0.2, 0.1, 0.9, 0.05]` — different topic
C3 (general loan terms) `= [0.1, 0.05, 0.1, 0.9]` — barely related

```
cos(R, C1): dot = 1.04, ||R|| = 1.049, ||C1|| = 0.995 → cos = 0.997
cos(R, C2): dot = 0.325, ||C2|| = 0.929 → cos = 0.334
cos(R, C3): dot = 0.39,  ||C3|| = 0.912 → cos = 0.408
```

Weighted by retrieval rank (rank 1 weight 0.5, rank 2 weight 0.3, rank 3 weight 0.2):

`Grounding Score = 0.5(0.997) + 0.3(0.334) + 0.2(0.408) = 0.499 + 0.100 + 0.082 = 0.68`

Rubric: PASS ≥ .8, WARN .65–.8, FAIL < .65 → **WARN**. The response is tightly grounded in the top-ranked income document but only loosely connected to the two lower-ranked chunks, which is what's dragging the weighted average down — a useful diagnostic for retrieval quality independent of whether the claims themselves turn out to be true.

---

## 6. Relevance Score — RAGAS `ResponseRelevancy`

This is the one that literally is embedding cosine similarity, but worth a precision note: DeepEval's similarly-named `AnswerRelevancyMetric` works differently — it extracts statements from the output and has an LLM classify each as relevant/irrelevant (no embeddings involved). The mechanism on this slide — generate reverse-engineered questions from the answer, then compute cosine similarity against the original query — is specifically RAGAS's `ResponseRelevancy` (formerly `AnswerRelevancy`).

**Query:** "What's the status of my loan application and what's the maximum amount I qualify for?"

**Agent response:** "Your loan application has been approved. Based on your income and DTI ratio, you qualify for a maximum loan amount of $385,000 at a 6.5% fixed rate."

An LLM generates N synthetic questions that the response would answer, then each is embedded and compared to the original query embedding:

| Generated question | cosine sim to original query |
|---|---|
| "What is the status of my loan application?" | 0.91 |
| "What is the maximum loan amount I qualify for?" | 0.95 |
| "What's the maximum amount and status of my application?" | 0.97 |

`Relevance Score = (0.91 + 0.95 + 0.97) / 3 = 0.943`

Rubric: PASS ≥ .85 → **PASS**. One robustness detail worth keeping: RAGAS adds a "noncommittal" guard — if the answer is evasive (e.g., "it depends, you should consult a loan officer") the metric forces the score toward zero regardless of cosine similarity, since a vague non-answer can still generate questions that superficially resemble the original query.

---

## Summary

| Metric | Framework / method | Mechanism | Score | Verdict |
|---|---|---|---|---|
| Hallucination Detection | DeepEval `HallucinationMetric` | Per-context NLI contradiction check | 0.50 | WARN |
| Unsupported Claim Ratio | RAGAS `Faithfulness` (inverted) | Claim decomposition + entailment check | 0.50 | WARN |
| Factual Accuracy Score | RAGAS `AnswerCorrectness` | TP/FP/FN F1 + semantic similarity blend | 0.80 | WARN |
| Task Success Rate | DeepEval `TaskCompletionMetric` + aggregate | LLM-judged trajectory vs. goal, thresholded across runs | 0.82 (CI 0.69–0.90) | WARN |
| Grounding Score | No native metric — custom embedding cosine | Chunk-level cosine, rank-weighted | 0.68 | WARN |
| Relevance Score | RAGAS `ResponseRelevancy` | Reverse-question generation + cosine similarity | 0.94 | PASS |

The pattern across this single run — solid on relevance, shaky everywhere else — is a fairly realistic profile for an agent whose retrieval and number-handling need work but whose language generation is well-calibrated to the actual question asked.
