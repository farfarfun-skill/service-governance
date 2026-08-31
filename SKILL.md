---
name: service-release-governance
description: Govern how npm and Python services are built, installed, and started across development and production environments. Use when Codex prepares, reviews, troubleshoots, or documents a service release; configures package publishing or installation for a private/public npm registry or a private Python index/PyPI; uses funbuild; writes development or production startup commands for a Node.js or Python service; designs a service's CLI entrypoint; or audits whether production is isolated from local source changes.
---

# Service Release Governance

Development and production run the exact same shape of flow: build a package, install it, and start it through the package's own CLI. The only sanctioned differences are where the installed package comes from and which config file is passed. Development installs a package built fresh from the current working tree; production installs an immutable formal package published to an artifact repository. This skill covers npm (Node.js) and Python services only.

## Select The Mode

| Mode | Package source | Required flow |
| --- | --- | --- |
| Development | Package built from the current working tree, reinstalled locally on every start | Clear previous local build/install -> build from current source -> install the built package locally -> `<cli> server start` |
| Production | Versioned formal package installed from the artifact registry/index | Build -> publish -> install exact version from the registry -> `<cli> server start` |

`start`/`stop`/`restart`/`run` never take a `dev`/`prod` argument and never branch on one. The installed package alone determines what runs; a host only ever has one active install at a time, and whoever installed it (a developer running the local build, or ops installing a pinned registry version) already knows which one it is. The only steps that are genuinely asymmetric between the two modes are `install` (always the local-build flavor) and `publish` (always the release-to-registry flavor).

Never substitute one mode's package source for the other's: development must not install or start the exact registry-pinned production version in place of a local rebuild, and production must never install or start a locally built package, an editable install, a local package file (`file:` / local wheel or sdist path), or a workspace link. Do not let either process import modules from the source workspace through its working directory, `PYTHONPATH`, `NODE_PATH`, `NODE_OPTIONS --require`, or an equivalent override once the package is installed — always run the installed copy.

Ad hoc hot-reload tooling (`npm run dev`, `vite dev`, `uv run --reload`) is still fine as an unmanaged, fast-iteration workflow while actively editing code. It is not a substitute for the lifecycle script's `install`-then-`start`/`run` flow, which exists to prove the current code is installable and startable exactly like production.

## Entrypoint Contract

Every installable service package — npm or Python — exposes itself as a single named CLI, never a bare script or module path. Both modes start the service by invoking that installed CLI's `server start` subcommand with an identical command shape; only the `--config` path and which package got installed differ between them.

1. Expose a stable CLI command matching the product/service name (e.g. `funflix`), not the package's internal directory or filename:
   - npm: declare it in `package.json`'s `bin` field (`"bin": {"funflix": "./bin/funflix.js"}`), so installing the package puts `funflix` on `PATH`.
   - Python: declare it under `[project.scripts]` in `pyproject.toml` (`funflix = "funflix.cli:main"`), so installing the package puts `funflix` on `PATH` as a console script.
2. Structure that CLI as a subcommand tree with two groups:
   - A `server` group that owns runtime lifecycle only: at minimum `<cli> server start`, plus `server stop`/`server status` when the service needs them, so the CLI's own subcommands can back a `scripts/setup.sh` dispatch (see [Bash Service Guide](../bash-service-guide/SKILL.md)) instead of the script reimplementing lifecycle logic itself.
   - Top-level self-management subcommands that own the CLI's own package lifecycle: `<cli> upgrade [version]` (install the latest, or an explicit, version via the ecosystem's package manager), `<cli> rollback <version>` (install an explicit older version — this is how you roll back, not by reusing an install action), and `<cli> uninstall` (stop the running server first if one is live, then remove the package). Do not add a `<cli> install` subcommand: the CLI cannot install itself before it exists, so the very first install always goes through `pip install`/`npm install -g`/`funbuild install` directly.
3. Make configuration file-driven: `<cli> server start [--config <path>]`, where `--config` accepts `.json`, `.toml`, or `.env` and the CLI selects the parser by file extension. When `--config` is omitted, the CLI resolves a hardcoded default path baked into the package itself, following the XDG convention `${XDG_CONFIG_HOME:-~/.config}/<org>/<cli-name>/config.toml` (e.g. `~/.config/farfarfun/funflix/config.toml`). Production can rely on this default entirely — ops just places the file there and runs `<cli> server start` with no flags. `--config` remains available as an explicit override, which is how development typically points at a repo-tracked, non-secret file such as `config/dev.toml`.
4. Support discrete flags for the parameters operators change most often (`--port`, and similarly named flags for host/workers/etc.), and make an explicit flag override the same key from `--config` when both are given. This lets a start command rely on a config file while still allowing a one-off override without editing or duplicating that file.
5. `start`/`run` in the lifecycle script must call exactly `<installed-cli> server start [--config <path>] [--port ...]` against whatever package is currently installed — never a source-tree script, a raw `node`/`python -m` invocation bypassing the CLI, or a dev server left running outside the lifecycle script's control. The command shape is identical for a locally built install and a registry-pinned install; only the config path and, implicitly, which package got installed differ.
6. `<cli> server status` must report the installed package's version (e.g. via `pip show`/`npm ls -g`/the CLI's own `--version`) alongside PID/port, so operators can confirm what is actually running without a separate lookup.

