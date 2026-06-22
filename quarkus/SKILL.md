---
name: quarkus
description: Use this skill to develop the Quarkus framework or applications based on it
---

# Quarkus Development Skill

Expert guidance for Quarkus framework and application development.

## Build Commands

```bash
./mvnw -Dquickly                    # Full build, skip tests/docs/native
./mvnw install -f extensions/<name>/ # Build one extension
./mvnw verify -f extensions/<name>/ -Dtest-containers -Dstart-containers  # Run extension tests
./mvnw test -Dtest=MyTest -f extensions/<name>/deployment/               # Run single test
```

- Always use `install` (not just `compile`) — downstream modules need
  the jar in the local repo.
- If you change a runtime module, rebuild its deployment module too.
- Always add `-Dtest-containers -Dstart-containers` when running tests.
- **Do not use `-Dno-format`** — formatting and import sorting are
  applied automatically during compilation.

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
- The test class itself gets indexed — `@Inject` fields in the test
  class ARE visible to injection point scanning

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
- **Config keySet()**: `@WithUnnamedKey` causes the default key to always
  appear in map `keySet()` — use `isAnyPropertySet()` as a workaround
- **BeanResolver at BeanDiscoveryFinished**: `beansByType` is not
  initialized yet — `getBeansByRawType()` will NPE. Use `getBeans()`
  collection with manual filtering instead.
- **Test-scoped JDBC drivers**: with multiple JDBC drivers on classpath,
  a single test-scoped driver is automatically picked as default.
  To test ambiguity, exclude the test-scoped one and force-add two
  non-test-scoped drivers.
