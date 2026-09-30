# Reflection Brief — Evaluation and Observability Capstone

**Name:** Piyush Raspalle
**Date:** 2026-09-30

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Microsoft Windows 11 Home Single Language, 64-bit, 10.0.26200 |
| Python version | 3.13.15 |
| Date run | 2026-09-30 |
| Ran any system live? (which) | Insurance policy pipeline attempted live; the Anthropic request returned HTTP 401, so no successful live routing run was produced. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | No successful `routing.json` was generated because the live API request returned HTTP 401 |
| auto_approve / human_review / spot_check counts | Not available from a successful live run |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> The perturbation test produced a `RetryFutileEscalation` for the missing `endorsements` field, with `detected_pattern="endorsements_absent"`. The system made exactly **1 API call** (`client.call_count == 1`) and stopped. The source field was absent, so another identical model call would not provide new source information. Retrying a futile case wastes an API call without changing the available evidence; escalation makes the missing-source problem explicit for human handling.

**1b. Reading the router.** Pick one `human_review` record from your routing test case. Which of the three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's confidence alone, what would have happened?

> The high-confidence reviewer-disagreement test case had `0.99` confidence for every field, but the independent reviewer disagreed on `premium_amount`. The routing result was `human_review`, with `reviewer_disagreements = ["premium_amount"]` and `fields_below_threshold = []`. If confidence alone had been trusted, this record would not have been escalated even though the independent reviewer identified a conflict.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a single number hides?

> The `umbrella × exclusions` cell had `n=2`, confidence **0.93**, but accuracy **0.00**. The overall result was **Brier=0.291**. The slice exposes that confidence was badly miscalibrated for this particular policy type and field, which a single aggregate figure can hide.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `fixtures\documents\income_sum_mismatch.txt` |
| Classified type | Income verification / earnings statement |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each
cannot catch.

> The discrepancy test reports `field="total_monthly_income"`, `calculated=4500.0`, `stated=5000.0`, and `delta=-500.0`; the validation report has `consistent=False`. Tool use guarantees that the model returns structured JSON in the expected format, but it does not guarantee that the values inside that JSON are mathematically consistent. Valid JSON could contain a stated total of 5000.0 when the extracted components sum to 4500.0; the deterministic validator catches that inconsistency. Conversely, a mathematically consistent value can still be the wrong value if the model extracted the source incorrectly.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null instead of an invented value? Point to the schema choice that allows it.

> The missing-bonus test returns `bonus_monthly = None` and `bonus_ytd = None`. The extraction prompt states: **“Return null for any field not explicitly stated in the document. Do not infer, default, or fabricate.”** Null represents an unknown or unstated value; zero would incorrectly assert that the document established there was no bonus. The model schema allows the field to be nullable.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> The appraisal fixture contains **“about 2,400 sq ft”** and the extraction test requires `gross_living_area_sqft == 2400` as an integer. Normalizing during extraction gives downstream validation and consumers a consistent numeric representation and avoids requiring every later consumer to parse source-specific formatting.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed, 2 warnings |
| Briefing file | `briefing.md`; timeout evidence: `briefing-timeout.md` |
| Section the conflict landed in | `Well-Established` → `defect_rate_ppm` |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> `defect_rate_ppm` reports **180.0 ppm — supplier_audit (as of 2026-04-10)** and **190.0 ppm — internal_quality (as of 2026-04-08)**. Preserving both values, sources, and dates lets the reader see the underlying measurements rather than having the system silently replace one source with another.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the
run still finish?

> The timeout run begins with **“Sources unavailable: logistics unavailable (timeout)”**. It then places `late_shipment_count` under **Incomplete** with **“missing source: timeout reading logistics”**. This differs from `production_capacity_utilization`, which is also incomplete because **“no source reported this metric.”** The coordinator continues using the supplier audit, internal quality, and industry-news sources, so one unavailable source does not prevent the complete briefing from being produced.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> The industry-news evidence says Meridian Components issued a precautionary recall on **2026-02-19**, while another industry-news item reports a Port of Long Beach disruption affecting Meridian Components on **2026-03-17**. Requiring dates makes the timing of each observation explicit, so events occurring at different times are not automatically interpreted as contradictory measurements of the same event.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the output, don't trust the model's word* most clearly caught something a trusting design would have shipped.

> The clearest example was the **Mortgage Document Extraction System** in `tests/test_us04_validator.py`. The extraction can produce valid structured data, but the deterministic validator calculated `4500.0` while the stated total was `5000.0`, producing a `delta=-500.0` and `consistent=False`. A design that trusted the structured extraction without independently checking the arithmetic could have shipped the inconsistent total.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> This mattered most in the **Insurance Policy Extraction Pipeline**. The calibration run showed `umbrella × exclusions` with **confidence 0.93 but accuracy 0.00**. The routing tests also included a case where every extracted field had **0.99 confidence**, yet an independent reviewer disagreed on `premium_amount` and the record was routed to `human_review`. These observations show that a high confidence value does not by itself establish correctness.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> For a workflow such as extracting financial or operational information from invoices, statements, or supplier reports, I would start with **independent review with deterministic routing**. I would instrument schema-validation failures, field-level confidence, reviewer disagreements, routing decisions, disagreement rates by field and document type, and downstream validation failures. These signals would show when the extraction was producing plausible-looking but unreliable structured results and when the routing thresholds or review process needed attention.


