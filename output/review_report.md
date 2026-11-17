# PR Review Report — input/pr.diff

## Summary
This PR adds a `POST /auth/login` endpoint (`src/auth/routes.py`) with lockout handling, session creation and audit logging, plus a README section. It contains two P1 security defects (a hardcoded API key and an injectable SQL query), an oversized, complex and untested handler, and a stale README import path.

## P1 Critical
| File:line | Agent | Finding | Suggested fix |
|-----------|-------|---------|---------------|
| src/auth/routes.py:10 | security-agent | Hardcoded live API key: `API_KEY = "sk-live-4f9a8c7d2b1e4a6f9c3d5e7b8a0f1c2d"`. The same key is sent in the audit request headers (lines 72 and 96), so it is exposed in source control. | Remove the key from source and load it from an environment variable or secrets manager. Rotate the exposed key. |
| src/auth/routes.py:40 | security-agent | SQL injection: `"FROM users WHERE username = '%s'" % username` builds the query by string formatting from user-supplied input. A crafted username such as `' OR '1'='1` changes the query logic (lines 39-40 form one statement). | Use a parameterised query, e.g. `cur.execute("SELECT id, password_hash, salt, failed_attempts, locked_until FROM users WHERE username = ?", (username,))`. |

## P2 Major
| File:line | Agent | Finding | Suggested fix |
|-----------|-------|---------|---------------|
| src/auth/routes.py:22 | security-agent | Password hashed with a single fast SHA-256 pass: `return hashlib.sha256((salt + password).encode()).hexdigest()`. Fast unsalted-iteration hashes are weak for password storage. | Use a dedicated password KDF such as bcrypt, scrypt, or argon2, or `hashlib.pbkdf2_hmac` with a high iteration count. |
| src/auth/routes.py:57 | security-agent | Non-constant-time comparison of the password hash: `if candidate != stored_hash:` | Use `hmac.compare_digest(candidate, stored_hash)`. |
| src/auth/routes.py:85 | security-agent | Session token is a SHA-256 of `user_id`, `time.time()` and `salt`, which is predictable and not drawn from a CSPRNG. The token is also stored in plaintext at line 88. | Generate the token with `secrets.token_urlsafe(32)` and store only a hash of it in the `sessions` table. |
| src/auth/routes.py:73 | security-agent | Sensitive data logged: `json={"event": "login_failed", "user": username, "ip": client_ip}` sends the submitted username and client IP to an external audit service on every failed login. The same applies at line 97 for `login_ok`. | Send a pseudonymous user ID instead of the raw username, and confirm the audit service's retention and access policy covers IP and user data. |
| src/auth/routes.py:26 | quality-agent | `login_user` spans lines 26-103 (about 78 lines), over the 50-line limit. | Split into helpers such as `_fetch_user`, `_record_failed_attempt`, `_create_session`, and `_send_audit_event`. |
| src/auth/routes.py:26 | quality-agent | Cyclomatic complexity is about 12. Decision points include the `or {}` default (27), `if not username or not password` (33), `if row is None` (44), `if locked_until and ...` (50), `if candidate != stored_hash` (57), `if failed_attempts >= MAX_ATTEMPTS` (60), the `remember` ternary (84), and two `except` branches (76, 100). | Move the failed-login and successful-login branches into helpers, and validate input in a separate function. |
| src/auth/routes.py:70 and :94 | quality-agent | The audit `requests.post` block with its try/except is duplicated (lines 70-77 and 94-101). | Extract `send_audit_event(event, username, client_ip)` and call it from both branches. |
| src/auth/routes.py:16, :20, :26 | quality-agent | New public functions `get_db`, `hash_password`, and `login_user` have no unit tests, and the diff touches no `tests/` files. | Add tests covering `hash_password` output, the lockout path, and the success and failure paths of `login_user`, with the DB and `requests` mocked. |
| src/auth/routes.py:16 | quality-agent, doc-agent | Public function `get_db()` has no docstring. It returns an open `sqlite3` connection that callers must close, which is not stated anywhere. | Add a docstring stating that it returns a new connection to `DB_PATH` and that the caller is responsible for closing it. |
| src/auth/routes.py:26 | quality-agent, doc-agent | Public function `login_user` has no docstring. | Add a docstring covering the JSON fields (`username`, `password`, `remember`), the responses (200 with `token` and `expires_in`, 400, 401, 423 with `retry_after`), and the side effects (session insert, audit event). |
| src/auth/routes.py:25 | doc-agent | New endpoint `POST /auth/login` is documented only by one README sentence. It has no request schema, no status code list (400, 401, 423), and no response shape. | Document the request body fields, the possible status codes, and the response payload in the README or in API docs. |

## P3 Minor
| File:line | Agent | Finding | Suggested fix |
|-----------|-------|---------|---------------|
| README.md:19 | doc-agent | Stale reference: `from acme.auth.handlers import login_user` points to a module that does not exist in this diff. The handler is `login_user` in `src/auth/routes.py`, and it is a Flask route handler, not an importable helper. | Remove the import and document the endpoint (`POST /auth/login`) instead, or change the import to the real path (for example `from src.auth import auth_bp`), matching the project's import root. |
| src/auth/routes.py:16, :20, :26 | quality-agent | Public functions lack type hints. | Annotate them, for example `def get_db() -> sqlite3.Connection`, `def hash_password(password: str, salt: str) -> str`, and `def login_user() -> tuple[Response, int]`. |
| src/auth/routes.py:20 | doc-agent | `hash_password(password, salt)` has no type hints on its public parameters. | Annotate the signature, for example `def hash_password(password: str, salt: str) -> str:`. |
| src/auth/routes.py:61 | quality-agent | Magic number `900` for the lockout duration in seconds. | Define `LOCKOUT_SECONDS = 900` at module level. |
| src/auth/routes.py:84 | quality-agent | Magic numbers `30 * 24 * 3600` and `3600` for session TTLs. | Define `REMEMBER_TTL_SECONDS` and `DEFAULT_TTL_SECONDS` constants. |
| src/auth/routes.py:74 and :98 | quality-agent | Magic number `timeout=2` for the audit request. | Define `AUDIT_TIMEOUT_SECONDS = 2`. |
| src/auth/routes.py:60 | doc-agent | Lockout logic (threshold check, 900-second lock, counter reset to 0) has no inline comment, and the `900` literal is unexplained. | Add a comment such as "Lock account for 15 minutes after MAX_ATTEMPTS failures and reset the counter", or define a named constant such as `LOCKOUT_SECONDS = 900`. |
| src/auth/routes.py:84 | doc-agent | Session TTL logic (30 days when `remember` is set, otherwise 1 hour) and the token derivation on line 85 have no inline comments. The literals `30 * 24 * 3600` and `3600` are unexplained. | Add a comment stating the TTL policy and what the token is derived from, and consider named constants for the TTL values. |

## Run Info
input/pr.diff · 2026-11-16 · coordinator: sonnet (claude-sonnet-5-5), sub-agents: haiku · security-agent, quality-agent, doc-agent

Notes: the `get_db` and `login_user` docstring findings were reported by both quality-agent and doc-agent and are listed once each. doc-agent's CHANGELOG finding is omitted because it is unconfirmable from the diff (CLAUDE.md: no findings unsupported by the diff).

---
GATE: FAIL
