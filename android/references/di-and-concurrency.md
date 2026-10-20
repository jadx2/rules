# DI and concurrency

The SKILL.md musts (no second DI library, injected dispatchers, no `GlobalScope`) apply; this file adds the how.

## Hilt
- Use Hilt, unless the project already uses another DI library.
- Use constructor injection, `@Binds` for interfaces and `@Provides` for builders and third-party types, unless the project already has another binding style.
- Prefer a custom `@Qualifier` over `@Named`, unless the project already uses `@Named`.
- Add scopes only for shared state, unless the project already scopes differently.
- Put Hilt modules next to what they bind, unless the project already centralizes them.
- DI audit: flag composables that resolve dependencies directly.

## Concurrency
- Combine flows with `combine`/`stateIn` and use `flatMapLatest` for query-driven loads, unless the project already has another convention.
- Use `runBlocking` only in a `fun main()` entry point, unless the project already uses it elsewhere. Tests use `runTest` (SKILL.md must).
