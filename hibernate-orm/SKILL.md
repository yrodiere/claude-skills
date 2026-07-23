---
name: hibernate-orm
description: This skill should be used when the user works on Hibernate ORM code, asks to "review a Hibernate PR", "fix a Hibernate bug", "write a Hibernate test", "build Hibernate", or needs guidance on Hibernate project conventions, structure, or workflows. Complements the java-architect skill with Hibernate-specific knowledge.
---

## Workspaces and Git Operations

- The main workspace is `/Volumes/CaseSensitive/Projects/hibernate-orm`, tracking the `main` branch (most recent version).
- Additional workspaces exist for each maintained branch, named `/Volumes/CaseSensitive/Projects/hibernate-orm_<version>` (e.g. `hibernate-orm_7.2`). These are used for version-specific debugging, backporting, etc.
- When backporting or moving code between branches, always use `git cherry-pick` rather than manually copying code, unless cherry-pick conflicts make it impractical.
- A `/Volumes/CaseSensitive/Projects/hibernate-backport.sh` script is available for interactively creating backport PRs (mainly for bugfixes).
- Always work on dedicated feature branches, never commit directly to `main` or maintenance branches. When the target branch is not specified, default to branching from `main`.
- All workspaces should have two remotes configured:
  - `upstream` -> `hibernate/hibernate-orm` (upstream repository)
  - `origin` -> `mbellade/hibernate-orm` (personal fork)

## PR Reviews

1. Fetch the PR details with `gh pr view <number>`
2. Fetch the PR diff with `gh pr diff <number>`; if asked to cherry-pick or checkout the changes, use `gh pr checkout <number>` to create a local branch with the changes
3. Find the related Jira issue key (HHH-XXXXX) from the PR title/branch/commits
4. Fetch Jira issue for context using the Atlassian MCP (if any)
5. Read the test changes and verify they reproduce the bug WITHOUT the fix
6. Analyze the fix for correctness, edge cases, and minimal scope
7. Check if previous review comments were addressed as asked (if any)
8. Present structured findings: Summary, Jira Context, Test Coverage, Fix Analysis, Concerns

Note: these steps can be done as parallel sub-agents (Agent tool); do not skip the Jira context step.

## Jira

- The project Jira is at https://hibernate.atlassian.net (project key: `HHH`).
- When working on a Jira issue, always read the full details of the report (including all available fields and comments) to have as much context as possible. Use the Atlassian MCP to query for issue information.
- Commit messages must always start with the Jira key they reference (e.g. `HHH-12345 Fix something`).
- Naming branches after the Jira key (e.g. `HHH-12345`) is a good default — it makes them easy to find.

## Project structure

- For general information about the repository, read the project's `README.md` file in the workspace root.
- The `design/` directory in the repository root contains internal design documents explaining the inner workings of the ORM codebase and the rationale behind them.

### Key modules

| Module | Purpose |
|--------|---------|
| `hibernate-core` | The main ORM engine — mapping, querying (HQL/SQM/SQL AST), session, boot, dialects |
| `hibernate-testing` | JUnit extensions (`@DomainModel`, `@SessionFactory`, `@Jpa`), custom assertions, test utilities |
| `hibernate-envers` | Audit/versioning |
| `hibernate-community-dialects` | Community-maintained database dialects |
| `hibernate-spatial` | Geospatial types support |
| `hibernate-vector` | Vector/embedding types support |
| `hibernate-jcache` / `hibernate-micrometer` / `hibernate-jfr` | Caching, metrics, flight recorder integrations |
| `hibernate-agroal` / `hibernate-c3p0` / `hibernate-hikaricp` | Connection pool integrations |
| `hibernate-graalvm` | GraalVM native image support |
| `hibernate-scan-jandex` | Classpath scanning via Jandex |
| `documentation` | Source files for the official Hibernate documentation; useful as a reference to understand key features and behaviors |
| `local-build-plugins` | Custom Gradle build plugins |

### Testing infrastructure

Testing infrastructure lives in `hibernate-testing` under `org.hibernate.testing.orm.junit` (JUnit extensions) and `org.hibernate.testing.orm.domain` (reusable domain models).

## Build system

- Compilation is slow (> 1 minute), avoid verifying code through compilation during intermediate editing steps unless explicitly requested. Rely on code reading and reasoning instead.
- Do not use `--no-build-cache` when building or running tests unless the user explicitly asks for it or there is a specific reason to suspect cache corruption. The Gradle build cache is automatically invalidated when relevant inputs change, so skipping it provides no benefit and significantly slows down an already slow build.
- The project uses Gradle. Build configuration is split between per-module `*.gradle` files and shared convention plugins in `local-build-plugins`.
- Convention plugins (e.g. `local.java-module`, `local.databases`, `local.publishing`) are applied via `plugins { id "local.<name>" }` in module build files.
- The default test database is H2 (configured in `gradle.properties` via `db=h2`). Other databases can be selected with `-Pdb=<name>` (available profiles are defined in `local-build-plugins/src/main/groovy/local.databases.gradle`).
- The `docker_db.sh` script in the repository root can be used to spawn Docker containers for all supported databases (e.g. `./docker_db.sh postgresql`). Use the corresponding profile name from `local.databases.gradle` as the `-Pdb` value to run tests against the spawned container (e.g. `./gradlew hibernate-core:test -Pdb=pgsql_ci`).
- Dependencies are managed via version catalogs defined in `settings.gradle` (`libs`, `jdks`, etc.).

## API vs SPI vs Internal

### Categories

