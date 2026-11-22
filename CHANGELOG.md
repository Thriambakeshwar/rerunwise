# Changelog

## 0.1.0 (unreleased)

First release. Everything below is covered by the unit/integration suite; see
`docs/testing-status.md` for what has and has not been validated.

- JUnit XML ingestion (plain, Maven Surefire `flakyFailure`/`rerunFailure`, duplicate-testcase retries) with XXE/DOCTYPE rejection
- Multiple-changepoint regime segmentation (Beta-Bernoulli optimal partitioning)
- Classification: broken / flaky (confirmed or statistical) / suspect / recovered / healthy / insufficient
- Retry planner with explicit false-red bound and model-based policy comparison
- CLI: `analyze`, `plan`, `explain`, `quarantine`, `simulate`, `benchmark`
- Simulation framework with ground truth and a reproducible benchmark
