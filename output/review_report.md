# PR Review Report — input/pr.diff

## Summary
This PR adds a `POST /auth/login` Flask endpoint with account lockout, session tokens and audit logging, plus a README section for it. The endpoint has two critical security flaws (a hardcoded live API key and SQL injection), an oversized and over-complex handler with no tests, and a README import path that does not match the code.

## P1 Critical
| File:line | Agent | Finding | Suggested fix | Confidence |
|-----------|-------|---------|---------------|------------|
| src/auth/routes.py:10 | security-agent | `API_KEY = "sk-live-4f9a8c7d2b1e4a6f9c3d5e7b8a0f1c2d"` hardcodes a live API key in source. | Load the key from an environment variable or secrets manager (for example `os.environ["AUDIT_API_KEY"]`). Rotate the exposed key, since it is now in git history. | High — exact secret-pattern match on an added line |
| src/auth/routes.py:40 | security-agent | `"FROM users WHERE username = '%s'" % username` | Use a parameterised query: `cur.execute("SELECT id, password_hash, salt, failed_attempts, locked_until FROM users WHERE username = ?", (username,))`. | High — user-controlled `username` formatted directly into SQL |

## P2 Major
| File:line | Agent | Finding | Suggested fix | Confidence |
|-----------|-------|---------|---------------|------------|
| src/auth/routes.py:22 | security-agent | `return hashlib.sha256((salt + password).encode()).hexdigest()` | Use a password KDF such as bcrypt, scrypt or argon2 with a per-user salt, instead of a single SHA-256 pass. | High — fast unsalted-style hash used for password storage |
| src/auth/routes.py:85 | security-agent | `token = hashlib.sha256(("%s:%s:%s" % (user_id, time.time(), salt)).encode()).hexdigest()` | Generate session tokens with `secrets.token_urlsafe(32)`. The current token is derived from a timestamp and user id, not a CSPRNG. | High — session token built from `time.time()` and non-random inputs |
| src/auth/routes.py:53 | security-agent | `return jsonify({"error": "account locked", "retry_after": remaining}), 423` | Return the same response for unknown users and locked accounts, or accept the trade-off knowingly. The current responses (401 vs 423) reveal whether a username exists and whether it is locked. | Medium — distinct responses for unknown and locked accounts |
| src/auth/routes.py:30 | security-agent | `remember = data.get("remember", False)` | Validate that `remember` is a bool and that `username` and `password` are strings before use. A non-string password raises TypeError in `hash_password` and returns a 500. | Medium — user-controlled JSON fields used without type checks |
| src/auth/routes.py:57 | security-agent | `if candidate != stored_hash:` | Compare digests with `hmac.compare_digest(candidate, stored_hash)` to avoid timing side channels. | Low — non-constant-time comparison of secret-derived values, exploitability unclear |
| src/auth/routes.py:87 | security-agent | `"INSERT INTO sessions (user_id, token, expires_at) VALUES (?, ?, ?)"` | Store a hash of the session token rather than the raw token, so a database leak does not expose usable sessions. | Low — plaintext session token storage, depends on the DB threat model |
| src/auth/routes.py:26-103 | quality-agent | `login_user` is 78 lines, over the 50-line limit. | Extract the lockout check, the failed-attempt update, the session creation, and the audit call into helper functions. | High — line span counted directly from the added hunk |
| src/auth/routes.py:26-103 | quality-agent | Cyclomatic complexity of `login_user` is about 12 (the `or` on line 27, `if` on lines 33, 44, 50, 57, 60, the `and` on line 50, the ternary on line 84, the `or` on line 33, and two `except` clauses), which is over 10. | Split the function into smaller steps, such as credential lookup, lockout check, password verification, and session issuance. | High — decision points counted on the added lines |
| src/auth/routes.py:70-77 and 93-101 | quality-agent | The audit `requests.post` block with the same headers, timeout, and `except requests.RequestException: pass` is duplicated, differing only in the event name and user/ip payload. | Move this into a helper such as `send_audit_event(event, username, ip)` and call it from both branches. | High — the two blocks are visibly identical apart from the event string |
| src/auth/routes.py:16-22 and 25-103 | quality-agent | New public functions `get_db`, `hash_password`, and `login_user` are added with no unit tests in this diff. | Add unit tests for `hash_password` (known digest), `get_db`, and `login_user` (success, bad password, locked account, and lockout threshold). | Medium — no test files appear in this diff, but tests could live in a separate PR |
| src/auth/routes.py:26 | doc-agent | Public route handler `login_user()` has no docstring. It is a new public API endpoint (POST /auth/login) with no docstring describing request fields, responses, or status codes. | Add a docstring covering the JSON body (`username`, `password`, `remember`), the success response (`token`, `expires_in`), and the 400, 401, and 423 responses. | High — new public function with no docstring on the added `def` line |
| src/auth/routes.py:16 | doc-agent | Public function `get_db()` has no docstring. | Add a one-line docstring, e.g. "Return a new SQLite connection to DB_PATH. Caller must close it." | High — new public function with no docstring on the added `def` line |

