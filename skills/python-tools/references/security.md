# Secure coding

Rules for code that handles untrusted input, files, processes, network
access and secrets. The ruff `S` rules catch many of these automatically.
Suppress one only for a specific line, with the rule code and the reason:

```python
subprocess.run(command, check=True, timeout=30)  # noqa: S603 - fixed list
```

## Untrusted input

- Never `eval` or `exec` data from configuration, files or the network.
  Parse literals with `ast.literal_eval`, and structured data with `json`,
  `tomllib` or a pydantic model.
- Never unpickle (`pickle`, `marshal`, `shelve`) data that crossed a trust
  boundary: sockets, HTTP, downloads, user files. Use JSON or an explicit
  binary layout (`struct`).
- For YAML, use `yaml.safe_load`, never `yaml.load`.
- Validate external input at the boundary (types, ranges, allowed keys),
  preferably with a pydantic model. Reject unknown keys, and never `setattr`
  arbitrary names onto objects.
- Build dynamic imports (`importlib.import_module`) only from an allowlist or
  from validated identifiers, never from a raw request string.
- Use bound parameters for SQL, never f-strings (see
  [postgres.md](postgres.md)).

## Filesystem

- Resolve paths and check that they stay under the intended root:

  ```python
  root = base_dir.resolve()
  target = pathlib.Path(root, user_supplied_name).resolve()

  if not target.is_relative_to(root):
      raise ValueError(f'path escapes base directory: {user_supplied_name}')
  ```

- Extract archives with `tarfile.extractall(path, filter='data')`. For zip
  files, reject members that are absolute or contain `..`.
- Create temporary files with `tempfile.mkstemp` or `TemporaryDirectory`.
  Never hardcode `/tmp/...`.
- Create sensitive files with restrictive permissions
  (`os.open(path, os.O_CREAT | os.O_WRONLY, 0o600)`).

## Processes and network

- Call `subprocess.run` with an argument list, `shell=False` (the default),
  `check=True` and a timeout:

  ```python
  subprocess.run(
      ['git', 'clone', '--depth', '1', repo_url, str(dest)],
      check=True,
      capture_output=True,
      timeout=60,
  )
  ```

  Never interpolate input into a shell string.
- Give every HTTP or socket call a timeout, and keep TLS verification on.
- Bind servers to `127.0.0.1` by default. Listening on `0.0.0.0` must be an
  explicit setting.

## Secrets and randomness

- Keep credentials out of code, committed configuration, fixtures and logs.
  Read them from the environment or a secret store (as `SecretStr` settings,
  see [pydantic.md](pydantic.md)), and fail clearly when they're missing.
- Never log secrets, tokens, full connection URLs or large binary payloads.
- Use `secrets` for tokens, ids and anything that must not be guessable.
  Use `random` only for simulation and jitter.
- Compare secrets with `hmac.compare_digest`.

## Supply chain

- Commit the lockfile, and install in CI with `uv sync --locked`.
- Audit dependencies with
  `uv run --with pip-audit pip-audit --skip-editable`.
- Publish with trusted publishing, never with long-lived tokens in the
  repository.
