# Runtime Patterns

Read this file when `scripts/setup.sh` or a per-service script needs concrete runtime commands or when `prod` behavior is easy to get wrong.

- [Choose The Right Pattern](#choose-the-right-pattern)
- [Python Pattern](#python-pattern)
- [Frontend Pattern](#frontend-pattern)
- [Flutter Web Pattern](#flutter-web-pattern)
- [Backgrounding Pattern](#backgrounding-pattern)
- [Sanity Checks](#sanity-checks)

## Choose The Right Pattern

- If the repository has `pyproject.toml`, `requirements.txt`, or a Python package entrypoint, use the Python pattern for that service.
- If the repository has `pubspec.yaml` with a `dependencies.flutter.sdk: flutter` entry, use the Flutter Web pattern for that service — check this *before* the frontend pattern's `package.json` check, mirroring `funbuild`'s own detection order (its `FlutterBuild` is tried before `NpmFrontendBuild` specifically because a Flutter web project commonly carries a secondary `package.json` for frontend tooling, which would otherwise misclassify it as a plain Node frontend).
- If the repository has `package.json`, `vite.config.*`, `next.config.*`, or a build output directory such as `dist/`, use the frontend pattern for that service.
- Flutter's non-web targets (Android/iOS/desktop app builds) are not a long-running service and fall outside this skill's start/stop/PID lifecycle model — treat their build/release steps as a repository-specific extension, not a `setup.sh` service.
- If the repository has both backend and frontend runtimes, apply the matching pattern per service instead of collapsing them into one command model.
- If neither pattern fits, preserve repository-specific conventions and still honor the core rules from [rules.md](rules.md).

## Python Pattern

`start`/`run` always use the installed console-script entrypoint, regardless of whether the install came from a local build or the registry. Prefer, in order of confidence:

1. A console script from whichever package is currently installed (a locally built copy after `install`, or the exact formal package after a registry install)
2. `python -m package` using that same environment, with the working directory outside the source checkout

Avoid these as the `start`/`run` command, in either case:

- `python scripts/foo.py`
- `python app.py` from an ad hoc working directory
- `uv run <command>` when it can resolve the current workspace or an editable install
- Dev-only reload servers used as the lifecycle script's daemon

Typical split — only `install` and `publish` are environment-specific; `start`/`run`/`stop`/`status` are identical either way:

- ad hoc dev iteration (outside the lifecycle script): local dev server, reloader, or `uv run ... --reload`
- `install` (dev-only): clear previous build/install, rebuild, force-reinstall the wheel locally (`funbuild install` when available)
- `publish` (prod-only): `funbuild build` (alias `funbuild release`) or the existing Python release command
- production install (done outside the repo's `setup.sh`, on the prod host): exact package version from the repository chosen by `service-release-governance`, via `pip`/`uv pip` or the installed CLI's own `upgrade`/`rollback` subcommand
- `start`/`run`: the installed CLI or module entrypoint, no dev-only reload flags or source-path overrides, no `dev`/`prod` argument
- `status`: PID/port plus the installed package's version

In multi-service repositories:

- keep Python service commands scoped to that service's root, virtualenv tooling, and ports
- do not reuse backend defaults for unrelated frontend services

## Frontend Pattern

`start`/`run` always use the installed production server/CLI, regardless of whether the install came from a local build or the registry. Prefer:

1. A documented production server command from whichever package is currently installed
2. An SSR server or static asset command whose `build/` or `dist/` belongs to that installed release, outside the source checkout

Avoid these as the `start`/`run` command:

- `vite dev`
- `next dev`
- Any hot-reload or watch command

Typical split — only `install` and `publish` are environment-specific; `start`/`run`/`stop`/`status` are identical either way:

- ad hoc dev iteration (outside the lifecycle script): `npm run dev`, `pnpm dev`, or framework-equivalent local dev server
- `install` (dev-only): clear previous build/install, rebuild (`npm run build`), install the built package locally (`funbuild install` when available)
- `publish` (prod-only): build and upload the formal package through the configured registry
- production install (done outside the repo's `setup.sh`, on the prod host): exact package version from that registry, via `npm install -g`/`npm ci` or the installed CLI's own `upgrade`/`rollback` subcommand
- `start`/`run`: SSR server or static asset command against whichever installed release is currently on the host, no `dev`/`prod` argument
- `status`: PID/port plus the installed package's version

In multi-service repositories:

- keep ad hoc dev iteration scoped to the service root and `start`/`run` scoped to the installed release root
- do not assume the frontend and backend share the same `publish` pipeline

## Flutter Web Pattern

Flutter web has no daemonizing production CLI of its own — `flutter build web` produces a static `build/web` directory, so a served Flutter web app is just a static-asset service. Reuse the frontend pattern's static-asset serving story rather than inventing a separate lifecycle model.

[Service Release Governance](../../service-release-governance/SKILL.md) still explicitly covers npm and Python packages only — its `pip install`/`npm ci`-from-a-registry model and its ecosystem table do not include Dart/Flutter, and it has no gate rules written against `funpub`. But `funbuild` itself does have a Flutter build type (`FlutterBuild`, detected from a root `pubspec.yaml` whose `dependencies.flutter.sdk` is `flutter`), so `funbuild build`/`funbuild release` and `funbuild install` are real commands here, not a gap to route around by hand. The distribution model is just different from npm/PyPI: publish means uploading a build artifact to a private generic artifact store via `funpub`, not publishing an installable package to a language registry. Keep applying the same underlying principles service-release-governance stands for (immutable versioned artifact, no dev/prod source mixing, no serving straight from the checkout) — just through this ecosystem's actual tools instead of assuming service-release-governance's npm/PyPI-specific commands apply.

`funbuild`'s default behavior for a `FlutterBuild` project, all overridable per-stage via a `funbuild:` block in `pubspec.yaml` (string or list of shell commands):

| Stage | Default command |
| --- | --- |
| build | `flutter pub get` + `flutter build apk --release` + `flutter build web --release` |
| install | none — a no-op unless the project's `pubspec.yaml` defines `funbuild.install` |
| publish | `funpub upload` to the private generic repo `funpackage`: the APK direct, the `build/web` directory zipped first, to remote paths `flutter/<pubspec-name>/apk` and `flutter/<pubspec-name>/web` |
| clean | `flutter clean` |

Two consequences for a web-only service worth calling out explicitly:

- The default `build`/`publish` stage builds and uploads the Android APK too, since `FlutterBuild` doesn't know the repository only ships a web target. If the service is web-only, override `funbuild.build` (and `funbuild.publish` if the APK upload should also be dropped) in `pubspec.yaml` to skip the `flutter build apk` step, e.g. `funbuild: {build: flutter build web --release}`.
- `funbuild install` has no default behavior for Flutter — it does not promote `build/web` into a serving root on its own. If the lifecycle script's `install` action is expected to do that, either configure `funbuild.install` in `pubspec.yaml` to run the promotion command, or run `flutter build web` plus the promotion step directly in the service script instead of assuming `funbuild install` already covers it.

There is no package-manager "install" step for a static bundle the way there is for npm/PyPI — "install" here means promoting a freshly built (or freshly downloaded) `build/web` into the same well-known serving root the static server reads from (e.g. a version-stamped directory plus a `current` symlink/copy step).

`start`/`run` always serve whichever release is currently promoted into that serving root, regardless of whether it came from a local dev build or a downloaded production artifact. Prefer, in order of confidence:

1. A static server already wrapped by an installed CLI's own `server start` (per the primary backgrounding pattern) — see [Providing The Missing CLI](#providing-the-missing-cli-a-self-published-npm-wrapper) below when the repository doesn't already have one
2. A generic static file server (existing repository convention, e.g. an already-vendored static-serving tool) pointed at the promoted `build/web` directory, outside the source checkout

Avoid these as the `start`/`run` command:

- `flutter run` (any target) — this is a debug/hot-reload session tied to a connected device or `web-server`/`chrome` target, not a production server
- `flutter run -d web-server --web-port=...` used as the lifecycle daemon
- Serving directly out of the source checkout's `build/web` without promoting it through the same serving-root convention other releases use

### Providing The Missing CLI: A Self-Published npm Wrapper

Flutter's own build output has no daemonizing executable — `funbuild`/`funpub` version and distribute the static `build/web` artifact, but nothing in that pipeline produces a `server start`/`server run` command. To get option 1 above (the primary, CLI-owned-PID pattern) instead of falling back to option 2's generic static-file-server, build that CLI yourself as a small embedded npm package and publish it independently of the web bundle.

Keep the two artifacts and their publish pipelines separate — building this CLI does not change anything about how the web bundle itself is built or published:

- the Flutter web bundle: built and published via `funbuild`/`funpub`, exactly as described above
- the CLI wrapper: a self-contained npm package, versioned and published via ordinary `npm publish`/`npm install -g`, whose only job is to daemonize, serve, and (optionally) reverse-proxy whatever web bundle is currently promoted into the serving root — it does not embed or replace the bundle itself

Embedding the CLI in the app directory fixes *where* its source lives, not *which version* is compatible with which web build — decide that explicitly, or `install`/production install end up guessing. The simplest option is to keep the CLI package's version in lockstep with the app's own version (e.g. mirror `pubspec.yaml`'s `version` into `package.json` as part of `install`/`publish`) instead of letting npm semver drift independently. If the two genuinely need independent version numbers, pin the compatible pair in one place — a release manifest or a matching git tag — so no step has to guess which CLI version goes with which web bundle.

Layout — embed the CLI package inside the app directory it deploys, one container level down so `extbuild/` can hold other artifact kinds later:

```text
apps/<app>/
├── <Flutter source>
└── extbuild/
    └── npm-cli/
        ├── package.json    # version is this CLI's own source of truth; must not set "private": true
        ├── bin/cli.js
        └── src/            # daemonize / static file serving / reverse proxy
```

Embed rather than create a sibling `<app>-cli` package: the CLI and the app it serves are two halves of one release, built and shipped together — splitting them into separate top-level packages turns "which CLI version matches which build" into something that has to be tracked separately. `extbuild/` itself stays a container, not the package root, so a future second artifact kind (a different packaging or deployment format) has somewhere to live without contending for the directory name.

If the repository uses an npm/pnpm workspace, glob-match members (`apps/*/extbuild/npm-cli`) instead of listing each app by name, so a new app's CLI is picked up automatically once it follows the same layout.

`package.json` must not carry `"private": true` — that field makes `npm publish` fail unconditionally with no `--force` override. Check what the app's scaffolding tool set by default before wiring up publish. Also configure an explicit publish registry/target for the CLI package rather than relying on npm's implicit default — this mirrors `service-release-governance`'s rule that every publish target in the repository must be explicit, never defaulted, so a slip doesn't push a private wrapper to the public registry.

The CLI needs no runtime dependencies — implement daemonizing, static serving, and reverse-proxying with the host language's standard library. A deployment wrapper is not the place to add third-party supply-chain surface.

1. **Self-daemonize.** `start` detaches into the background and must verify a short startup grace period before reporting success — a port collision or similar failure can make the detached process exit almost immediately, and a bare fork-then-return will report success anyway. Do not trust PID existence alone as proof the process is still the one that was started; PIDs get reused, so cross-check the recorded PID's command line (e.g. `/proc/<pid>/cmdline`, or the platform equivalent) against what was launched. Keep PID/log/metadata files under an XDG-style per-user config directory, namespaced by CLI name — not inside the repository or tied to a particular working directory.
2. **Static file serving.** If an existing static-file-serving script is already in production, match its observable behavior exactly when replacing it — don't let a language/runtime swap silently change what the browser sees. At minimum: SPA route fallback, gzip negotiation, conditional requests (304), and content-type-appropriate `Cache-Control` (no-cache for HTML, long-lived/immutable for hashed static assets).
3. **Generic reverse proxy**, when the app needs same-origin API access. Take the upstream host/port as CLI flags rather than hardcoding a specific environment's address resolution, so the package can be exercised standalone outside the repository. Strip hop-by-hop headers, cap request body size, reject ambiguous requests (e.g. both `Transfer-Encoding: chunked` and a duplicate `Content-Length`), and distinguish upstream connection failure from upstream timeout with different error responses.

`start`/`run` invoke the installed CLI's own `server start`/`server run` exactly like the primary backgrounding pattern in [rules.md](rules.md#runtime-files) — no bespoke Flutter-specific process handling in the Bash lifecycle script.

Typical split — only `install` and `publish` are environment-specific; `start`/`run`/`stop`/`status` are identical either way. Once a repository has the CLI wrapper, every step below covers *two* artifacts (the web bundle and the CLI); a repository without one only has the first:

- ad hoc dev iteration (outside the lifecycle script): `flutter run -d chrome` or `flutter run -d web-server --web-port=<port>` for the app; running `bin/cli.js` directly (or an npm script wrapping it) for iterating on the CLI itself
- `install` (dev-only): clear the previous local build/promoted copy, run `flutter build web` from the current working tree (via `funbuild install` only if `pubspec.yaml` configures that stage to do it), promote the fresh `build/web` into the serving root, and `npm install -g` the CLI package from the same working tree so both come from the same commit
- `publish` (prod-only): `funbuild build` (alias `funbuild release`) for the web bundle when the project's default or configured `funbuild.publish` stage matches what the repository wants to ship (drop the APK step first if the service is web-only, per above; otherwise call `flutter build web --release` and `funpub upload` directly), plus `npm publish` for the CLI wrapper against its explicitly configured registry
- production install (done outside the repo's `setup.sh`, on the prod host): `funpub download flutter/<pubspec-name>/web --version <version>` to fetch and promote the web bundle, plus `npm install -g <cli-package>@<version>` to install the matching CLI version — resolve `<version>` from wherever the compatible pair is pinned (see above), not by installing whatever's latest on either side
- `start`/`run`: the installed CLI's `server start`/`server run`, pointed at whatever is currently promoted into the serving root, no dev-only reload flags, no `dev`/`prod` argument
- `status`: PID/port, the promoted web bundle's version, and the installed CLI package's own version — report both once they're independently versioned artifacts

In multi-service repositories:

- keep the Flutter web service's port and static-serving root distinct from any Python or non-Flutter frontend service
- do not assume Flutter web shares a build/publish pipeline with a separate Node-based frontend service — even in a repository that also has one, `funbuild` picks a single build type per manifest, so a Flutter web app and a separate npm frontend are still two independently published services

## Backgrounding Pattern

The primary case: the installed CLI daemonizes itself and owns its own PID file (written next to its resolved config path). Build commands as Bash arrays so arguments remain distinct, and let the CLI handle backgrounding directly — no `nohup`, no script-managed PID file:

```bash
do_start() {
  "${CLI_NAME}" server start ${CONFIG_PATH:+--config "${CONFIG_PATH}"} --port "${PORT}"
}

do_run() {
  exec "${CLI_NAME}" server run ${CONFIG_PATH:+--config "${CONFIG_PATH}"} --port "${PORT}"
}
```

- `do_start` simply invokes the CLI and returns once it reports it has detached; the CLI itself writes `<cli-name>.pid` next to whichever config path it resolved.
- `do_run` uses `exec` so signals and exit status reach the CLI's foreground process directly.
- `do_stop`/`do_status` call `"${CLI_NAME}" server stop`/`"${CLI_NAME}" server status`, which read that same PID file — the script never opens or writes it itself.
- Do not use `eval` or a command string. Use `bash -c` only when shell syntax is genuinely required and parameters can be passed safely.

Fallback case: only when the installed CLI cannot daemonize itself, the Bash script must do the backgrounding and own a PID file itself:

```bash
command=("${CLI_NAME}" server start --port "${port}")
nohup "${command[@]}" </dev/null >>"${log_file}" 2>&1 &
pid=$!
```

Or, if background start reuses the foreground implementation, invoke an internal script action and make the foreground path end in `exec`:

```bash
nohup bash "${SCRIPT_PATH}" __run </dev/null >>"${log_file}" 2>&1 &
pid=$!

do_run() {
  service_command
  cd "${SERVICE_ROOT}"
  exec "${SERVICE_COMMAND[@]}"
}
```

Under the fallback, whichever style you choose:

- Capture `$!` immediately and write it to the PID file atomically.
- Refuse a duplicate start when the existing PID is live; handle stale or invalid PID files explicitly.
- Verify that the child survives a short startup grace period before reporting success.
- Write logs into `.run/`.
- Redirect stdin from `/dev/null` and do not let `start` depend on the terminal staying open.
- In multi-service scripts, ensure the PID and log file path is unique per service (e.g. `.run/api.pid`) — no environment suffix is needed since a host only runs one active install per service.
- If the command daemonizes itself or leaves an unmanaged process tree, use the runtime's native PID mechanism or an existing process supervisor — this is exactly the signal that the primary (CLI-owned) pattern above should be used instead.

## Sanity Checks

- Does `start` call `<cli> server start` and let the CLI itself detach and own its PID file, rather than the script wrapping it in `nohup`?
- Does `run` call `<cli> server run` (not `server start`) via `exec`, staying attached to the terminal?
- Does `status` call `<cli> server status`, which reads the same PID file `start`/`run` wrote, and report the installed package's version alongside PID/port?
- Does `stop` call `<cli> server stop`, trusting the CLI to target its own PID file rather than the script guessing a path?
- Under the non-daemonizing-CLI fallback only: does `stop` target the PID file written by the chosen background start path, send `TERM`, wait for exit, and avoid deleting state while the process is still live? Can a stale or reused PID cause the script to signal an unrelated process?
- Does `start`/`run` avoid reload, watch, and development-only flags regardless of which package is installed?
- Does `start`/`run` resolve only the currently installed package — a fresh local build after `install`, or the pinned registry install — never the source checkout or a stale build output?
- Does the script correctly avoid taking a `dev`/`prod` argument on `start`/`stop`/`restart`/`run`/`status`?
- Does `uninstall`, if the script owns it, stop the service before removing the package?
- Does the script rely on the repository root or activated environment in a way that must be made explicit?