## P3 Minor
| File:line | Agent | Finding | Suggested fix | Confidence |
|-----------|-------|---------|---------------|------------|
| src/auth/routes.py:61 | quality-agent | Magic number `900` (lockout duration in seconds) is used without a named constant. | Define `LOCKOUT_SECONDS = 900` at module level. | High — literal visible on added line |
| src/auth/routes.py:84 | quality-agent | Magic numbers `30 * 24 * 3600` and `3600` (session TTLs) are inlined. | Define `REMEMBER_TTL_SECONDS` and `DEFAULT_TTL_SECONDS` constants. | High — literals visible on added line |
| src/auth/routes.py:74 and 98 | quality-agent | Magic number `timeout=2` for the audit HTTP call is repeated in two places. | Define `AUDIT_TIMEOUT_SECONDS = 2` and reference it in both calls. | High — literal visible on added lines |
| src/auth/routes.py:20 | doc-agent | `hash_password(password, salt)` has no type hints on its public parameters. | Annotate as `password: str, salt: str` and return `-> str`. | High — unannotated parameters on added public function signature |
| src/auth/routes.py:39-40 | doc-agent | Raw SQL built with `%` formatting has no comment. This is non-obvious and conflicts with the parameterised-query standard. A reader cannot tell whether this is intentional. | Add an inline comment explaining the query, and replace the string formatting with a parameterised `cur.execute(query, (username,))` call. Coordinate with the security finding on this line. | Medium — complex query construction on added lines lacks explanation |
| src/auth/routes.py:60-62 | doc-agent | Lockout logic is undocumented. It locks for 900 seconds at 5 failures and resets `failed_attempts` to 0 when the lock is set. The magic numbers and reset behaviour are not explained. | Add an inline comment stating the lockout policy, e.g. "Lock for 15 minutes after MAX_ATTEMPTS consecutive failures and reset the counter." Consider naming 900 as a constant. | Medium — non-obvious policy logic with magic number on added lines |
| src/auth/routes.py:84-85 | doc-agent | Session token derivation (SHA-256 of user_id, timestamp, and salt) and the `remember` TTL choice (30 days vs 1 hour) have no explanation. | Add an inline comment describing how the token is generated and the expiry policy. | Medium — non-obvious token and TTL logic on added lines |
| README.md:14-20 | doc-agent | The README section documents `POST /auth/login` only briefly. It does not describe the request body, the `remember` option, the response shape, or the error codes (400, 401, 423). | Expand the Authentication section with request and response examples and the status codes. | Medium — endpoint is mentioned but its contract is not fully documented in the added README text |
| README.md:18-20 | doc-agent | The README import path `acme.auth.handlers` does not match the diff. The function is defined in `src/auth/routes.py`, and `src/auth/__init__.py` exports only `auth_bp`, so `login_user` is not importable from `acme.auth.handlers` as shown. | Update the import to the actual module path, or add a `handlers` module that exposes `login_user`. | Medium — import path in README is inconsistent with the file layout in the diff, but package root mapping is not visible |
| CHANGELOG (not in diff) | doc-agent | This PR adds user-facing authentication behaviour and a new endpoint, but the diff includes no CHANGELOG entry. | Add a CHANGELOG entry under the unreleased section for the new login endpoint and its lockout policy. | Low — no CHANGELOG file is visible in the diff, so its existence cannot be confirmed |

## Run Info
input/pr.diff · 2026-11-18 · coordinator: sonnet · sub-agents (haiku): security-agent, quality-agent, doc-agent

---
GATE: FAIL
