# Perturbation Log

These experiments were written from the fresh runs in this evidence pack. The separate student-reference ZIP was not used as wording or artifact content.

## System 1 — validated, routed policy pipeline

- **Change I made:** In a controlled replay response, I changed the required `endorsements` field to `null`, representing a source document where the field cannot be found.
- **Command I ran:** `.venv/bin/pytest tests/test_us01_retry.py -v -k missing_source` and a direct replay check captured in `01-policy-pipeline/perturbation-detail.txt`.
- **What I predicted:** The validator would classify the case as `missing_source`; retrying would not recover information that is absent from the source, so the extractor should escalate after one API call.
- **What actually happened:** `RetryFutileEscalation`, `field=endorsements`, `detected_pattern=endorsements_absent`, `api_calls=1`.
- **How this differs from the unperturbed run:** Normal well-formed replay tests return a `PolicyExtraction`; this perturbed case stops at the missing-source boundary instead of spending additional retries.

## System 2 — schema-enforced two-pass extraction

- **Change I made:** Starting from a consistent structured income extraction, I changed the stated monthly total from `9642.17` to `9742.17` before passing it to the existing deterministic validator.
- **Command I ran:** The validator replay is captured in `02-mortgage-extraction/perturbation-detail.txt`.
- **What I predicted:** The calculated total would remain `9642.17`, the 100-dollar difference would exceed the validator tolerance, and the result would become inconsistent.
- **What actually happened:** `perturbed consistent=False`; discrepancy: `calculated=9642.17`, `stated=9742.17`, `delta=-100.0`.
- **How this differs from the unperturbed run:** The same structured extraction with stated total `9642.17` returned `consistent=True` with no discrepancies.

## System 3 — multi-source synthesis

- **Change I made:** Enabled the existing `--simulate-timeout` configuration for the logistics reader.
- **Command I ran:** `.venv/bin/supply-chain-investigate meridian --offline --simulate-timeout`.
- **What I predicted:** The logistics source would be marked unavailable, its unique metrics would move to `Incomplete`, and the coordinator would still render the remaining evidence.
- **What actually happened:** The briefing begins `Sources unavailable: logistics unavailable (timeout)` and places `late_shipment_count` in `Incomplete`; the run still completed with the other sources rendered.
- **How this differs from the unperturbed run:** Without the timeout, logistics contributes `11.0 shipments` and `78.0 percent` on-time delivery and the on-time metric appears in `Contested` alongside the audit's `95.0 percent`.
