# Tooling and logging

The SKILL.md musts (run formatter, linter and tests before finishing; no `Log`/`println`; no sensitive data in logs) apply; this file adds the how.

## Tooling
- Use ktlint 1.x plus Android Lint with Slack's `compose-lints`, unless the project already has another formatter or linter setup.
- Do not add Detekt until 2.0 is stable, unless the project already uses it.

## Logging
- Use Timber, unless the project already uses another logging wrapper.
- Plant `DebugTree` in debug only, unless the project already plants trees differently. Release forwards warnings and errors to the crash reporter or logs nothing.
- There is no crash-reporting default: follow the project's SDK.
