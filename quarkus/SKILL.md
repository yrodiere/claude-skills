---
name: quarkus
description: Use this skill to develop the Quarkus framework or applications based on it
---

# Quarkus Development Skill

Expert guidance for Quarkus framework and application development.

## Build Commands

Prefer `mvnd` (Maven Daemon) over `./mvnw` when available — it keeps a
warm JVM across builds and parallelizes modules by default.
Fall back to `./mvnw` if mvnd is not installed.

```bash
mvnd -Dquickly                      # Full build, skip tests/docs/native (parallel by default)
mvnd install -f extensions/<name>/  # Build one extension
mvnd verify -f extensions/<name>/ -Dtest-containers -Dstart-containers  # Run extension tests
mvnd test -Dtest=MyTest -f extensions/<name>/deployment/               # Run single test
```

- Always use `install` (not just `compile`) — downstream modules need
  the jar in the local repo.
- If you change a runtime module, rebuild its deployment module too.
- Always add `-Dtest-containers -Dstart-containers` when running tests.
- **Podman is available** as a Docker-compatible container engine. Tests
  that use Testcontainers work transparently with podman — do not skip
  container-based tests because the `docker` CLI is unavailable.
- **Do not use `-Dno-format`** — formatting and import sorting are
  applied automatically during compilation.
- When using `./mvnw` instead of `mvnd`, add `-T 0.5C` to parallelize
  module builds.
- **Remember the build is very long (10+ minutes)**. It is not a viable strategy to re-run it
  just to look more precisely for errors. If you intend to do that, make sure to save build
  logs to a file when building, then work on that file for various grep operations.

## Project Structure

- `extensions/<name>/runtime/` — Runtime classes, recorders, beans
- `extensions/<name>/deployment/` — `@BuildStep` processors (tests live here)
- `extensions/<name>/deployment-spi/` — Build items shared between extensions
- `extensions/<name>/runtime-dev/` — Dev mode runtime classes

**Deployment depends on runtime, NEVER the reverse.** Runtime code must
not reference deployment classes.

## Build Steps

### Recorders Bridge Deployment and Runtime

A `@Recorder` lives in the runtime module but is invoked from deployment
build steps. It generates bytecode that runs at application startup.

```java
// In deployment module:
@BuildStep
@Record(ExecutionTime.RUNTIME_INIT)
void configure(MyRecorder recorder, ...) {
    recorder.doSomething(buildTimeValue);
}
```

### Build Items

- `SimpleBuildItem` — at most one instance per build
- `MultiBuildItem` — multiple instances collected as `List<>`
- Build items in `deployment-spi/` are shared between extensions
- Build items in `deployment/` are internal to the extension

### Cycle Detection

The build step chain is validated statically. If step A produces item X,
and step B consumes X and produces item Y, and Y feeds back to A's
inputs, a cycle is detected — even if the production is conditional.

Common cycle pattern with `BeanDiscoveryFinishedBuildItem`:
```
BeanDiscoveryFinished → (your step) → AdditionalBeanBuildItem → Arc → BeanDiscoveryFinished
```

Fixes:
- Move `AdditionalBeanBuildItem` production to a step that does not
  depend on `BeanDiscoveryFinishedBuildItem`
- Convert from `AdditionalBeanBuildItem` to `SyntheticBeanBuildItem`
  (feeds into a later Arc phase, after `BeanDiscoveryFinished`)
- Extract the offending production into a separate build step

`SyntheticBeanBuildItem` does NOT cause cycles because it feeds into
`BeanRegistrationPhaseBuildItem`, which is after `BeanDiscoveryFinished`.

### Synthetic Beans

```java
syntheticBeans.produce(SyntheticBeanBuildItem.configure(MyBean.class)
        .scope(Singleton.class)
        .unremovable()
        .setRuntimeInit()
        .addInjectionPoint(ClassType.create(DotName.createSimple(MyDep.class)))
        .createWith(recorder.myBeanSupplier())
        .done());
```

- Use `.setRuntimeInit()` if the bean needs runtime config
- Declare all dependencies as `.addInjectionPoint()` — Arc removes beans
  it considers unused, and programmatic lookups via `Arc.container()` are
  invisible to Arc's unused bean detection
- Use `.unremovable()` for beans looked up programmatically

## Hibernate ORM

### Programmatic Transactions
- **Prefer `QuarkusTransaction`** over `UserTransaction`
  - `QuarkusTransaction.requiringNew().run(() -> { ... })`
  - `QuarkusTransaction.joiningExisting().run(() -> { ... })`

### Query APIs
- **Prefer `session.createSelectionQuery()`** (Hibernate 6+ API)
- Avoid legacy `createQuery()` unless necessary

### Session Management
- `@Inject Session` for regular Hibernate Session
- `@Inject StatelessSession` for stateless operations
- `@Inject Mutiny.SessionFactory` for Hibernate Reactive

## Testing

### Test Annotations

- **`QuarkusExtensionTest`** (with `@RegisterExtension`) — For deployment
  module tests. Creates a synthetic application.
- **`@QuarkusTest`** — Full application startup. For integration tests.
- **`@QuarkusIntegrationTest`** — Tests against built artifact.

### QuarkusExtensionTest Patterns

```java
@RegisterExtension
static final QuarkusExtensionTest config = new QuarkusExtensionTest()
    .withApplicationRoot((jar) -> jar
        .addClasses(MyResource.class, MyService.class))
    .overrideConfigKey("quarkus.some.key", "value");
```

- Use `.assertException(t -> assertThat(t).hasMessageContaining(...))`
  for tests that expect build/startup failure
- Use `.withEmptyApplication()` for tests with no application classes
- Use `.setExcludedDependencies()` to remove extensions from classpath
- Use `.setForcedDependencies()` to add extensions to classpath
- The test class itself is a CDI bean and can have `@Inject` fields, which will be handled like any other bean.

### Test Location

- Extension deployment tests: `extensions/<name>/deployment/src/test/`
- Integration tests: `integration-tests/`

## Coding Style

- 4-space indentation (enforced by formatter)
- **Never manually sort imports** — `impsort-maven-plugin` handles it
- Use **JBoss Logging** (`org.jboss.logging.Logger`)
- No `@author` tags, no wildcard imports
- Use `@ConfigMapping` interfaces for configuration
- Use `String.format(Locale.ROOT, ...)` — `.formatted()` is a forbidden API

## Common Pitfalls

- **Classloading**: runtime code must never reference deployment classes
