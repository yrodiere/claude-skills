---
name: java-dev
description: Use this skill to develop or debug Java applications
---

## Safety
Our code is often deployed in mission critical scenarios. Make sure to take this in consideration.
We need to be very careful to not cause resource leaks (FDs, memory), deadlocks, classloader leaks, or silent data corruption.
When the AI detects a violation of these safety rules in existing code, it should output a 'CRITICAL SAFETY WARNING' block in the chat with actionable suggestions and the technical context of the risk (e.g., potential for a Deadlock or memory leak).
Our code should also emit appropriate warnings when it's possible to detect violation of assumptions, and such messages should include actionable suggestions.

## Reproducibility
We highly value reproducibility; if some code path is not fully deterministic try to shape it so that it's more deterministic,
unless specifically required. For example when working on bootstrap code or build code, prefer a sorted structure over data
structures using natural (pseudo-random) hashing - however be mindful of efficiency requirements: when creating code which is expected to be executed very often at runtime, performance considerations need to take preference over reproducibility.
Try to understand from the context if what kind of code we're working on - for example if we're working on a Maven plugin, it will be build code. When in doubt about what to prioritize, ask the user for clarifications.
Also consider security requirements, for example in some context data structures might need salting - good security practices
and correctness take priority over anything else. When opting for a particular solution because of security or correctness requirements, make sure to document the reasons.

## Build Tools
When `mvnd` (Maven Daemon) is available, prefer it over `mvn` or `./mvnw` for all Maven
builds. mvnd keeps a long-lived daemon JVM, reuses JIT-compiled code and cached classloaders
across builds, and parallelizes module builds by default — no `-T` flag needed.
If mvnd is not available, fall back to `./mvnw` or `mvn`.

## Code duplication
When needing to generate new helpers and utilities, always check first for existing code which could be suitably reused.

## Concurrency
Much of our code will be run in complex concurrent servers, but we aim for top efficiency and therefore most of our state will be confined into a single thread.
Prioritize thread-local storage or event-loop patterns over shared-state concurrency. If code is intended for a single-threaded 'Hot Loop', explicitly document that it is not thread-safe to prevent future developers from misusing it.
Always try to understand if the code we're working on is going to be single threaded or not, and what are the critical sections. Try to minimize critical sections; when they are necessary make sure they are treated correctly, suggest possible simplifications and document critical assumptions and tradeoffs clearly.

## Code clarity
Always prefer marking parameters and variables as final unless specifically needing them mutable.

## Testing
Strive to include a fully automated integration test; if this is very unpractical consult with the user about this requirement.
Include unit tests when they are adding value, especially for classes with complex logic and data transformations. Skip unit tests if they are creating excessive coupling while not adding test coverage which isn't already covered by integration tests.

## Performance
Our code is going to run on billions of devices, always be very careful to write efficient, simple and lean code, it needs to perform well and be mindful of allocations as well.
Avoid excessive use of java.util.stream and functional overhead in performance-critical paths. Prefer primitive collections (or arrays) to avoid wrapper types and unnecessary garbage collection pressure.
Avoid the use of lambdas, prefer methodhandles instead; the exception is the creation of examples and when testing APIs which would benefit from lambdas for better end user's readability.
Prefer methodhandles over reflection.

## Documentation
Generate Javadoc and comments on non-trivial methods only and keep it very brief. Avoid documenting obvious things.
When creating new classes, take care in choosing fitting names - when in doubt, propose some options to the user before proceeding.
Focus on the why and document tradeoffs.

## Minimize changes
To minimize changes of conflicts, ease of maintenance and review, you should always try to keep the number of modified lines to a minimum. So for example while we prefer using a "final" qualifier on arguments for new code, do not alter existing method signatures unless there's semantic need. Same rule for whitespaces and formatting: do not change any existing line unless there is need for it,
try to honour the existing formatting conventions. Do not introduce new @author tags unless specifically suggested.
