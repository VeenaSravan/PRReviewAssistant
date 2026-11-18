# Validation Report — PR Review Assistant

Scope: `output/review_report.md` checked against `input/pr.diff`, `CLAUDE.md` and `coordinator.md`; robustness checked with `input/broken_input.txt`. Date: 2026-11-19.

## 1. Correctness — rating and any unsupported findings

**Rating: High.** All `file:line` references match the diff (routes.py line N = diff line N+32). Both P1 findings (hardcoded `sk-live-` key at routes.py:10, SQL injection at routes.py:39-40) are real. The 78-line `login_user` count, the cyclomatic complexity of about 12, the missing docstrings on `get_db` and `login_user`, and `GATE: FAIL` are all correct.

**Factually incorrect**
- **P2, routes.py:22 (password hashing):** The confidence text says "unsalted-style hash" and the fix says "per-user salt", but the code already salts (`salt + password`). The real issue is a single fast SHA-256 pass instead of a password KDF.
- **P2, routes.py:70-77 and 93-101 (duplicated audit block):** The finding says the blocks differ in "event name and user/ip payload". The payloads are identical; only the event string differs. The row's own confidence text says so.

**Unsupported by the diff**
- **P3, "CHANGELOG (not in diff)":** The diff contains no CHANGELOG, and the report itself rates the finding Low confidence because the file's existence "cannot be confirmed". It breaks the rule "Do not add findings not supported by the diff."
- **P3, routes.py:39-40 ("raw SQL has no comment"):** The query is a single `SELECT`, not "complex construction". The defect is the injection, which is already P1, so this row only duplicates it.
- **P3, README.md:18-20 (import path), partly:** The reasoning that `src/auth/__init__.py` exports only `auth_bp` is irrelevant to `acme.auth.handlers`. The diff shows no `acme` package or `handlers` module. Only the mismatch with `src/auth/routes.py` is supported.

**Overstated**
- **P1, routes.py:10, "now in git history":** The diff shows the key being added, not that it was ever committed to a shared branch.
- **P2, routes.py:30 (`remember` validation):** A non-bool `remember` breaks nothing. The only concrete failure is a non-string `password` raising TypeError, so the finding belongs on the `password` handling at routes.py:29 and 35.

## 2. Completeness — gaps against the spec

The report has every structural element `coordinator.md` requires: title with file name, two-sentence Summary, P1/P2/P3 tables with the five columns, Run Info (input · date · model · sub-agents) and `GATE:` after the `---`. The gate value is correct. Output stays in `output/`, and nothing was committed.

Gaps:
- **Type hints (CLAUDE.md P3):** Only `hash_password` is flagged. `get_db()` and `login_user()` are also new public functions with no type hints.
- **De-duplication (coordinator step 4, previously step 3):** The doc row at routes.py:39-40 restates the P1 SQL injection, and the 900-second lockout constant appears in both a quality row and a doc row.
- **P1 SQL injection row:** The Finding cell holds only a code snippet, with no description.
- **Taxonomy:** The magic-number rows fit only loosely under "style issues". Weak hashing, guessable tokens and user enumeration are not in the taxonomy; P2 is defensible.
- **Run Info date:** 2026-11-18, one day stale.
- **Provenance:** `git status` showed `coordinator.md`, the three agent files and `output/review_report.md` as modified. The report may predate those edits; I did not compare against the committed versions.

## 3. Robustness — broken-input result before the fix

`input/broken_input.txt` contains one line: `zx9 ;;; ### ??? lorem 0000 <<>>`. It has no `diff --git`, `---`/`+++` or `@@` markers.

Before the fix, `coordinator.md` had no step for malformed input. **I did not run the original pipeline on this file**, so there is no observed "before" output. I stopped for two reasons:
1. The agents would have returned empty tables or invented findings, which CLAUDE.md prohibits.
2. Step 6 would have overwritten the valid `pr.diff` report with a meaningless `GATE: PASS`. That file already had uncommitted changes, so it could not be restored from git.

Expected behaviour of the unfixed spec (**inferred, not observed**): sub-agents launch on junk, near-empty or fabricated findings, then `GATE: PASS` written over the real report.

## Issue Fixed — problem, change made, evidence from the re-run

**Problem:** The spec had no handling for input that is not a unified diff. The coordinator would either report a false pass or invent findings.

**Change made:** In `coordinator.md`, a new step 2 validates the input. With no unified-diff markers, it launches no sub-agents and writes `GATE: FAIL — invalid input: not a unified diff` with empty tables and a one-line Summary, then stops. The later steps are renumbered 3-6.

**Evidence from the re-run (2026-11-19):**
- The input file was re-read and is unchanged: a single junk line.
- A search for `^(diff --git|--- |\+\+\+ |@@)` in the file returned 0 matches, so validation fails.
- No sub-agents were launched.
- `output/review_report_broken_input.md` contains empty P1/P2/P3 tables, a Summary giving the reason, Run Info listing "sub-agents: none (input validation failed)", and `GATE: FAIL — invalid input: not a unified diff`.
- `output/review_report.md` was not touched, so the `pr.diff` review is intact.

**Open item:** Step 6 still hardcodes `output/review_report.md`, while the invalid-input report went to a separate file to avoid overwriting the real one. The spec should say where invalid-input reports go, for example `output/review_report_<input-name>.md`.
