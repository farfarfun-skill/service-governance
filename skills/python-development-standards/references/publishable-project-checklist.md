# Publishable Python Project Checklist

Use this checklist when a Python project is intended to be installed, built, distributed, or published. Complete applicable items before handoff; report any item that cannot be verified.

## Confirm the Project Shape

- Identify whether the deliverable is a library, application, command-line tool, plugin, or combination.
- Confirm the distribution name, import package name, supported Python versions, build backend, license, and release source of truth from repository evidence.
- Derive `requires-python` from what the dependencies actually support rather than defaulting to the newest interpreter. Check the `requires-python` metadata of the latest published release of every runtime dependency, and cap this project's target and any declared upper bound at the most restrictive one (for example, if a required dependency's latest release supports only up to Python 3.10, target 3.10 here too, not a newer version). Prefer the lowest floor that still satisfies every dependency and any repository evidence of the minimum actually used.
- Preserve the existing package layout unless changing it is required. Do not impose `src/` layout or flat layout solely by preference.
- Remove placeholder names, descriptions, URLs, authors, examples, and `TODO` markers from publishable metadata and documentation.
- Do not invent ownership, contact, license, compatibility, performance, or support claims. Ask or omit optional metadata when repository evidence is absent.

## Complete README.md in Chinese

Write headings, explanations, setup instructions, and usage guidance in Chinese unless the user explicitly requests another language. Keep package names, identifiers, commands, code, protocol names, and license identifiers in their native form.

Include the following sections when applicable:

1. Project name and one concise description of its actual purpose.
2. Main capabilities and important limitations.
3. Supported Python version and required system services or native libraries.
4. Installation commands for the published package (`pip install`/`uvx`, as appropriate for the audience) and, for local development, `uv sync`.
5. A minimal quick-start example that runs against the current public API.
6. CLI commands, API entry points, configuration, environment variables, or file formats users need.
7. Development commands for tests, linting, formatting, typing, and building, run through `uv run` (for example, `uv run pytest`, `uv run ruff check`, `uv build`), but only when the project configures them.
8. License and repository, issue tracker, or documentation links that are known and valid.

Apply these quality checks:

- Match every example, option, default, path, and output to the implementation.
- Make the first example the shortest successful path; move advanced usage later.
- Explain destructive commands, credentials, network access, and generated files before users encounter them.
- Use fenced code blocks with correct language tags and keep commands directly runnable from the documented directory.
- Avoid empty sections, duplicate metadata, marketing filler, and badges that are not backed by a working service.

## Complete pyproject.toml

- Use `pyproject.toml` as the primary project and tool configuration. Keep legacy packaging files only when a supported workflow still requires them.
- Define `[build-system]` with `hatchling` as the build backend (`requires = ["hatchling"]`, `build-backend = "hatchling.build"`) unless the project already ships a working, different backend.
- Manage dependencies, the virtual environment, and the lock file with `uv` (`uv add`, `uv sync`, `uv lock`, `uv.lock`). Do not add `poetry`, `pipenv`, `pdm`, or a hand-rolled `requirements.txt` workflow to a project that standardizes on `uv`.
- Complete applicable `[project]` fields: `name`, `version` or `dynamic`, `description`, `readme`, `requires-python`, `license`, `authors`, `dependencies`, `optional-dependencies`, `scripts`, `entry-points`, and `urls`.
- Keep the distribution name distinct from the import name when they genuinely differ, and document the import users should write.
- Use one version source. Configure dynamic versioning completely when selected; do not declare `dynamic = ["version"]` without a working provider.
- List only runtime requirements in `dependencies`. Put test, lint, documentation, and build tools under `[dependency-groups]` or `[tool.uv]` dev dependencies.
- Use environment markers and extras for genuinely conditional features. Avoid exact pins in reusable libraries; use tested lower bounds and add upper bounds only for known incompatibilities. Keep application reproducibility in `uv.lock`.
- Define CLI entry points through `[project.scripts]` and point them at a `typer` app's callable (for example, the `Typer` instance or a thin `main()` wrapper around it) that returns an appropriate exit result.
- Configure package discovery for the real layout and include required non-Python package data. Include `py.typed` when the package intentionally publishes inline type information.
- Keep formatter, linter, type checker, test, and coverage configuration consistent with the supported Python floor.
- Ensure the referenced README and license files exist with matching case. Ensure project URLs are valid and use the correct repository.
- Parse the final TOML with the current toolchain; do not rely on visual inspection alone.

## Keep Generated Files Out of Git

Ensure `.gitignore` includes at least:

```gitignore
__pycache__/
*.py[cod]
*$py.class
build/
dist/
*.egg-info/
.venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.coverage*
htmlcov/
logs/
*.log
```

- Preserve additional project-specific ignore rules. Do not ignore source, fixtures, lock files, or required package data by broad pattern.
- Ignore a project's `logs/` directory and loose `*.log` files by default, since they almost always hold runtime output (including from `farlog`). Track a specific log file only when the project demonstrably ships it as required data (for example a fixture consumed by a test), and say so explicitly rather than leaving it tracked by omission.
- Inspect tracked files with `git ls-files` and filter for `__pycache__/`, `.pyc`, `.pyo`, and any tracked `logs/`/`*.log` paths before deleting anything.
- If generated bytecode or log files are tracked or were committed previously, add the ignore rules first, then remove only the exact matched files and directories from Git and the working tree. Commit that removal normally and verify the tracked-file query returns no matches afterward.
- Remove generated build, test, and coverage artifacts only after resolving their exact paths. Never target the repository root or an unresolved variable with a recursive delete.
- Do not rewrite Git history solely to erase bytecode or logs. Treat exposed secrets as a separate security incident that may require history rewriting and credential rotation.

## Run Post-Change Verification

1. Run the configured formatter check, linter, type checker, and full test suite through `uv run` (for example, `uv run ruff check`, `uv run mypy`, `uv run pytest`).
2. Build both source and wheel distributions with `uv build` (backed by `hatchling`).
3. Run the configured metadata validation, such as `uv run twine check dist/*`, when available.
4. Inspect archive contents and exclude caches, tests or docs not intended for distribution, local paths, secrets, and unrelated files.
5. Install the built wheel into a fresh temporary environment (`uv venv` plus `uv pip install`) and smoke-test the documented import, primary API, and CLI entry points.
6. Confirm the installed package exposes its declared version and required package data.
7. Re-run every README quick-start command that can execute safely in the local environment.
8. Check `git status`, `git diff --check`, ignore behavior, and tracked bytecode one final time.
9. Report the exact commands run, their results, and any skipped check with its reason.

When the project already uses `funbuild`, its pipeline (`funbuild upgrade`, `funbuild build`, `funbuild install`, `funbuild tag`) can perform steps 2, 3, and 5 in one pass; still run steps 1, 4, 6, 7, and 8 yourself and confirm before letting it push or tag.

Do not upload a distribution, publish a release, create a tag, or push changes unless the user explicitly requests that external action.