## Build And Install For Development

1. Clear prior local build output and any previously locally-installed copy of the package before rebuilding, so a stale artifact can never masquerade as current code.
2. Build from the current working tree, including uncommitted changes.
   - For a repository `funbuild` already auto-detects (npm/pnpm/yarn, uv, Poetry, or a hybrid uv+npm repo), prefer `funbuild install`. It runs the same clear -> build -> local-install sequence `funbuild build` uses for a formal release, minus the version bump, publish, push, and tag — exactly the dev-local shape of the flow, with no risk of accidentally publishing or tagging.
   - Otherwise use the project's existing ecosystem-native build and local-install commands (e.g. `npm run build` then `npm pack && npm install -g ./<name>-<version>.tgz`, or `uv build`/`python -m build` then `pip install dist/*.whl --force-reinstall`).
3. Confirm the installed CLI now resolves to the freshly built code (e.g. check the reported version or a build marker), not a previous install.
4. Start via [the shared entrypoint contract](#entrypoint-contract): `<installed-cli> server start`, optionally passing `--config <dev-config-path>` to point at a repo-tracked dev config instead of the CLI's built-in default.

## Build, Publish, And Install For Production

Inspect the repository before choosing commands. Reuse its manifest, lockfile, build wrapper, publishing configuration, and existing release automation. Do not add a second release path when one already works.

1. Identify every independently deployed service, whether it is npm-based or Python-based, its package name, formal version, and intended source commit or tag.
2. Reject development versions — Python dev/local versions (`.devN`, `+local`), npm prereleases/tags (`-alpha`, `-rc`, `:next`), `workspace:*`/`file:`/`link:` dependency protocols, or mutable aliases (`latest`, a moving branch tag) — unless the user explicitly requests a prerelease environment.
3. Select the repository already configured for that ecosystem: a private npm registry (`.npmrc` `registry=`/scoped `@org:registry=`) or npmjs for npm; a private Python index (`pip.conf`/`PIP_INDEX_URL`/`[[tool.uv.index]]`) or PyPI for Python. Prefer an organization-controlled private repository. Use a public repository only when public distribution is required or no approved private repository exists.
4. Build from the intended committed source and publish. Prefer `funbuild build` (alias `funbuild release`) whenever the repository is one it already auto-detects — npm/pnpm/yarn, uv, Poetry, or a hybrid uv+npm repo. It runs the whole pipeline in one command: pull, bump the version, clean prior build output, build, install the built artifact locally as a sanity check, publish to the configured registry/index, clean again, commit and push, and tag. This is also the canonical implementation of a `setup.sh publish` action — do not hand-roll a separate build/publish sequence when `funbuild` already covers the ecosystem. Fall back to the project's existing ecosystem-native build and publish command (`npm publish`, `python -m build` + `twine upload`, `poetry publish`, `uv build`/`uv publish`) only when `funbuild` does not detect the project or the repository already has different, working release automation.
5. Publish a new immutable version. Never overwrite or reuse a released formal version — npm rejects republishing an existing version by default; a Python index must not be force-overwritten.
6. Create a clean production-like environment, install the exact version from the selected repository, and confirm resolution did not fall back to a local path, `node_modules` symlink, or unintended index/registry.
7. Start via [the shared entrypoint contract](#entrypoint-contract): `<installed-cli> server start`, relying on the CLI's built-in default config path unless the deployment explicitly passes `--config <prod-config-path>`. Run the smallest smoke check that proves the installed version starts.

Do not invent repository URLs, credentials, signing keys, or package coordinates. Stop and request the missing value when repository configuration does not establish them. Keep credentials in the ecosystem's supported secret store or CI secret mechanism (`NPM_TOKEN`, `TWINE_PASSWORD`/`UV_PUBLISH_TOKEN`, etc.); never write them into source, `.npmrc`/`pip.conf` committed to the repo, commands shown in logs, or release records.

Rolling back a bad production release is the same shape of flow with an older, explicit version — use `<installed-cli> rollback <version>` when the CLI supports it, or otherwise reinstall the exact older pinned version through the ecosystem's package manager. There is no separate `setup.sh`-level rollback action and no reason to reuse a generic `install` action for it.

## Apply The Ecosystem Contract

| Ecosystem | Manifest / lockfile | Registry config | Build/install path |
| --- | --- | --- | --- |
| npm | `package.json` + `package-lock.json`/`pnpm-lock.yaml`/`yarn.lock` | `.npmrc` (`registry=`, `@scope:registry=`, `//registry/:_authToken=`) | Dev: `funbuild install` (or `npm run build` then `npm pack && npm install -g ./<name>-<version>.tgz`). Prod: `funbuild build`/`funbuild release` (or `npm run build` -> `npm publish` with `version` bumped and no `-dirty`/local changes) -> install `name@exact-version` (`npm ci`-friendly) into a clean root |
| Python | `pyproject.toml`/`setup.cfg` + `uv.lock`/`poetry.lock`/pinned `requirements*.txt` | `pip.conf`/`PIP_INDEX_URL`/`~/.pypirc`/`[[tool.uv.index]]` | Dev: `funbuild install` (or `uv build`/`python -m build` then `pip install dist/*.whl --force-reinstall`). Prod: `funbuild build`/`funbuild release` (or `python -m build` + `twine upload`, `poetry publish`, or `uv build && uv publish`) -> install an exact version (`pip install pkg==X.Y.Z`, `uv pip install pkg==X.Y.Z`) into a clean venv |

Both rows start the same way once installed: `<installed-cli> server start` (see [Entrypoint Contract](#entrypoint-contract)) — no `dev`/`prod` argument on the command itself. Only the config path (explicit override vs. the CLI's built-in default) and which package got installed differ.

A repository with a root `pyproject.toml` ([project]) plus a buildable `package.json` is a hybrid repo; `funbuild build`/`funbuild install` builds, installs, and (for `build`) publishes both halves with one command and writes one shared version. Do not split it into two separate build/publish pipelines unless the two halves are actually released independently.

Treat repository-specific commands as project configuration, not universal constants. Private-first means the private registry/index is the selected source for the service package; public fallback must be explicit and must not silently shadow or be shadowed by an internal package with the same name (npm scoping, Python index priority).

## Ecosystem-Specific Rules

### npm / Node.js

- Ad hoc hot-reload (`npm run dev`, `vite dev`) is fine for interactive coding outside the lifecycle script. The lifecycle script's `install` action must clear previous build output and any previous local install, rebuild (`npm run build`), and install the built package locally so the `<cli>` command reflects current source; its `start`/`run` action then invokes `<cli> server start` — the same contract as production, with no `dev`/`prod` argument on the command itself.
- Never ship `dependencies` that use `file:`, `link:`, `workspace:*`, or a Git URL pinned to a branch (not a tag/commit) for a production-facing internal package — pin to a published version instead.
- `npm publish` must run from a clean tree at the tagged commit; reject publishing with uncommitted changes or a `-dirty` version suffix.
- Production install should be reproducible (`npm ci`, or an equivalent lockfile-respecting install) against the exact published version, with `NODE_ENV=production` and dev-only tooling excluded.
- Both dev and prod start commands must resolve `require`/`import` targets from the respectively installed package's `node_modules`/`dist`, never from a `NODE_PATH` override or a path back into the source checkout.

### Python

- Ad hoc `uv run`/`uv run --reload` or a local venv pointing at the checkout is fine for interactive coding outside the lifecycle script. The lifecycle script's `install` action must clear previous build output and any previous local install, rebuild, and force-reinstall the built wheel locally so the `<cli>` command reflects current source; its `start`/`run` action then invokes `<cli> server start` — the same contract as production, with no `dev`/`prod` argument on the command itself.
- Never let a production install resolve to an editable install, a local wheel/sdist path, or a `-e`/`--editable`/local `file://` requirement.
- Reject Python "dev" versions per PEP 440 (`.devN`) and local version identifiers (`+local`) as production releases unless explicitly approved as a prerelease environment.
- Production install must target an isolated virtual environment, install the pinned exact version, and must not read `PYTHONPATH` or `sys.path` overrides that point back into the source checkout.
- Both dev and prod start commands must be the respectively installed console-script entry point (`<cli> server start`) run from outside the source directory — never `python path/to/script.py` or a raw `python -m package` inside the checkout.

For the Bash-level implementation of the `dev`/`publish`/`start` split (PID files, logs, backgrounding), see [Bash Service Guide](../bash-service-guide/SKILL.md) and its [runtime-patterns.md](../bash-service-guide/references/runtime-patterns.md).

## Gate Production

Return `block` when any of these is true:

- Production starts directly from a source checkout, or consumes an editable/linked/local-file install (`pip install -e`, `npm link`, `file:`/`link:`/`workspace:*` dependency).
- Production start does not go through the installed package's CLI `server start` subcommand. (Omitting `--config` is fine when the CLI's built-in default path resolves correctly; it is not by itself a violation.)
- The formal package was not successfully published to the selected registry/index.
- The production install is unpinned, resolves a different version, or cannot prove its registry/index source.
- The release version is mutable, already exists and would be overwritten, or is a development/prerelease version (npm prerelease tag, PEP 440 `.dev`/local version) without explicit approval.
- The installed artifact cannot start or fails its required smoke check.
- `uninstall` removed a package while the service was still recorded as running, instead of stopping it first.
- Required credentials would be exposed or repository identity is unknown.

Return `revise` for missing reproducibility evidence that does not yet prove a violation. Return `allow` only after the formal package is published, installed by exact version from the intended registry/index in a clean environment, and started successfully via its CLI without workspace source access.

Report the mode, ecosystem (npm or Python), package name and version, source commit/tag, registry/index, build/publish command, clean install command, installed artifact identity and reported version, start command (including the `--config` path if one was passed), smoke result, and final decision. Redact credentials.
