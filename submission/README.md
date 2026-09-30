# Evaluation & Observability Capstone — Evidence Pack

This is a fresh evidence pack generated from the supplied course repository. The separate student-reference ZIP was not copied into the submission or used as prose/artifact content.

## Structure

- `environment.txt` — execution environment and limitations
- `reflection-brief.md` — completed reflection grounded in this pack
- `perturbation-log.md` — one controlled experiment per system
- `01-policy-pipeline/` — policy tests, routing artifact, calibration, perturbation, screenshot
- `02-mortgage-extraction/` — mortgage tests, three replay runs, discrepancy perturbation, screenshot
- `03-supply-chain/` — supply-chain tests, briefing, timeout run, screenshot

## Important reproducibility note

The execution sandbox had no outbound package-index access and no `ANTHROPIC_API_KEY`. Therefore:

- Offline/replay paths were run and captured.
- The policy live pipeline command was attempted only as a documented limitation; its offline routing-test fallback was captured instead.
- `mypy` and `ruff` were unavailable. Their captures say so and include `compileall` as a supplemental syntax check; they are not represented as successful mypy/ruff runs.
- The supply-chain run used a temporary compatibility layer outside this submission folder because Chroma/sentence-transformers were not installed. No project source files were modified.

For a final local submission against the exact rubric, replace the policy fallback with the live pipeline run after setting `ANTHROPIC_API_KEY`, and rerun `mypy`/`ruff` after installing the dev dependencies normally.