- **API** — contracts exposed to applications. Packages without `internal`, `impl`, or `spi` in the name. Stable across all releases within a **major** version. Only additions allowed; no alterations or removals.
- **SPI** — integration points for frameworks and libraries. Packages with `spi` in the name. Stable across all releases within a **minor** version. Best effort across minor versions, but not guaranteed.
- **Internal** — code for internal use only. Packages with `internal` or `impl` in the name. No compatibility guarantees; can change at any time.

### Key annotations

- `@Internal` — marks code that is internal but leaks through API/SPI due to technical constraints or historical reasons. No compatibility guarantees.
- `@Incubating` — code still under development that may become stable API/SPI in the future. No compatibility guarantees; can change or be removed at any time, including in bugfix releases.

### Guidelines for contributors

- Never break API contracts within a major version. New API methods/classes are fine; changing or removing existing ones is not.
- SPI changes should be confined to minor version bumps. Avoid breaking SPI in micro (bugfix) releases.
- When adding new API or SPI, consider whether it should be marked `@Incubating` first.
- Internal code can be freely refactored, but be aware that `@Internal`-annotated code in API/SPI packages may be used by external consumers despite the annotation.
- Versioning follows `major.minor.micro.qualifier` (e.g. `6.6.1.Final`). Major = potential API breaks, minor = additions only, micro = bugfixes only.

## Code style

The authoritative formatting reference is the IntelliJ code style config at [hibernate_orm_codestyle.xml](hibernate_orm_codestyle.xml). Read it when generating or editing code to match the project's formatting conventions (wrapping, spacing, indentation, brace placement, etc.).

The project uses [Spotless](https://github.com/diffplug/spotless) to enforce a subset of formatting rules (e.g. license headers, import ordering, trailing whitespace). Spotless does not cover all style conventions, so always follow the full code style guidelines when writing code. Before pushing a branch or opening a PR, run `./gradlew spotlessApply` on the affected modules (e.g. `./gradlew hibernate-core:spotlessApply`) to auto-fix any violations. This should be one of the last steps in a work cycle, after the final review of the changes and before committing/pushing.

Key rules not expressed in the XML:

- All source files must include the license header:
  ```
  /*
   * SPDX-License-Identifier: Apache-2.0
   * Copyright Red Hat Inc. and Hibernate Authors
   */
  ```
- Always use `var` where appropriate and where it doesn't reduce readability.
- Never use wildcard imports — always use single-class imports.
- Never use non-ASCII characters in code, comments, or Javadoc. Use plain ASCII alternatives: a single hyphen `-` instead of em/en dashes, straight quotes `"` instead of smart quotes, etc.
- When method call parameters or declarations need to be wrapped, use chop down style (each parameter on its own line, newline after opening and before closing parenthesis).

## Testing

- All changes — improvements, new features, and bug fixes — must be accompanied by a test case that validates correct behavior.
- Before creating a new test, verify if an existing similar test case exists and expand on that.
- Use `@DomainModel` + `@SessionFactory` (+ optionally `@ServiceRegistry` when needed) or `@Jpa` JUnit extensions for tests.
- Try to create the necessary test data in a `@BeforeAll` to separate it from the test logic itself. A `@BeforeEach` is also acceptable when needing to restore initial state for each test method. Exception: when the test itself is about creating the data.
- All tests should clean up after themselves, deleting data is easily done with an `@AfterAll` method that invokes `SessionFactoryScope.getSessionFactory().getSchemaManager().truncateMappedObjects()`.
- All tests should define their mapping model in static inner classes.
- All test entities should have explicit names (i.e. `@Entity(name="...")`).
- Test entities should never use identifiers with an explicit `IDENTITY` generation strategy, unless that's specifically what the test is verifying. Even then, such tests should be restricted to dialects with identity column support.
- All tests should refer to their corresponding Jira issue with the `@Jira("https://hibernate.atlassian.net/browse/HHH-12345")` annotation. Existing tests that use alternative annotations with only the Jira key do not need to be modified.
- Tests which verify dialect-specific features should not use `instanceof` / `getClass()` checks, but instead rely on the `@RequiresDialect` or `@RequiresDialectFeature` annotations.
- During development, prefer running test methods one at a time. The `@SessionFactory` extension destroys and rebuilds the session factory when a test method fails, which means `@BeforeAll` data is lost and subsequent methods will produce invalid results. When JetBrains MCP tools are available, prefer running individual test methods through the IDE (discover run points in the test file, then execute them). This is the preferred approach because the output is directly visible in the user's IDE UI. Alternatively, use parallel sub-agents (Agent tool) to run multiple test methods concurrently via Gradle CLI. There is no problem with concurrent execution since each test gets its own session factory. Running the whole class is fine (and faster) when expecting all tests to pass (e.g. after development, or during simple refactorings).
- When using parallel sub-agents to run tests via Gradle CLI, first compile the test sources manually once (e.g. `./gradlew <module>:testClasses`), then launch sub-agents that only execute tests (not compile). This avoids potential issues with multiple Gradle processes compiling in parallel. This is not needed when running tests through JetBrains MCP, which handles compilation automatically.
- Always ask for confirmation before running tests, unless explicitly asked to do so.

## Documentation

- New features should be reported in the `whats-new.adoc` file in the repository root, unless they're trivial or not of interest to the end user.
- Changes in behavior (API changes, switching default configurations, or simply different behavior) should always be reported in the `migration-guide.adoc` file in the repository root.
- When changing or adding to already-documented features, the corresponding documentation in the `documentation` module should be updated as well. Big enough new features may also warrant adding new documentation.
