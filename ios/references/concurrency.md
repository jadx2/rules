# Concurrency and language defaults

Existing projects: ask before changing language mode or concurrency settings (SKILL.md must).

- New projects use Swift 6 language mode, unless the project already uses another mode or has a convention for this. Why: the compiler enforces the concurrency rules.
- New projects set `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, unless the project already uses another isolation default. Why: MainActor by default keeps annotation noise low.
- New projects set `SWIFT_APPROACHABLE_CONCURRENCY = YES`, unless the project already has a convention for this.
- Set all three explicitly in the build settings. Xcode 27's template still starts in Swift 5 mode.
- Minimum iOS for new projects is iOS 26, unless the project already sets a deployment target. Why: current-and-previous-major convention, plus Liquid Glass and current SwiftUI with no `#available` noise.
- When a client needs wider reach, the documented floor is iOS 17, unless the project already documents another floor.

```
SWIFT_VERSION = 6.0
SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor
SWIFT_APPROACHABLE_CONCURRENCY = YES
```
