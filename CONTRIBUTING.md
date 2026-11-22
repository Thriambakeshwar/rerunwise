# Contributing

Thanks for helping. This project values correctness and honest claims over feature count.

## Setup

```bash
python -m venv .venv && . .venv/bin/activate
python -m pip install -e .
python -m unittest discover -s tests -t .
```

Python >= 3.11. The runtime has no third-party dependencies, so please do not add any. Dev-only tools (ruff, mypy, coverage) are fine in CI.

## Ground rules

- Every behaviour change needs a test. Statistical changes need a test against a closed form or a simulation with a fixed seed.
- Do not report benchmark numbers you did not produce with `benchmarks/run.sh`. If you change the algorithms, re-run it and update `benchmarks/results-*.txt` and the README table together.
- Parser support for a new runner must come with a **real captured** JUnit file (scrubbed of secrets) in `tests/fixtures/`, not a hand-written imitation. Say in the PR which tool and version produced it.
- Keep `docs/testing-status.md` truthful: PASS only for checks you actually ran.

## Pull requests

Small, focused PRs. Describe the problem, the approach, and what you ran. By contributing you agree your work is licensed under the MIT license.
