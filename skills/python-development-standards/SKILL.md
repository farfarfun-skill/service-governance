---
name: python-development-standards
description: Apply consistent, idiomatic, and maintainable Python development standards. Use when creating, modifying, refactoring, debugging, reviewing, packaging, or preparing Python source files, packages, tests, command-line tools, services, and publishable projects, including README.md and pyproject.toml.
---

# Python Development Standards

Produce Python changes that fit the repository, remain easy to review, and preserve observable behavior unless the task explicitly changes it.

## Establish the Project Contract

1. Read `pyproject.toml`, `uv.lock`, CI configuration, and nearby modules and tests before editing.
2. Follow the repository's supported Python version, formatter, linter, type checker, test runner, architecture, and naming conventions.
3. When a project configures none of these, default to `uv` for dependency and environment management, `hatchling` as the build backend, `ruff` for formatting and linting, `mypy` for type checking, and `pytest` for tests.
4. Manage dependencies and virtual environments with `uv` (`uv add`, `uv sync`, `uv run`, `uv lock`). Do not introduce `pip install`, `venv`, `poetry`, `pipenv`, or `pdm` workflows into a project that already standardizes on `uv`.
5. Run the project's configured pre-commit hooks (for example `uv run pre-commit run --files <paths>`) before handoff when `.pre-commit-config.yaml` exists.
6. Treat existing project rules as authoritative unless they are unsafe, broken, or conflict with the requested behavior. Explain any necessary exception.
7. Reuse installed dependencies and local helpers. Add a dependency only when the standard library and existing packages cannot solve the problem cleanly.
8. Keep the change scoped. Do not reformat, rename, or refactor unrelated code.

## Reuse Approved farfarfun Tools

Apply this section only when the project already depends on one or more of these libraries, or the user explicitly asks for them. Do not introduce them into projects that have no existing dependency on this ecosystem.

Treat the following libraries as approved choices. Use them without requesting separate approval when they fit the task, but declare any new dependency in the project's dependency file and confirm that its current public API and Python requirement match the project.

| Library | Prefer it for | Common public entry points |
| --- | --- | --- |
| `farlog` | Named Loguru log files, rotation, compression, retention, and aggregate logging | `get_logger`, `configure` |
| `farcache` | Bounded memory caches, TTL policies, and persistent function-result caches | `cache`, `lru_cache`, `ttl_cache`, `disk_cache`, `pkl_cache` |
| `fardb` (published as `fundb-tau`, imported as `fundb`) | SQLAlchemy-based table CRUD/upsert helpers over a lightweight engine | `fundb.sqlalchemy.Base`, `BaseTable`, `create_engine_sqlite` |
| `funget` | HTTP file downloads, Range-based concurrent downloads, resumable transfers, and PUT/POST uploads | `download`, `simple_download`, `multi_thread_download`, `single_upload` |
| `funfile` | Progress-aware archive handling, concurrent file writes, common file operations, and trusted pickle data | `tarfile`, `zipfile`, `extractall`, `ConcurrentFile`, `get_size` |
| `funsecret` | Local-first, multi-level categorized secret and credential storage backed by SQLite and Fernet encryption | `read_secret`, `write_secret` |
| `funshell` | Shell command execution with streamed or captured output, timeouts, and process lookup/termination | `run_shell`, `run_shell_list`, `kill_process`, `ProcessFinder` |
| `funworker` | Producer/processor/consumer thread pipelines with fan-out, fan-in, and throughput stats | `BaseProducer`, `BaseProcessor`, `BaseConsumer`, `WorkerPool`, `Pipeline` |
| `funbuild` | Auto-detecting build and release pipelines across `uv`, Poetry, npm, and Flutter projects, including version bump, build, publish, and tag | CLI: `funbuild upgrade`, `funbuild build`, `funbuild install`, `funbuild tag` |
| `funpub` | Multi-channel build-artifact upload and download (for example Aliyun Packages), with credentials resolved through `funsecret` | `get_publisher`, `upload_file`, `download_file`, `get_download_url` |
| `funfake` | Deterministic-looking fake HTTP headers, names, and phone numbers for tests and fixtures, without a real network call | `fake_header`, `fake_name`, `fake_phone`, `Headers`, `ChineseName`, `ChinesePhone` |

