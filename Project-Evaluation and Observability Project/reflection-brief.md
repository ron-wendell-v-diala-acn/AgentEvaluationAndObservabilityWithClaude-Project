# Reflection Brief — Evaluation and Observability Capstone

**Name:** Ron Wendell V. Diala
**Date:** 2026-11-05

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Microsoft Windows 11 Enterprise (10.0.26200) |
| Python version | 3.13.14 |
| Date run | 2026-11-05 |
| Ran any system live? (which) | No. Policy used the offline test fallback; mortgage used replay fixtures; supply chain used `--offline`. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped ([tests](./01-policy-pipeline/tests.txt)) |
| Routing output file | Not generated; no live API run. The focused routing tests are captured in [pipeline-run.txt](./01-policy-pipeline/pipeline-run.txt). |
| auto_approve / human_review / spot_check counts | Not available without the live pipeline run. |
| Perturbation | Raised routing threshold from 0.90 to 0.96, then restored it ([routing-threshold-perturbation.txt](./01-policy-pipeline/routing-threshold-perturbation.txt)). |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> Separately from the threshold experiment, offline test
> `test_ac_01_04_missing_source_halts_immediately` uses `POL-2025-009` with `endorsements=None`.
> It asserts a `RetryFutileEscalation` for `field="endorsements"` and
> `detected_pattern="endorsements_absent"` after exactly **1 API call**
> ([retry test capture](./01-policy-pipeline/retry-perturbation.txt)). A retry cannot recover
> information absent from the source; it would waste a call and risk inventing a value instead of
> routing the case to a person.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> No live routing record was produced because this run used the offline fallback. The focused test
> `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` passed
> ([pipeline-run.txt](./01-policy-pipeline/pipeline-run.txt)); it shows reviewer disagreement is the
> signal that overrides even 0.99 confidence. Trusting confidence alone would auto-approve the
> disputed result. Separately, changing the threshold from **0.90** to **0.96** made otherwise
> auto-approved 0.95-confidence fields route to `human_review`; 4 routing tests failed against
> baseline expectations and 5 passed. Restoring 0.90 returned all 9 tests to passing
> ([routing-threshold-perturbation.txt](./01-policy-pipeline/routing-threshold-perturbation.txt)).

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> `umbrella / exclusions`: confidence **0.93**, accuracy **0.00**, Brier **0.865**; overall Brier
> **0.291** ([calibration-report.txt](./01-policy-pipeline/calibration-report.txt)). Slicing shows
> the confidently wrong subgroup that a moderate overall score can hide.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed ([tests](./02-mortgage-extraction/tests.txt)) |
| Document run | `appraisal_informal_sqft.txt` in replay mode ([extract-run.txt](./02-mortgage-extraction/extract-run.txt)) |
| Classified type | `appraisal`, logged by `--verbose` to stderr ([extract-run.txt](./02-mortgage-extraction/extract-run.txt)) |
| Perturbation | Raised tolerance to USD 1300.00, then restored USD 1.00 ([tolerance-perturbation.txt](./02-mortgage-extraction/tolerance-perturbation.txt)). |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> The mismatch run reports `consistent: false`; `total_monthly_income` is calculated as **9642.17**
> and stated as **10892.17**, delta **-1250.0** ([discrepancy-run.txt](./02-mortgage-extraction/discrepancy-run.txt)).
> Schema validation guarantees shape and field types/nullability, not arithmetic agreement. A schema
> cannot catch a sum mismatch; the arithmetic validator cannot tell whether a structurally valid,
> internally consistent value was misread from the source.
>
> In a separate config perturbation, raising `DEFAULT_TOLERANCE_USD` from **1.00** to **1300.00**
> changed that same **1250.0** discrepancy to `consistent: true` with no reported discrepancies.
> Restoring the default made the replay report `consistent: false` again
> ([tolerance-perturbation.txt](./02-mortgage-extraction/tolerance-perturbation.txt)); this
> demonstrates how an overly permissive tolerance can hide a real mismatch.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> `bonus_monthly` is `null`, not guessed ([missing-field-run.txt](./02-mortgage-extraction/missing-field-run.txt)).
> The extraction schema allows the field to be nullable, so missing source information stays explicit.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> The source says “about 2,400 sq ft”; the output is `gross_living_area_sqft: 2400`
> ([extract-run.txt](./02-mortgage-extraction/extract-run.txt)). Normalizing at extraction gives
> downstream calculations and comparisons a consistent numeric unit.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed ([tests](./03-supply-chain/tests.txt)) |
| Briefing file | [briefing.md](./03-supply-chain/briefing.md) |
| Section the conflict landed in | Contested |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> `on_time_delivery_rate`: **95.0%**, supplier_audit, **2026-04-10**; **78.0%**, logistics,
> **2026-04-05** ([briefing.md](./03-supply-chain/briefing.md)). Preserving both attributed
> measurements lets a reader investigate the source and date differences instead of mistaking an
> arbitrarily reconciled figure for certainty.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> The timeout run says “Sources unavailable: logistics unavailable (timeout)” and lists
> `late_shipment_count` under **Incomplete** as “timeout reading logistics”
> ([timeout-run.txt](./03-supply-chain/timeout-run.txt)). That is distinct from a source that was
> reachable but reported nothing. The coordinator continues with other sources and finishes with
> exit code 0 rather than treating an outage as an empty result.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> The on-time delivery claims are dated **2026-04-10** (supplier_audit) and **2026-04-05**
> (logistics) ([briefing.md](./03-supply-chain/briefing.md)). Keeping those dates beside the values
> makes the different reporting snapshots visible instead of implying a same-time contradiction.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> The clearest catch was the mortgage `income_sum_mismatch.txt` replay: schema-shaped extraction
> output still failed the deterministic total check by **1250.0**
> ([discrepancy-run.txt](./02-mortgage-extraction/discrepancy-run.txt)).

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> Mortgage extraction: the output had valid structure, but its line items did not support the stated
> total. The schema alone would have accepted that shape; the independent consistency check exposed
> the error.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> For mortgage applications, I would start with **validated retry with escalation**: normalize
> values and validate required fields and arithmetic, then escalate genuinely absent information
> rather than retrying or fabricating it. I would track null-required-field and discrepancy rates,
> retry counts, escalation outcomes, and error rates by field and document type.
