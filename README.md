# rerunwise

![license](https://img.shields.io/badge/license-MIT-blue) ![python](https://img.shields.io/badge/python-%E2%89%A53.11-blue) ![dependencies](https://img.shields.io/badge/runtime%20dependencies-0-brightgreen)

**Tells you which tests are flaky, which are really broken, and what your retry policy is actually buying you. It works from the JUnit XML your CI already produces.**

## The problem

Retrying failed tests hides flakiness but costs CI time, and it hides *real* failures too. The usual retry plugins use one fixed count for everything. Most flaky-test dashboards flag any test that has both passed and failed. That label is wrong for a test that was broken for two days and then fixed, and for a single infrastructure blip.

`rerunwise` separates the cases by looking at the **shape of each test's history**:

| What happened | Naive "has passed and failed" | rerunwise |
|---|---|---|
| Fails ~15% of runs, randomly | flaky | **flaky** (with a rate and 95% interval) |
| Broke on commit X, still failing | flaky | **broken** (+ onset run/commit) |
| Failed for 5 runs, then fixed | flaky | **recovered** |
| One isolated failure | flaky | **suspect** (not enough evidence) |
| Same commit failed and passed | flaky | **flaky, confirmed** |

It then turns the fitted failure rates into a retry plan with an explicit bound, for example "retrying this test up to 2x keeps its false-red probability under 0.1% at 95% credibility". It also says which tests cannot reach that target with a sane number of retries, so you quarantine or fix them.

## Quick start (about five minutes, no network, no dependencies)

```bash
git clone <your-repo-url> rerunwise && cd rerunwise
python -m pip install .            # Python >= 3.11, standard library only
rerunwise simulate --out demo-history --tests 120 --runs 50 --retries 2 --seed 3   # SIMULATED data
rerunwise analyze demo-history --top 10
```

```text
120 tests, 50 runs: 3 broken, 16 flaky, 13 suspect, 5 recovered, 83 healthy

STATUS  RATE (95% CI, current regime)  FAIL/RUNS  TEST
------  -----------------------------  ---------  ---------------------------------------------
BROKEN  99.0% (95.1%-100.0%)           50/50      suite.mod7.TestCase8::test_always_failing_111
BROKEN  97.8% (89.3%-100.0%)           22/50      suite.mod3.TestCase0::test_broken_3
FLAKY   20.6% (10.8%-32.6%)            10/50      suite.mod11.TestCase0::test_flaky_11
FLAKY   34.3% (22.1%-47.7%)            17/50      suite.mod8.TestCase4::test_flaky_60
...
```

(That output is real, produced from simulated data. To use your own data, point `analyze` at a directory of JUnit XML files, one file per CI run.)

```bash
rerunwise plan demo-history                       # retry budgets + policy comparison
rerunwise explain demo-history --test 'suite.mod8.TestCase4::test_flaky_60'
rerunwise quarantine demo-history --unrescuable-only
rerunwise analyze demo-history --fail-on broken   # exit code 1 for CI gating
```

```text
suite.mod8.TestCase4::test_flaky_60
  status:   FLAKY (confirmed by same-code outcomes)
  evidence: 17 same-code contradictory outcome(s); failure rate ~34.3% (95% CI 22.1%-47.7%)
  current regime rate: 34.31% (95% CI 22.06%-47.74%, upper 45.50%)
  regimes:  [0:50) 17/50
  history:  .FFFF.......F.F....FF..F.F......F.F..F.FFF...F....   (F = failed first attempt, | = regime change)
```

## How it works

1. **Parse** JUnit XML. The first attempt of each test is the unbiased Bernoulli trial for that run. Retry evidence is read from Surefire `flakyFailure`/`rerunFailure` elements or from duplicate `<testcase>` entries.
2. **Segment** each test's history into stationary regimes with exact Bayesian optimal partitioning (a Beta-Bernoulli marginal likelihood per regime, with a calibrated penalty per changepoint).
3. **Classify** from the current regime. A Jeffreys-Beta posterior gives the failure rate with a credible interval. Same-commit contradictory outcomes upgrade a flake call to *confirmed*.
4. **Plan** retries. The smallest `r` with `p_upper^(r+1) <= alpha` is chosen, capped by `max_retries`. Tests that cannot reach the target are flagged for quarantine or a fix, and consistently failing tests get no retries.

Details, formulas and assumptions: [`docs/methodology.md`](docs/methodology.md). Architecture: [`docs/architecture.md`](docs/architecture.md).

## Benchmarks: simulated, read the caveats

All numbers come from `rerunwise benchmark` on **simulated** CI history with known ground truth (8 seeds, 200 tests, 60 runs; reproduce with `benchmarks/run.sh`; full output in `benchmarks/results-v0.1.0.txt`). The generator favours a regime-aware method, and the baselines are my own re-implementations of two common heuristics, **not** the actual third-party tools. This is not evidence about your test suite.

| Method | Precision | Recall | F1 |
|---|---|---|---|
| rerunwise | 1.000 | 0.822 | 0.902 |
| baseline: passed-and-failed | 0.429 | 0.950 | 0.591 |
| baseline: flip count >= 2 | 0.493 | 0.950 | 0.649 |

The trade-off is real: rerunwise misses low-rate flakes that have not failed twice yet (the closed-form detection chances are in the benchmark output; for example, a 2% flake needs about 100 runs for a 60% chance of being seen).

Retry policies, scored on fresh draws from the true rates:

| Policy | False-red / 1000 runs | Retries / run |
|---|---|---|
| fixed-3 | 48.9 | 40.25 |
| fixed-3, skip tests classified broken | 48.9 | 4.25 |
| rerunwise adaptive (alpha 0.001) | 93.5 | 4.20 |
| rerunwise adaptive + default 1 retry | 52.2 | 4.25 |

**Honest reading:** nearly all of the retry saving comes from *not retrying consistently failing tests* (40.25 to 4.25 retries per run at identical false-red rate). That needs reliable broken-test detection, which rerunwise provides. The per-test adaptive budgets did **not** beat a simple "fixed-3, skip broken" rule in this simulation. Their value here is the explicit false-red bound per test and the list of tests that retries cannot rescue.

## What is tested, and what is not

Short version (full table in [`docs/testing-status.md`](docs/testing-status.md)): the statistics, parser, history loader, classifier, planner, simulator and CLI have unit and integration tests. **Not yet validated:** parsing of captured real-world CI output from every runner, real-project accuracy, Windows/macOS, and the GitHub Actions workflows (written but not executed). There are no runner adapters yet. The tool only reads JUnit XML.

## Limitations

- Assumes attempts are independent given the current failure rate. Order-dependent or shared-state flakes make retries less effective than the plan predicts.
- Needs history: with fewer than `min_runs` (default 5) executed runs a test is `insufficient`; rare flakes need many runs to be seen.
- Run ordering comes from file names, modification times or a manifest. Wrong ordering gives wrong regimes.
- Retrying can mask real nondeterministic bugs. The plan is decision support, not an auto-fixer.

## Roadmap

See [`docs/roadmap.md`](docs/roadmap.md): validation on real CI corpora, a SQLite history store, a pytest/Jest quarantine plugin, an HTML report, and cost-aware policy search.

## Contributing, security, license

[`CONTRIBUTING.md`](CONTRIBUTING.md) · [`SECURITY.md`](SECURITY.md) · MIT, see [`LICENSE`](LICENSE).
Maintainer: _add your name and contact here before publishing._
