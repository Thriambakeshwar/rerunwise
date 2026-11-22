# Security policy

## Reporting

Please report vulnerabilities privately through GitHub's "Report a vulnerability" (Security tab) once the repository is public. Do not open a public issue. Expect an acknowledgement within a few days; this is a volunteer-run project and no response time is guaranteed.

## Threat model (v0.1.0)

rerunwise reads local JUnit XML files and writes text/JSON to stdout or a path you give it. Input files may come from untrusted pull requests, so:

- XML documents containing `DOCTYPE` or `ENTITY` declarations are rejected before parsing (blocks entity-expansion and external-entity attacks). Covered by tests.
- Files above 64 MiB are rejected.
- The tool never executes test code, shell commands or network requests.
- Output is plain text/JSON/Markdown. Test names are printed verbatim, so if you embed Markdown output in a PR comment, treat names as untrusted text.

Not in scope or not yet analysed: algorithmic-complexity attacks via extremely many tests (changepoint segmentation is O(n^2) in runs per failing test, bounded by `--window`, default 200), and supply-chain risk of your own CI. No third-party code is imported at runtime.
