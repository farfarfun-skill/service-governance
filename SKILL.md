---
name: service-release-governance
description: Govern how services are built, published, installed, and started across development and production environments. Use when Codex prepares, reviews, troubleshoots, or documents a service release; configures package publishing or installation for PyPI, npm, Maven, or another artifact repository; chooses between private and public registries; uses nltbuild; writes production startup commands; or audits whether production is isolated from local source changes.
---

# Service Release Governance

Keep development fast and production reproducible. Development may run checked-out source directly. Production must run an immutable formal package that was published to a repository and then installed from that repository.

## Select The Mode

| Mode | Allowed source | Required flow |
| --- | --- | --- |
| Development | Current working tree, direct source command, or editable install | Change source -> run locally |
| Production | Versioned formal package installed from a repository | Build -> publish -> install exact version -> start installed artifact |

Never reuse a development checkout, editable install, local package file, workspace link, or build directory as the production runtime. Do not let a production process import modules from the source workspace through its working directory or `PYTHONPATH`, `NODE_PATH`, classpath, or an equivalent override.

## Prepare A Formal Release

Inspect the repository before choosing commands. Reuse its manifest, lockfile, build wrapper, publishing configuration, and existing release automation. Do not add a second release path when one already works.

1. Identify every independently deployed service, its package ecosystem, package name, formal version, and intended source commit or tag.
2. Reject development versions such as Python dev/local versions, npm prereleases, Maven `SNAPSHOT`, or mutable aliases unless the user explicitly requests a prerelease environment.
3. Select the repository already configured for that ecosystem. Prefer an organization-controlled private repository. Use a public repository only when public distribution is required or no approved private repository exists.
4. Build from the intended committed source. Use `nltbuild build` for a Python project already configured for `nltbuild`; it performs the build and upload in one command. Otherwise use the project's existing ecosystem-native build and publish command.
5. Publish a new immutable version. Never overwrite or reuse a released formal version.
6. Create a clean production-like environment, install the exact version from the selected repository, and confirm resolution did not fall back to a local path or unintended repository.
7. Start the service through the installed package's entry point or retrieved artifact. Run the smallest smoke check that proves the installed version starts.

Do not invent repository URLs, credentials, signing keys, or package coordinates. Stop and request the missing value when repository configuration does not establish them. Keep credentials in the ecosystem's supported secret store or CI secret mechanism; never write them into source, commands shown in logs, or release records.

## Apply The Ecosystem Contract

| Ecosystem | Typical repository | Formal release path |
| --- | --- | --- |
| Python | Private Python index or PyPI | Build/upload with configured `nltbuild build` or existing backend; install an exact version with `pip`/project installer; start the installed console entry point or module |
| npm | Private npm registry or npmjs | Pack/build and `npm publish` with project configuration; install `name@exact-version`; start the installed package entry point |
| Maven/Gradle | Private Maven repository or Maven Central | Use the project wrapper to deploy a non-`SNAPSHOT` artifact; resolve exact coordinates from the repository; start the retrieved JAR/application distribution |
| Other | Ecosystem-native artifact repository | Build and publish an immutable version; install or pull that exact version into a clean runtime; start only the retrieved artifact |

Treat repository-specific commands as project configuration, not universal constants. Private-first means the private repository is the selected source for the service package; public fallback must be explicit and must not silently replace an internal package with the same name.

## Gate Production

Return `block` when any of these is true:

- Production starts directly from a source checkout or consumes an editable, linked, or local-file installation.
- The formal package was not successfully published to the selected repository.
- The production install is unpinned, resolves a different version, or cannot prove its repository source.
- The release version is mutable, already exists and would be overwritten, or is a development/prerelease version without explicit approval.
- The installed artifact cannot start or fails its required smoke check.
- Required credentials would be exposed or repository identity is unknown.

Return `revise` for missing reproducibility evidence that does not yet prove a violation. Return `allow` only after the formal package is published, installed by exact version from the intended repository in a clean environment, and started successfully without workspace source access.

Report the mode, ecosystem, package name and version, source commit/tag, repository, build/publish command, clean install command, installed artifact identity, start command, smoke result, and final decision. Redact credentials.
