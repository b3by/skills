# Security assessment

Two parts, both required, scaled to what the project exposes: a white-box
review of the source, configuration and dependencies, and black-box checks
through the public interface. Record the candidate SHA, environment, tools
and their versions for both.

Work only against local, disposable environments with fictional data. Live
systems, third-party services, load testing and destructive techniques need
the owner's separate authorization. Stop the services you started and
restore any state you changed.

## White-box review

- Map the trust boundaries: where untrusted data enters (network, files,
  configuration, environment, plugin code, messages) and what it can reach.
- Trace untrusted input to dangerous sinks: SQL, shell and subprocess calls,
  `eval`/`exec`, deserialization (`pickle`, YAML), dynamic imports, file paths,
  templates, outgoing HTTP requests. Check each against the rules in
  [`python-tools` security](../../python-tools/references/security.md).
- For services: authentication, authorization on every object and role
  boundary, session and token handling, sensitive fields in responses, logs
  and errors.
- Secrets: search the tree and history for committed credentials and keys.
  Report their location and type, never their value.
- Configuration: debug settings, default bind addresses, permissions of
  created files, CI workflow permissions and secrets usage, container images.
- Dependencies: audit the locked dependencies, and check each reported
  advisory against current sources: is the vulnerable code reachable, and is
  a fixed version available? Don't upgrade anything during the review.

  ```bash
  uv sync --locked --all-extras
  uv run --with pip-audit pip-audit --skip-editable
  uv run ruff check . --select S
  ```

## Black-box checks

Test the built candidate through its public interface only, the way a user
or attacker would reach it.

- **Libraries:** call the public API with hostile input: oversized values,
  wrong types, path traversal (`../`), malformed files, unexpected encodings.
  Check that it fails with documented exceptions, without leaking data,
  hanging or writing outside the intended locations.
- **Services:** start the candidate locally and probe its endpoints:
  unauthenticated access, one fictional user reading or changing another
  user's objects, privilege boundaries, expired or replayed tokens, invalid
  and oversized input, error responses that leak internals, and size or rate
  limits. A refusal status isn't enough: verify that the data wasn't read or
  changed.
- **CLIs:** hostile arguments, file paths and environment variables.

Record each case: identity or role, preconditions, the request or call
(sanitized), expected result, actual result and the state you observed. A
local run can't verify production TLS, proxies or cloud configuration; mark
those as unverified.

## Findings

- Give each finding an ID, an impact-based severity (critical, high, medium,
  low, informational) and a confidence (confirmed, suspected).
- Include prerequisites, affected components, reproduction, impact, the fix
  and how to verify it. Separate new regressions from existing issues.
- A clean scan means only that the scan found nothing, never that there are
  no vulnerabilities. Triage scanner output by hand before reporting it.
- Keep credentials, personal data and working exploit details out of public
  issues. Publish a redacted summary and ask the owner for a private channel
  for the details. Never publish a security advisory without authorization.
