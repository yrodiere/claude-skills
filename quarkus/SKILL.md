---
name: quarkus
description: Use this skill to develop the Quarkus framework or applications based on it
---

# Quarkus Development Skill

Expert guidance for Quarkus framework and application development.

## Hibernate ORM Best Practices

### Programmatic Transactions
- **Prefer `QuarkusTransaction`** for programmatic transaction management
  - Use `QuarkusTransaction.requiringNew().run(() -> { ... })` for new transactions
  - Use `QuarkusTransaction.joiningExisting().run(() -> { ... })` to join existing transactions
  - Don't use `begin()`/`commit()`/`rollback()` - use the functional API instead
  - Don't use `UserTransaction` directly unless specifically required

### Query APIs
- **Prefer `session.createSelectionQuery()`** for read queries
  - Modern Hibernate 6+ API
  - Type-safe and more expressive
  - Example: `session.createSelectionQuery("SELECT name FROM Person", String.class)`
- Avoid legacy `createQuery()` unless necessary for compatibility

### Session Management
- Use `@Inject Session` for regular Hibernate Session
- Use `@Inject StatelessSession` for stateless operations

## General Quarkus Patterns

### Dependency Injection
- Use `@Inject` for CDI bean injection
- Use qualifiers like `@PersistenceUnit` for multiple persistence units
- Prefer field injection when possible in tests

### Testing
- Use `QuarkusExtensionTest` for extension tests (deployment module)
- Use `QuarkusTest` for integration tests
- Use `QuarkusUnitTest` for older versions of Quarkus only where QuarkusExtensionTest is not available
- For Hibernate ORM/Reactive tests, prefer simple top-level entity types like `io.quarkus.hibernate.orm.MyEntity` instead of creating custom entities

## Quarkus Build Specifics

Quarkus is a very large project. Building/testing the whole project is not time-efficient.

### Full Build

When you need to build the whole project (or a large portion of it),
use parallel threads with `-Dquickly`:

```bash
./mvnw -T 0.5C -Dquickly
```

`-T 0.5C` uses half a thread per CPU core, providing significant speedup
for the large multi-module build.

### Running Tests
- **Always `cd` to the module first** before running Maven commands, OR
- Use `-f <path-to-module>` to specify the module from the root
- **For extensions requiring containers** (Hibernate Reactive, Reactive Panache, etc.), add `-Dstart-containers -Dtest-containers`
  - These modules use PostgreSQL and need containers started
  - Hibernate ORM uses H2 and doesn't require containers
- Examples:
  - `cd extensions/hibernate-orm/deployment && mvn test -Dtest=MyTest`
  - `cd extensions/hibernate-reactive/deployment && mvn test -Dtest=MyTest -Dstart-containers -Dtest-containers`
  - `mvn test -Dtest=MyTest -f extensions/hibernate-orm/deployment`

### Module Structure
- Extensions have separate runtime and deployment modules
- Tests live in the deployment module: `extensions/<extension-name>/deployment`
- **If you modify runtime code**, you must compile and install it before running deployment tests:
  - `cd extensions/<extension-name>/runtime && mvn clean install -DskipTests`
  - Then run your deployment tests

## Common Pitfalls to Avoid

- Don't use `UserTransaction` when `QuarkusTransaction` is available
- Don't use legacy query APIs when modern ones exist
- Don't forget that Quarkus uses build-time configuration for many features
- Don't run Maven from the wrong directory - always cd to the module or use -f