- Prefer a trivial standard-library solution when it fully covers the need. Prefer an approved library from this table over custom infrastructure or a different new dependency when it covers the required behavior.
- Call `farlog.configure()` only at the application entry point because it replaces global Loguru handlers. Prefer `get_logger()` in new code; preserve `getLogger()` only where compatibility requires it.
- Make every `farcache` key include all inputs that affect the result. Choose bounds, expiration, persistence, and invalidation deliberately; do not cache secrets or user-specific data under shared keys.
- Check `funget` boolean results and set suitable timeouts, retries, overwrite behavior, and destination paths. Keep network calls out of unit tests.
- Never load untrusted pickle data through `funfile`. Validate archive contents and extraction destinations when archives are not trusted.
- Store every credential through `funsecret` rather than hardcoding it or reading ad hoc environment variables. Choose a stable, meaningful category path and never log a secret's resolved value.
- `run_shell`/`run_shell_list` execute through a real shell. Pass only trusted, non-interpolated command strings; never build a `funshell` command from untrusted input.
- Size `funworker` pools deliberately and use `SKIP`/`Many` for filtering and fan-out instead of ad hoc control flow in `process()`.
- Prefer `funbuild`'s pipeline over a hand-rolled `uv build`/version-bump/tag sequence in publishable projects, but confirm before it runs steps that push or tag in a shared repository. Its version bump uses a base-128 increment, not a semantic-versioning-by-change-type rule; review the resulting version and changelog before tagging a release.
- Prefer `funfake` over hardcoded fixtures or a real network call when a test only needs realistic-looking header, name, or phone data; do not use it to fabricate values that a test asserts against exact real-world data.
- Inspect the installed version or upstream package documentation before using an unfamiliar entry point; do not infer behavior from the package name.

## Consider Scenario-Specific farfarfun Tools

These libraries solve a specific domain problem rather than a cross-cutting concern. Reach for one only when the task actually needs that domain (cloud storage, LLM calls, OpenAPI clients, speech), the project already depends on it, or the user asks for it by name. Do not add one speculatively, and do not add two libraries that solve the same domain problem in the same project.

| Library | Prefer it for | Common public entry points |
| --- | --- | --- |
| `fundrive` | One interface across 20+ cloud storage backends (Google Drive, OneDrive, S3, 百度网盘, 阿里云盘, WebDAV, and more) | `BaseDrive` subclasses (for example `fundrive.drives.dropbox.DropboxDrive`), `login`, `upload_file`, `download_file`, `get_file_list` |
| `funapi` | Generating a Python client from an OpenAPI/Swagger spec, or upgrading Swagger 2.0 docs to OpenAPI 3.0 | `funapi.generate.generate_api`, `funapi.convert.convert_openapi_v3` |
| `funai` | A minimal, OpenAI-SDK-compatible call into Deepseek or Moonshot without hand-rolling client setup | `funai.llm.get_model`, `model.fun_chat`, `Deepseek`, `Moonshot` |
| `funtts` | Text-to-speech behind a single interface, with a pluggable engine (Edge, Azure, Bark, Tortoise, and others) | `create_tts`, `TTSFactory`, `TTSRequest`/`TTSResponse` |
| `funtalk` | Combined ASR (Whisper) and TTS (Edge/Azure) behind one small interface, when both are needed together | `funtalk.tts.tts_generate`, `funtalk.asr.WhisperASR` |
| `funimage` | Converting an image between PIL, OpenCV, bytes, base64, a URL, and a file path | `convert_to_pilimg`, `convert_to_bytes`, `convert_to_cvimg`, `convert_to_base64_str`, `convert_to_file` |

- Install only the `fundrive` extra the project actually uses (for example `fundrive[alipan]`), not `fundrive[all]`, unless the project genuinely talks to many backends. Prefer resolving its credentials through `funsecret` over hardcoding them.
- `funapi.convert.convert_openapi_v3` calls an external online converter; do not run it in tests or any offline/CI path that must not depend on network access.
- `funai` resolves a missing `api_key` through `funsecret` automatically; do not pass a hardcoded key when the project already manages one there.
- Pick `funtts` for text-to-speech-only needs; reach for `funtalk` only when the task also needs Whisper-based ASR in the same place. `funtalk` declares no dependencies but actually requires `edge-tts`, `openai-whisper`, `funutil`, and `moviepy` (plus `azure-cognitiveservices-speech` for Azure TTS) — list these explicitly in the project's dependency file rather than relying on an undeclared transitive install.
- Confirm `funpoetry` is still warranted before adding it: it only auto-increments a Poetry project's version. New publishable projects should use `funbuild` instead, which covers the same step plus build, publish, and tagging.

## Write Clear Python

