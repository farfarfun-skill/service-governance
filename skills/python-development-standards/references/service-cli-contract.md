# Python Service CLI Contract

Use this checklist when a Python project is a long-running service that gets installed and started/stopped through its own CLI (typically dispatched by a `scripts/setup.sh` lifecycle script), not just an ordinary library or one-shot command-line tool. It states the Python-specific realization of the cross-ecosystem "install a package, start it through its own CLI" contract that [Service Release Governance](../../service-release-governance/SKILL.md) and [Bash Service Guide](../../bash-service-guide/SKILL.md) define at a language-agnostic level.

## Expose a Single Named CLI

- Declare the CLI under `[project.scripts]` in `pyproject.toml`, named after the product/service (`funflix = "funflix.cli:main"`), so installing the package puts `funflix` on `PATH` as a console script. Never rely on a bare script path or `python -m package` as the primary entrypoint.
- Structure the CLI as a subcommand tree with two groups:
  - A `server` group that owns runtime lifecycle only: `server start` (detaches and backgrounds itself), `server run` (identical startup, but stays attached to the invoking terminal — the foreground counterpart of `start`), `server stop`, `server status`.
  - Top-level self-management subcommands that own the CLI's own package lifecycle: `upgrade [version]` (install the latest, or an explicit, version via `pip`/`uv`), `rollback <version>` (install an explicit older version), and `uninstall` (stop the running server first if one is live, then remove the package). Do not add an `install` subcommand — the CLI cannot install itself before it exists, so the very first install always goes through `pip install`/`funbuild install` directly.

## Make Configuration File-Driven

- `server start [--config <path>]`, where `--config` accepts `.json`, `.toml`, or `.env` and the CLI selects the parser by file extension.
- When `--config` is omitted, resolve a hardcoded default path baked into the package, following the XDG convention `${XDG_CONFIG_HOME:-~/.config}/<org>/<cli-name>/config.toml` (e.g. `~/.config/farfarfun/funflix/config.toml`). Production can rely on this default entirely; development typically points `--config` at a repo-tracked, non-secret file such as `config/dev.toml`.
- Support discrete flags for the parameters operators change most often (`--port`, and similarly named flags for host/workers/etc.), with an explicit flag overriding the same key from `--config` when both are given.

## Own PID Lifecycle Inside the CLI

- `server start` and `server run` write a `<cli-name>.pid` file into the same directory as whichever config path was actually resolved (default path or `--config` override) — e.g. `~/.config/farfarfun/funflix/funflix.pid` next to `~/.config/farfarfun/funflix/config.toml`.
- `server status` locates the process by reading that same PID file, and reports the installed package's version (e.g. via `pip show` or the CLI's own `--version`) alongside PID/port — not just liveness.
- `server stop`'s actual termination step goes through `funshell port <port> --kill` against the port `server start` bound to, rather than hand-signaling the PID read from the PID file. Depend on the `funshell` package directly and call its port-kill API in-process — do not shell out to the `funshell` command from a Python CLI; that shell-out form is for non-Python callers.
- This keeps the CLI fully self-sufficient for `start`/`run`/`stop`/`status` even on a host with no lifecycle script or repository checkout at all.

## Build and Install for Development

- Clear any previous local build output and previously locally-installed copy before rebuilding, so a stale artifact can never masquerade as current code.
- Prefer `funbuild install` when the project is one it auto-detects (uv, Poetry, or a hybrid uv+npm repo) — it clears, builds, and locally installs in one step with no risk of accidentally publishing or tagging.
- Otherwise build with `uv build`/`python -m build`, then force-reinstall the wheel locally (`pip install dist/*.whl --force-reinstall`).
- Confirm the installed CLI now resolves to the freshly built code (reported version or a build marker), not a previous install.
- Ad hoc `uv run`/`uv run --reload` or a local venv pointing at the checkout is fine for interactive coding outside the lifecycle script, but is never what a lifecycle script's `start`/`run` action itself invokes.

## Build, Publish, and Install for Production

- Prefer `funbuild build` (alias `funbuild release`) when the project is one it auto-detects. It runs the whole pipeline in one command: bump the version, clean prior build output, build, install the built artifact locally as a sanity check, publish, clean again, commit/push, and tag. Fall back to `python -m build` + `twine upload`, `poetry publish`, or `uv build && uv publish` only when `funbuild` does not detect the project or different working release automation already exists.
- Reject PEP 440 development versions (`.devN`) and local version identifiers (`+local`) as production releases unless the user explicitly requests a prerelease environment.
- Publish to a private Python index (`pip.conf`/`PIP_INDEX_URL`/`[[tool.uv.index]]`) or PyPI. Prefer an organization-controlled private index; use a public one only when public distribution is required or no approved private index exists. Never overwrite or reuse an already-published version — a Python index must not be force-overwritten.
- Install into a clean, isolated virtual environment, pinned to the exact published version (`pip install pkg==X.Y.Z`, `uv pip install pkg==X.Y.Z`). Never let a production install resolve to an editable install, a local wheel/sdist path, or a `-e`/`--editable`/local `file://` requirement, and never read a `PYTHONPATH`/`sys.path` override that points back into the source checkout.

## Start the Service

`start`/`run` always use the installed console-script entrypoint, regardless of whether the install came from a local build or the registry. Prefer, in order of confidence:

1. The console script from whichever package is currently installed (a locally built copy after a dev install, or the exact formal package after a production install).
2. `python -m package` using that same environment, with the working directory outside the source checkout.

Avoid these as the `start`/`run` command, in either case:

- `python scripts/foo.py` or `python app.py` from an ad hoc working directory.
- `uv run <command>` when it can resolve the current workspace or an editable install.
- A dev-only reload server used as the lifecycle daemon.
- A raw module path, script, editable install, `PYTHONPATH` override, or an implicit `uv run` workspace resolution in place of the installed CLI entrypoint.

In multi-service repositories, keep Python service commands scoped to that service's own root, virtualenv tooling, and ports — do not reuse another service's defaults.
