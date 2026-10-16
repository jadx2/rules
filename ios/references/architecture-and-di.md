# Architecture, DI and navigation

- Use MVVM with `@MainActor @Observable` view models, unless the project already uses another architecture.
- Skip the view model for screens that only display data, unless the project already has a convention for this.
- Use Factory 3 for DI, unless the project already uses another DI approach. Why: compile-time safe, with Swift Testing support.
- Register services as protocols, unless the project already registers concrete types.
- View models and services take dependencies with `@Injected`, unless the project already injects differently. Views get their services through a view model (SKILL.md DI audit).
- Use `@Entry` environment values only for values that depend on the view-tree position, such as a per-tab router, unless the project already uses another mechanism.
- Navigate with `NavigationStack` and a path-based router, one router per tab, passed through `@Entry`, unless the project already has a navigation convention.

```swift
@MainActor @Observable
final class HomeRouter {
    var path: [HomeRoute] = []
    func push(_ route: HomeRoute) { path.append(route) }
}

extension EnvironmentValues {
    // No sensible default router exists: each tab injects its own with .environment(\.homeRouter, router).
    @Entry var homeRouter: HomeRouter? = nil
}
```

```swift
@MainActor @Observable
final class ProfileViewModel {
    @ObservationIgnored @Injected(\.profileService) private var service
}
```