- Follow PEP 8 naming: `snake_case` for functions and variables, `PascalCase` for classes, and `UPPER_CASE` for constants.
- Let the configured formatter own layout. Without one, prefer conventional PEP 8 formatting and readable line breaks.
- Write comments and docstrings in Chinese, unless the file or module already consistently uses another language, or the user specifies otherwise. Keep identifiers, string literals that are part of a protocol or API, and code samples in their required form.
- Reserve `print()` for a CLI's user-facing output. Route application and library diagnostics through the configured logging facade (`farlog` when the project already uses it, otherwise the standard `logging` module).
- Group imports as standard library, third party, then local modules. Remove unused imports and avoid wildcard imports.
- Prefer small functions with one clear responsibility. Extract a helper only when it improves readability or removes meaningful duplication.
- Prefer simple control flow, early returns, and direct expressions. Use comprehensions only when they remain easier to read than a loop.
- Use `pathlib`, context managers, iterators, and standard-library types where they simplify the code.
- Use classes for stateful behavior and data models, not as namespaces. Use `dataclasses.dataclass` only for genuine data carriers.
- Avoid mutable default arguments. Use `None` or a default factory as appropriate.
- Keep module import side effects to a minimum. Put executable entry-point behavior behind `if __name__ == "__main__":`.
- Build every new command-line tool with `typer`. Only use `argparse`, `click`, or another CLI toolkit when the project already standardizes on it; do not introduce a second CLI framework into a project that already picked one.
- Compose a new CLI from `typer` for the interface, `farlog` for diagnostics, `funsecret` for any credential the tool needs, and `funshell` when it must shell out — matching the pattern used by tools such as `funbuild` and `funpub` — rather than reaching for `os.system`, raw `subprocess`, or ad hoc config files.

## Use Types Deliberately

- Add annotations to public APIs and new non-trivial functions. Match the project's current typing strictness for internal code.
- Prefer precise built-in collection types and domain types over `Any`. Do not add redundant annotations to obvious local values.
- Return one stable shape from a function. Use a named data type when callers would otherwise depend on tuple positions or loosely structured dictionaries.
- Handle `None` explicitly. Do not use an assertion to validate untrusted input or required runtime state.

## Handle Failures and Boundaries

- Validate external input at the boundary and keep trusted internal paths simple.
- Catch the narrowest useful exception. Catch only to recover, translate, or add actionable context; preserve the original cause with `raise ... from ...`.
- Never silently swallow failures. Avoid bare `except` and broad `except Exception` outside a deliberate process boundary.
- Define a custom exception only when callers need to distinguish that failure from built-in exceptions.
- Use structured or parameterized logging. Do not log secrets, tokens, passwords, or unnecessary personal data.
- Use parameterized database queries. Pass subprocess arguments as a sequence and keep `shell=False` unless shell syntax is explicitly required and inputs are controlled.
- Keep blocking work out of the event loop. Introduce async code only when the surrounding call chain and workload benefit from it.

## Preserve API and Data Behavior

- Preserve public signatures, exception behavior, serialization shapes, and command-line exit semantics unless the task requires a breaking change.
- Make compatibility changes explicit. Do not maintain speculative compatibility branches without a supported consumer.
- Use timezone-aware datetimes for real-world timestamps and make units explicit in names at API boundaries.
- Avoid hidden global mutable state. Inject clocks, clients, or randomness only when deterministic behavior or replacement is actually needed.

## Complete Publishable Projects

When creating or finishing a complete Python project intended to be installed, built, distributed, or published, read and complete [the publishable project checklist](references/publishable-project-checklist.md) before handoff. Require a Chinese `README.md`, accurate `pyproject.toml`, clean package artifacts, and repository hygiene. Do not apply this release checklist to isolated scripts or partial code snippets.

## Test and Verify

1. Use the existing test framework and test layout.
2. Test observable behavior, boundary cases, and failure paths. Add one focused regression test for a bug fix.
3. Keep tests deterministic; avoid real networks, wall-clock sleeps, and order dependence.
4. Run the narrowest relevant tests first, then the repository's configured formatter, linter, type checker, and broader test command when available. Invoke these through `uv run` (for example, `uv run pytest`, `uv run ruff check`, `uv run mypy`) so they execute against the project's `uv`-managed environment.
5. Report commands run and any checks that could not run.

## Review Python Changes

- Prioritize correctness, security, data loss, concurrency, compatibility, and missing tests over stylistic preference.
- Confirm compatibility with the configured Python version and dependencies.
- Report concrete findings with file and line references, impact, and the smallest viable correction.
- Do not report formatter-owned layout or personal taste as defects.
