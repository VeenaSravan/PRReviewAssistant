# PR Review Assistant

PR Review Assistant analyses a GitHub PR diff (`input/pr.diff`) and writes a severity-rated code review to `output/review_report.md`. A sonnet coordinator launches three haiku sub-agents in parallel (`security-agent`, `quality-agent` and `doc-agent`), merges their findings into P1 Critical, P2 Major and P3 Minor tables, and ends the report with `GATE: FAIL` if any P1 finding exists or `GATE: PASS` otherwise. To run it interactively, start Claude Code in this folder and enter `/review input/pr.diff`. To run it headless, for example in CI, use `claude -p "/review input/pr.diff" --model sonnet`. Input that is not a unified diff is rejected with `GATE: FAIL — invalid input: not a unified diff`.

## Screenshot

![Review report](screenshot.png)
