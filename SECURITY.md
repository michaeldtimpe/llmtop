# Security Policy

## Reporting a Vulnerability

If you believe you have found a security vulnerability in `llmtop`, please report it privately so it can be triaged and fixed before public disclosure.

- **Preferred:** open a [private security advisory](https://github.com/michaeldtimpe/llmtop/security/advisories/new) on GitHub.
- **Alternative:** email michaeldtimpe@gmail.com with the details.

Please include:

- A description of the issue and the impact you believe it has.
- Steps to reproduce, ideally with a minimal proof-of-concept.
- The version / commit you observed it on, plus your OS and Python version.

You should expect an initial response within 7 days. Please do not file a public issue or PR for security reports until a fix has been released.

## Scope

`llmtop` is a local, read-only macOS observability tool. It:

- reads kernel stats via `vm_stat`, `sysctl`, and `lsof`,
- inspects open file descriptors of other processes via `psutil`,
- talks to local model-server APIs (ollama on `127.0.0.1:11434`, omlx on `127.0.0.1`).

Reports about any of the following are in scope:

- arbitrary command execution, file write, or privilege escalation triggered by `llmtop`'s parsing of process names, file paths, API responses, or CLI arguments,
- credential or token leakage (e.g. via the omlx API key in `~/.omlx/settings.json`),
- log files (`--log`, `--jsonl`) containing data that should not be persisted,
- network requests to anything other than the documented localhost endpoints.

Out of scope:

- vulnerabilities in upstream dependencies (`psutil`, `httpx`, `rich`, etc.) that are not reachable through `llmtop`. Please report those upstream.
- behavior that requires an attacker who already has the ability to run arbitrary code as the user running `llmtop`.

## Supported Versions

Only the latest commit on `main` is supported. Fixes will be released as new commits / tags; older tags will not be patched.
