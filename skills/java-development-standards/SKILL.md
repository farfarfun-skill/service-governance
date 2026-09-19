---
name: java-development-standards
description: Apply consistent, maintainable, and production-safe Java development standards. Use when creating, modifying, refactoring, debugging, or reviewing Java source files, tests, Maven or Gradle projects, libraries, services, and framework-based applications.
---

# Java Development Standards

Produce Java changes that fit the repository, make ownership and failure behavior explicit, and preserve observable behavior unless the task explicitly changes it.

## Establish the Project Contract

1. Read `pom.xml`, Gradle files, wrapper configuration, CI configuration, and nearby source and tests before editing.
2. Follow the configured Java release, build tool, formatter, static analysis, test framework, package structure, and application framework.
3. Treat existing project rules as authoritative unless they are unsafe, broken, or conflict with the requested behavior. Explain any necessary exception.
4. Reuse installed dependencies and local utilities. Add a dependency only when the JDK and existing libraries cannot solve the problem cleanly.
5. Keep the change scoped. Do not reformat, rename, or refactor unrelated code.

## Write Clear Java

- Use standard naming: lowercase packages, `UpperCamelCase` types, `lowerCamelCase` methods and variables, and `UPPER_SNAKE_CASE` constants.
- Let the configured formatter own layout. Keep one public top-level type per file and make the filename match it.
- Prefer the simplest design that meets current requirements. Introduce an interface, hierarchy, or generic abstraction only for a real boundary or multiple concrete uses.
- Keep methods focused and control flow direct. Prefer guard clauses over deep nesting.
- Make fields `private` and `final` when they should not be reassigned. Prefer immutable values and defensive copies at mutable collection or array boundaries.
- Use records for transparent data carriers only when the configured Java version and project conventions support them.
- Program to collection interfaces in public APIs. Return empty collections instead of `null` and avoid exposing mutable internal collections.
- Use streams for clear transformations, not for stateful logic or control flow. Prefer a loop when it is easier to understand.
- Avoid `Optional` fields and parameters. Use `Optional` as a return type only when absence is expected and meaningful; never call `get()` without proving presence.

## Model Ownership and Dependencies

- Prefer constructor injection for required collaborators. Keep dependencies explicit and avoid service locators and hidden global state.
- Separate domain behavior from transport, persistence, and framework wiring when the repository already has those boundaries.
- Keep data-transfer models at system boundaries. Do not duplicate models without a real difference in ownership, lifecycle, or representation.
- Implement `equals` and `hashCode` together and base them on stable identity or value semantics. Add `toString` only when it is safe and useful for diagnostics.
- Do not introduce Lombok, reflection, code generation, or annotation-driven indirection unless the project already relies on it or the benefit clearly exceeds the hidden behavior.

## Handle Failures and Boundaries

- Validate external input at the boundary and express invariants close to the owning type.
- Catch the narrowest useful exception. Catch only to recover, translate, or add actionable context, and preserve the original cause.
- Never swallow exceptions or use exceptions for normal control flow. Follow the project's established checked-versus-unchecked exception policy.
- Use try-with-resources for every `AutoCloseable` resource.
- Use parameterized logging and the existing logging facade. Do not use `System.out` for application logging or log secrets and unnecessary personal data.
- Use prepared statements or the persistence framework's parameter binding. Do not construct queries from untrusted strings.
- Treat shared mutable state as a concurrency boundary. Prefer immutable data and framework-managed executors; do not create unmanaged threads in server code.
- Preserve interrupt status when an `InterruptedException` cannot be propagated.

## Preserve API and Data Behavior

- Preserve public method signatures, exception contracts, serialized fields, database semantics, and HTTP behavior unless the task requires a breaking change.
- Make compatibility changes explicit. Do not keep speculative overloads or adapters without a supported caller.
- Use `java.time` for dates and times. Make timezone and units explicit at external boundaries.
- Use `BigDecimal` for exact decimal business values when binary floating point is unsuitable, and specify rounding deliberately.
- Avoid returning `null` from collection APIs and avoid accepting ambiguous sentinel values.

## Test and Verify

1. Use the existing test framework, assertion library, and test layout.
2. Test observable behavior, boundary cases, and failure paths. Add one focused regression test for a bug fix.
3. Keep tests deterministic; avoid real networks, wall-clock sleeps, and order dependence.
4. Use the checked-in Maven or Gradle wrapper when present. Run the narrowest relevant test first, then configured formatting, static analysis, and broader verification tasks.
5. Report commands run and any checks that could not run.

## Review Java Changes

- Prioritize correctness, security, data loss, concurrency, transaction boundaries, compatibility, resource leaks, and missing tests over stylistic preference.
- Confirm compatibility with the configured Java release, framework, and dependencies.
- Report concrete findings with file and line references, impact, and the smallest viable correction.
- Do not report formatter-owned layout or personal taste as defects.
