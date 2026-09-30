# Reflection Brief — Evaluation and Observability Capstone

**Name:** Piyush Raspalle
**Date:** 2026-09-29

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux x86_64, kernel 6.18.44 |
| Python version | 3.13.5 |
| Date run | 2026-09-29 |
| Ran any system live? | No. Anthropic API key was not available; offline/replay paths were used. |

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped — `01-policy-pipeline/tests.txt` |
| Routing output file | `01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 0 / 2 / 1 in the fresh routing-function artifact |

### 1a. Retry boundary

The controlled missing-field run in `01-policy-pipeline/perturbation-detail.txt` produced:

`RetryFutileEscalation`, `field=endorsements`, `detected_pattern=endorsements_absent`, `api_calls=1`.

The field was deliberately made `null`. Once validation identifies the condition as `missing_source`, another model call cannot create evidence that is absent from the source. Retrying would add latency and cost while increasing the chance of inventing a value; escalation preserves the uncertainty instead.

### 1b. Reading the router

In `01-policy-pipeline/routing_decisions.json`, `RUN-UMB-03` is `human_review`. Its `confidence_summary` is high (`0.98` for every listed field), but the record also contains `reviewer_disagreements: ["premium_amount"]` and `integration_failures: ["premium_matches_components_sum"]`. Those independent signals drive the human-review route. If confidence alone had been trusted, this high-confidence record would not have been sent to a human.

A second fresh record, `RUN-HOME-02`, shows the confidence signal alone: `premium_amount` is `0.71` and the decision is `human_review`.

### 1c. Where the aggregate lies

The calibration artifact `01-policy-pipeline/calibration-report.txt` contains:

`umbrella  exclusions n=2 conf=0.93 acc=0.00 brier=0.865`

while the overall result is:

`OVERALL brier=0.291`

The slice exposes a serious calibration problem in one policy-type/field combination that the moderate aggregate Brier score could hide. The useful lesson is to inspect strata rather than treating one overall metric as proof that the system is calibrated.

> **Execution note:** The requested live `policy-extractor pipeline data/policies/` command is recorded as unavailable in `01-policy-pipeline/pipeline-run.txt` because no API key was present. The same routing logic is independently exercised by the fresh routing artifact and the 9 passing routing tests.

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed — `02-mortgage-extraction/tests.txt` |
| Document runs | `extract-run.txt` and `discrepancy-run.txt` |
| Classified/extracted example | Appraisal fixture; `gross_living_area_sqft: 2400` |

### 2a. Two guarantees

`02-mortgage-extraction/discrepancy-run.txt` reports `consistent: false` with:

`calculated: 9642.17`, `stated: 10892.17`, `delta: -1250.0`.

Structured output guarantees that the model response has the required machine-readable shape. It does not prove that the numbers inside that valid JSON obey a business invariant. The arithmetic validator supplies that second guarantee by independently recomputing the total. Conversely, schema validation cannot detect a mathematically valid but incorrect total, while arithmetic validation cannot repair a malformed response that cannot be parsed into the expected structure.

### 2b. Refusing to fabricate

In the `income_missing_bonus.txt` portion of `02-mortgage-extraction/extract-run.txt`, `bonus_monthly` is `null` and `bonus_ytd` is also `null`. The schema intentionally permits nullable fields. This lets the extraction represent “not stated” directly instead of forcing an invented numeric value into a field that the source did not provide.

### 2c. Normalization

The appraisal run in `02-mortgage-extraction/extract-run.txt` returns `gross_living_area_sqft: 2400`. The fixture is specifically the informal-square-footage case described by the project instructions (“about 2,400 sq ft”). Normalizing during extraction gives downstream code one numeric representation rather than requiring every later consumer to parse wording, commas, units, and approximation markers independently.

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed — `03-supply-chain/tests.txt` |
| Briefing file | `03-supply-chain/briefing.md` |
| Section containing the conflict | `Contested` |

### 3a. Annotate, don't arbitrate

The fresh `03-supply-chain/briefing.md` preserves both sides of the on-time-delivery conflict:

- `95.0 percent — supplier_audit (as of 2026-04-10)`
- `78.0 percent — logistics (as of 2026-04-05)`

Keeping both values tells the reader that the disagreement is an evidence issue rather than silently replacing it with a model-selected number. It also preserves the dates and source identities needed to investigate whether the difference is caused by measurement scope, reporting period, or another definition difference.

### 3b. Source goes dark

`03-supply-chain/timeout-run.txt` begins with:

`Sources unavailable: logistics unavailable (timeout)`

and later places `late_shipment_count` under `Incomplete` with `missing source: timeout reading logistics`. That is different from a normal absence of a claim: the system knows the source was expected but could not be read. The run still finishes because the coordinator isolates the failed reader and continues with successful sources instead of aborting the whole investigation.

### 3c. Dates as a guardrail

The same briefing records `average_lead_time_days` as `12.0 days` from `supplier_audit` on `2026-04-10` and `12.0 days` from `logistics` on `2026-04-05`. It also records the contested on-time metric as `95.0 percent` on `2026-04-10` versus `78.0 percent` on `2026-04-05`. Requiring dates prevents a reader from assuming two measurements describe the same observation window merely because their metric names match.

## 4. Synthesis

### 4a. One principle

The clearest example is System 2, in `03-supply-chain/briefing.md`—the supply-chain system does not simply trust a source or model output; it preserves a contested metric instead of silently collapsing it. An even more direct validation example is `02-mortgage-extraction/discrepancy-run.txt`, where valid structured JSON still fails the independent arithmetic check by `$1,250.00`. A trusting design could have shipped the parseable number; the validator stopped it.

### 4b. Confidence ≠ correctness

The policy system makes this explicit. In `01-policy-pipeline/routing_decisions.json`, `RUN-UMB-03` has `0.98` confidence for every field but is still routed to `human_review` because an independent reviewer disagreed and an integration check failed. That artifact demonstrates why self-reported model confidence should be treated as one signal, not as a final correctness certificate.

### 4c. Apply it

For a workflow where an LLM extracts structured results from messy business documents, I would start with **validated retry with escalation** when the dominant risk is an extraction error that can sometimes be corrected from the same source. I would instrument: validation-error category and field, attempt count, escalation rate, null/missing-field rate, final acceptance rate, and sampled human outcomes. For high-impact fields, I would add independent review and deterministic routing, as demonstrated by the policy router. Where multiple external sources are involved, I would preserve source, date, metric, and conflict status so that later users can audit why the system reported more than one value.
