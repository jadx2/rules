# iOS project generator: gap analysis against October 2026 practice

- Researched 2026-10-15.
- Subject: https://github.com/uguizu/ios-project-generator at commit `def181d` (2026-02-27). It was read-only and nothing in it was executed.
- File references (`lib/templates/x.sh:N`) are relative to that clone.
- Context: Xcode 27 (Swift 6.4, iOS 27 SDK) is current, and Swift 6.4 was released 2026-09-15. The generator predates both, plus WWDC26.

Source flags:
- **V**: the page was opened and read. Apple pages were read through their DocC JSON mirror.
- **S**: seen only in a search snippet or secondary summary.
- **U**: unverified.

Labels:
- "Community" marks opinion from blogs or forums, not Apple, swift.org or a maintainer.
- "Inference" marks my own reasoning from the cited facts.

---

## Summary: top 10 gaps, ranked by impact

1. **No `PrivacyInfo.xcprivacy`.** UserDefaults alone is a required-reason API (CA92.1). App Store Connect has rejected uploads without a manifest since 2024-05-01.
2. **Swift 6 mode without Xcode 26+ concurrency settings.** No `SWIFT_DEFAULT_ACTOR_ISOLATION=MainActor` and no Approachable Concurrency. Generated projects miss the defaults Apple applies to new app projects.
3. **Networking types are not Sendable or type-safe.** `[String: Any]` params, `any Encodable` with `AnyEncodable`, progress callbacks that are silently ignored, and an `upload` that never uploads. A dormant library's pattern (Moya) is hand-copied.
4. **No lint/format tooling and no CI config.** SwiftLint, SwiftFormat and the toolchain-bundled swift-format all exist; the generator wires none of them, so nothing enforces anything.
5. **No state/navigation/DI scaffold.** "Use MVVM" is a sentence in CLAUDE.md. There is no `@Observable` model, no `NavigationStack` routing, and DI is default-argument init.
6. **The design system works against the platform.**
   - The M3 tokens are hard-coded `UIColor` closures with no increased-contrast variants, which the HIG asks for.
   - Liquid Glass guidance says to drop custom bar backgrounds.
   - The `UIDesignRequiresCompatibility` opt-out is ignored when building with the iOS 27 SDK.
7. **No String Catalog and no accessibility baseline.** Xcode 26 generates type-safe string symbols. There is no `performAccessibilityAudit` test, and Accessibility Nutrition Labels are announced to become mandatory.
8. **Environments are modeled as three app targets, each with one xcconfig shared by Debug and Release.** That duplicates target config and can't vary per configuration. QA/Dev schemes run no tests, there is no secrets strategy, and `.gitignore` drops `Package.resolved`, so dependency versions aren't pinned.
9. **Fastlane setup is dated.** Apple ID in the Appfile rather than an App Store Connect API key, unpinned fastlane with no `Gemfile.lock` guidance, match not `readonly` on CI, and no CI workflow.
10. **No logging or observability.** No `os.Logger` wrapper or `OSSignposter`. No MetricKit, which was itself re-shaped in iOS 27 (`MetricManager` is iOS 27.0+, and `MXMetricManager` is deprecated from iOS 27.2).

**Correctness issues in the generated code.** These are smaller than the gaps above but should be fixed regardless:
- `SampleService` throws away the server `id` and assigns `UUID()`, so identity changes on every fetch (`lib/templates/swift_sources.sh:556`).
- The CLAUDE.md example `var task` has `case` without a `switch`, which is invalid Swift (`lib/templates/claude_md.sh:72-75`).
- CLAUDE.md says "No UIKit" (`claude_md.sh:14`), yet `ThemeColors.swift` is `import UIKit` / `UIColor` (`lib/templates/colors.sh:844-852`).

---

## 1. Swift language mode and concurrency

**Generator today**
- `SWIFT_VERSION` is "5" or "6", defaulting to 6 (`lib/validation.sh:43-50`, `lib/prompts.sh:179,247`). It is written to `project.yml` at `lib/templates/project_yml.sh:17`.
- None of these are set: `SWIFT_DEFAULT_ACTOR_ISOLATION`, `SWIFT_APPROACHABLE_CONCURRENCY`, or upcoming-feature flags.
- No type is `Sendable` or `@MainActor`. Examples: `NetworkClientProtocol` (`swift_sources.sh:234`) and `MockNetworkClient`, a mutable `final class` (`swift_sources.sh:580-583`).
- The Swift 5 path switches tests to XCTest (`lib/templates/tests.sh:7,57`).

**Current guidance**
- **Releases.**
  - Swift 6.4 was released 2026-09-15 (https://swift.org/blog/swift-6.4-released/, V). Xcode 27 ships Swift 6.4 and the iOS 27 SDK (https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes, V).
  - Xcode 26 ships Swift 6.2 (https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes, V).
- **Default MainActor isolation (SE-0466, Swift 6.2).**
  - Set per module with `-default-isolation MainActor` (`SWIFT_DEFAULT_ACTOR_ISOLATION`) or with SwiftPM `.defaultIsolation(MainActor.self)`. Without the flag, a module stays nonisolated. https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md (V)
- **Apple's recommendation (WWDC25 "Embracing Swift concurrency", June 2025).**
  - Main-actor-by-default "is enabled by default for new app projects created with Xcode 26."
  - "We recommend that all projects adopt [Approachable Concurrency]."
  - Use MainActor for app/UI modules and nonisolated for general-purpose libraries.
  - https://developer.apple.com/videos/play/wwdc2025/268/ (V)
- **Related proposals.**
  - SE-0461 makes nonisolated async functions run on the caller's actor (`NonisolatedNonsendingByDefault`), with `@concurrent` to opt out: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md (V).
  - SE-0470 adds isolated conformances (`extension C: @MainActor P`): https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md (V).
  - Approachable Concurrency is a bundle of upcoming-feature flags whose current list is in Quick Help (Apple DTS, https://developer.apple.com/forums/thread/793791, V).
- **Template defaults, read from the Xcode 27.0 (27A266a) template files on 2026-10-15 (V):**
  - `Project Templates/Base/App Base.xctemplate/TemplateInfo.plist` sets `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` and `SWIFT_APPROACHABLE_CONCURRENCY = YES`.
  - `Project Templates/Base/Base_ProjectSettings.xctemplate/TemplateInfo.plist` sets `SWIFT_VERSION = 5.0`, so new app projects still start in Swift 5 language mode. Only the SwiftUI Document App base sets 6.0. This matches a Hacking with Swift forum post (https://www.hackingwithswift.com/forums/swift/new-xcode-16-4-projects-default-to-swift-5-why/29888, community).
  - Not yet confirmed by building a fresh project and diffing its settings. That would show the inherited values end to end.
- **Swift 5 mode is still supported.**
  - Xcode 26.x and 27.x list language modes 6, 5, 4.2 and 4 (https://developer.apple.com/support/xcode/, V).
  - The swift.org migration guide frames Swift 6 mode as opt-in and incremental (https://www.swift.org/migration/documentation/migrationguide/, V).

**Gap / recommendation**
- **Recommended default:** Swift 6 language mode with MainActor default isolation and Approachable Concurrency on the **app target**.
  - Because XcodeGen doesn't inherit Xcode's template defaults, `project.yml` must set these explicitly (inference).
- **Local packages** (if adopted, see §2):
  - Networking and model packages stay nonisolated.
  - Their public types are `Sendable`, and slow work is marked `@concurrent` where needed.
- **Swift 5 option:** not sensible as a generator choice for new projects.
  - Its only real use is legacy code. A new project has no legacy code.
  - Keeping it doubles the test templates (Swift Testing vs XCTest) for no gain. Swift Testing works in Swift 5 mode anyway (§5), so the coupling in `tests.sh:7` is wrong on its own terms.

**Community split?** Yes, on default MainActor outside UI code.
- Donny Wals: opt in for app targets and avoid it for networking packages (https://www.donnywals.com/should-you-opt-in-to-swift-6-2s-main-actor-isolation/, 2025-09-11, V).
- Fatbobman documents friction with SwiftData and Codable types needing `nonisolated` (https://fatbobman.com/en/posts/default-actor-isolation/, S).
- Some teams stay on Swift 5 mode plus Approachable Concurrency during migration. For a greenfield project that argument doesn't apply.

**Confidence:** High on proposals and the WWDC25 recommendation. Medium on the exact Xcode 27 template defaults (unverified).

---

## 2. Project generation and modularization

**Generator today**
- XcodeGen `project.yml` with classic groups. `sources: - path: Sources` is repeated per target (`project_yml.sh:39-42, 81-84, 119-122`).
- Three app targets with near-identical settings blocks (`project_yml.sh:34-147`).
- No `options.defaultSourceDirectoryType`, no `packages:`, no local SPM modules.
- `.gitignore` ignores `*.xcodeproj` and `Package.resolved` (`lib/templates/gitignore.sh:18,23`).

**Current guidance**
- **XcodeGen** is maintained but slow-moving. 2.46.0 shipped 2026-07-16; the repo has about 401 open issues (https://github.com/yonaskolb/XcodeGen/releases/tag/2.46.0, V).
  - It supports Xcode 16 synchronized folders via `type: syncedFolder`, or project-wide via `options.defaultSourceDirectoryType: syncedFolder`. Support started in 2.44.0, with fixes through 2.45.4.
  - Source: https://raw.githubusercontent.com/yonaskolb/XcodeGen/master/CHANGELOG.md (V).
- **Xcode 16 buildable folders.** "Buildable folders only record the folder path into the project file without enumerating the contained files", which reduces merge conflicts (Xcode 16 release notes, https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes, V).
  - Whether new Xcode projects default to them is U.
- **Tuist 4.x** is very active: 4.212.0 shipped 2026-10-12 (https://github.com/tuist/tuist/releases, V).
  - It uses Swift manifests (`Project.swift`) and is pitched at modularization (https://tuist.dev/en/docs/guides/features/projects, V).
  - It has supported buildable folders since 4.62.0 (2025-08-18, https://community.tuist.dev/t/support-for-buildable-folders-in-tuist-4-62-0/723, V).
  - Its binary cache is a paid feature (https://pepicrft.me/blog/cache-for-any-xcode-project/, 2025-10-22, V). Pricing is U.
- **Local SPM modularization** uses a thin app target plus feature/core modules in a local `Package.swift`.
  - Point-Free's isowords uses 86 modules (https://github.com/pointfreeco/isowords, V).
  - Majid Jabrayilov's "microapps" series (community): https://swiftwithmajid.com/2022/01/12/microapps-architecture-in-swift-spm-basics/ (S) and https://swiftwithmajid.com/2025/05/27/microapps-architecture-in-swift-scaling/ (S).
  - Bonus (inference from SE-0466): packages let you set isolation per module.
- **Pinning dependencies.** `Package.resolved` should be committed for apps so builds are reproducible.
  - For an Xcode project it lives in `*.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/`. The generator's `*.xcodeproj` ignore discards it, as does the explicit `Package.resolved` ignore (inference; U as a cited rule).

**Gap / recommendation**
- **Synchronized folders.** Turn on `defaultSourceDirectoryType: syncedFolder` so a regenerated project behaves like a modern Xcode project.
- **Modularization.** Pick a convention: start with one local package (`Packages/Core`) holding networking and models, then add feature modules when the pattern is clear.
- **Pinning.** Stop ignoring `Package.resolved`. Either commit the generated `.xcodeproj`, or copy the resolved file to a tracked location, or switch to SPM-hosted dependencies via the local package. The local package's own `Package.resolved` is tracked.
- **Deduplication.** Collapse the three duplicated app targets (see §7).

**Community split?** Yes.
- Since Xcode 16 buildable folders, the main merge-conflict argument for generators is weaker. Small apps increasingly use plain Xcode plus local SPM packages.
- Generators still pay off for many targets and modules and for consistent settings. Tuist's own case: https://tuist.dev/blog/2025/03/21/git-conflicts (S).
- XcodeGen is simpler but slower-moving. Tuist is richer, has commercial features and needs a heavier toolchain.

**Confidence:** High on tool facts. Medium on the "which default" judgement.

---

## 3. Architecture, state, navigation and DI

**Generator today**
- `App` → `ContentView` with a static "Hello, world!" (`swift_sources.sh:7-44`).
- No view model, no `@Observable`, no navigation container.
- CLAUDE.md says "Use MVVM pattern for view models" (`claude_md.sh:200`) but nothing demonstrates it.
- DI is a default-argument initializer: `init(networkClient: NetworkClientProtocol = NetworkClient())` (`swift_sources.sh:550`).

**Current guidance**
- **Observation is the state model from iOS 17.** Apple's migration article lists the benefits as fine-grained invalidation, `@State`/`@Environment` instead of object wrappers, and tracking of optionals and collections.
  - Source: https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro (V).
- **Wrapper roles** (WWDC23 "Discover Observation in SwiftUI", https://developer.apple.com/videos/play/wwdc2023/10149/, V):
  - `@State` for data the view owns.
  - `@Environment(Type.self)` for shared data.
  - `@Bindable` when the view needs bindings.
- **Earlier deployment targets:** pointfreeco/swift-perception, 2.0.12 (2026-08-30, V).
- **Navigation.** Apps targeting iOS 16+ should "transition away from" `NavigationView` toward `NavigationStack`/`NavigationSplitView` (https://developer.apple.com/documentation/swiftui/migrating-to-new-navigation-types, V).
  - Community convention is an `@Observable` router that owns a `NavigationPath` or a typed route array, with `navigationDestination(for:)` (S; e.g. https://techconcepts.org/blog/swiftui-navigation-2026, May 2026).
- **DI options:**
  - SwiftUI `@Entry` on `EnvironmentValues` back-deploys to iOS 13 and needs the Xcode 16 toolchain (https://developer.apple.com/documentation/swiftui/entry(), V).
  - swift-dependencies 1.17.1 (2026-08-28) works with plain `@Observable` models, no TCA needed (https://github.com/pointfreeco/swift-dependencies, V).
  - Factory 3.5.0 (2026-10-14) is Swift 6 / Xcode 26 ready, with a noted Swift 6.2 `@Injected` caveat (https://github.com/hmlongco/Factory, V).
- **TCA.**
  - 1.26.2 is the latest 1.x (2026-08-28, V).
  - 2.0 is a beta preview gated to the paid Point-Free Max tier. It drops Combine and removes `BindingReducer` (https://pointfree.co/blog/posts/206-beta-preview-composablearchitecture-2-0, 2026-04-01, V).
  - No public 2.0 release has shipped yet (S).
- **"MV" (no view model), community:**
  - Thomas Ricouard, "SwiftUI in 2025: Forget MVVM" (https://dimillian.medium.com/swiftui-in-2025-forget-mvvm-262ff2bbd2ed, 2025-06-02, S).
  - Apple forums thread https://developer.apple.com/forums/thread/710501 (S).
  - Others keep `@Observable` view models for testability (S).

**Gap / recommendation**
- **Scaffold.** Ship one worked feature that demonstrates the chosen pattern:
  - an `@Observable` model (MainActor),
  - a list → detail flow in `NavigationStack` with a typed `Route` enum,
  - dependencies injected through the initializer or `@Entry`.
- **Testing seam.** Whatever pattern is chosen, it must be unit-testable without UI.
- **DI defaults.** Replace default-argument construction of `NetworkClient()` with one composition root, the `App` struct, so production wiring lives in one place.

**Community split?** Strongly.
- MVVM vs MV vs TCA. TCA 2.0's paywalled beta adds hesitation.
- A router object vs path state in the view.
- A third-party DI container vs Environment plus init injection.
- Common ground: `@Observable` over `ObservableObject` on iOS 17+, and `NavigationStack` over `NavigationView`.
- This is an **open decision** (see below).

**Confidence:** High on Apple APIs. The pattern choice is opinion.

---

## 4. Networking

**Generator today** (`lib/templates/swift_sources.sh:123-516`)
- **Request modeling.** A Moya-inspired `Endpoint` protocol: "inspired by Moya's TargetType" (`:129-136`).
- **`HTTPTask`:**
  - It carries `requestJSON(Encodable)` (an existential) and `requestParameters([String: Any], ...)` (`:156-162`).
  - Query values are stringified with `"\(value)"` (`:356-358`).
  - `AnyEncodable` type-erases the body (`:413-423`).
- **Upload and download:**
  - `upload(_:progress:)` ignores `progress` and calls `session.data(for:)` instead of an upload API (`:283-287`).
  - `download` ignores `progress` and throws `invalidURL` for the wrong task type (`:291-294`).
- **Error mapping and response validation:**
  - `NetworkError.unknown` exists but transport errors are never mapped (`:262`).
  - Non-HTTP responses pass validation silently (`:401-403`).
- **Base URL.** `URL(string: Configuration.baseURL)!` force-unwrap (`:453`).
- **Concurrency and auth.** No `Sendable` on any type, no auth/token handling, no retry, no request logging.

**Current guidance**
- **URLSession async API surface (WWDC21 "Use async/await with URLSession", https://developer.apple.com/videos/play/wwdc2021/10095/, V):**
  - `data(for:)`, `upload(for:from:)` / `upload(for:fromFile:)`, `download(for:)` and `bytes(for:)`, each with a per-task `URLSessionTaskDelegate`.
  - Async `download` does not delete the temp file.
- **Progress.** You get the `URLSessionTask`, and so its `progress`, through the per-task delegate's `urlSession(_:didCreateTask:)` (iOS 16+, https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate/urlsession(_:didcreatetask:), V).
  - On iOS 15 the fallback is counting bytes with `bytes(for:)` (inference, U).
- **Typed throws (SE-0413, Swift 6.0).** The proposal itself says "untyped `throws` is better for most scenarios."
  - Reserve typed throws for errors that stay within a module or package and are always handled, generic pass-through code, and constrained environments.
  - "Resist the temptation to use typed throws because there is only a single kind of error."
  - Source: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md (V).
- **Moya is dormant.** Latest release 15.0.3, about 4 years old, with no async/await (https://github.com/Moya/Moya/releases, V; https://swiftpackageindex.com/Moya/Moya, V).
  - The common alternative is a generic `Request<Response>` value, as in kean/Get 2.2 (https://github.com/kean/Get, V; community).
  - No primary source rules on enum vs struct.
- **swift-openapi-generator.** 1.0 shipped Dec 2023 (https://github.com/apple/swift-openapi-generator/releases/tag/1.0.0, V; https://www.swift.org/blog/swift-openapi-generator-1.0/, S).
  - The latest release is 1.14.1, published "14 Oct", year not shown (V/U).
  - The URLSession transport supports iOS 13+ (https://github.com/apple/swift-openapi-urlsession, V).
- **Token refresh (community pattern):** an actor holds the in-flight refresh `Task` so concurrent 401s await a single refresh.
  - https://www.donnywals.com/building-a-token-refresh-flow-with-async-await-and-swift-concurrency/ (S)
  - Reentrancy caveat: https://blog.jacobstechtavern.com/p/advanced-swift-actors-re-entrancy (S)
  - No Apple source.

**Gap / recommendation**
- **Remove `[String: Any]` and `any Encodable`.** Model requests as `Sendable` values generic over the response, for example `struct Request<Response: Decodable & Sendable>: Sendable` with `[URLQueryItem]` and an `Encodable & Sendable` body. That gives compile-time body and response types without type erasure (inference; it follows from Swift 6 Sendable checking).
- **Fix or delete the progress API.** Either implement progress through the per-task delegate (iOS 16+) or remove the parameters. An ignored callback is a silent failure.
- **Fix `upload`.** Use `upload(for:from:)`.
- **Configuration.** Inject `baseURL` once at the composition root instead of force-unwrapping per endpoint.
- **Errors.** Keep untyped `throws` at the client's public surface per SE-0413. Map transport errors (`URLError`) explicitly.
- **Seams.** Add an auth/interceptor hook (header injection plus single-flight refresh) and request/response logging via `Logger` (§10).
- **OpenAPI.** If the backend publishes an OpenAPI spec, offer swift-openapi-generator as the alternative.

**Community split?** Yes:
- enum endpoints vs request structs;
- hand-rolled client vs OpenAPI codegen;
- typed throws for a domain `NetworkError`. The proposal discourages it at API boundaries; some teams use it anyway.

**Confidence:** High on API and proposal facts and on the generated-code defects (read directly). Medium on "request structs are the convention" (community only).

---

## 5. Testing

**Generator today**
- Swift Testing only when `SWIFT_VERSION == 6`, XCTest otherwise (`tests.sh:7-114`).
- The Swift Testing suite copies XCTest idioms: implicitly unwrapped `var sut: SampleService!`, a `test` prefix on `@Test` names, and do/catch plus `Issue.record` instead of `#expect(throws:)` (`tests.sh:18-54`).
- `MockNetworkClient` is a non-Sendable mutable class matching `mockResponse as? T` (`swift_sources.sh:580-617`).
- The UI test asserts on a literal string instead of an accessibility identifier (`tests.sh:132`).
- No test plan, no snapshot tests, and no parameterized-test example.
- Coverage is on (`project_yml.sh:70,190`). Tests run only in the Production scheme.

**Current guidance**
- **Swift Testing works in Swift 5 mode.** The swift-testing team said Swift 5 mode "will still be supported" (https://forums.swift.org/t/an-update-on-swift-testing-progress-and-stable-release-plans/71455, 2024-04-24, V). It ships with Xcode 16+ (https://github.com/swiftlang/swift-testing, V).
- **What Xcode added since.**
  - Xcode 26 / Swift 6.2: exit tests (`#expect(processExitsWith:)`) and attachments (Xcode 26 release notes, V; WWDC25 https://developer.apple.com/videos/play/wwdc2025/245/, V).
  - Xcode 27 / Swift 6.4: XCTest ↔ Swift Testing interop (assertions work across both), issue severity, `Test.cancel`, `--repeat-until`, and a test-plan interop mode (Xcode 27 release notes, V; WWDC26 https://developer.apple.com/videos/play/wwdc2026/262/, V).
  - New Xcode projects use "Complete" interop mode (WWDC26 "Migrate to Swift Testing", https://developer.apple.com/videos/play/wwdc2026/267/, V).
- **XCTest is still required for UI and performance tests.** "UI automation and performance testing APIs are only available in XCTest" (WWDC26 267, V).
- **API conventions** (https://developer.apple.com/documentation/testing/migratingfromxctest, V):
  - parameterized tests (`arguments:`);
  - traits (`.tags`, `.enabled(if:)`, `.timeLimit` in minutes);
  - `confirmation()` replaces expectations;
  - `withKnownIssue`;
  - suites run in parallel by default, with `.serialized` as the opt-out.
- **Sendable mocks:**
  - `Mutex` (iOS 18+, https://developer.apple.com/documentation/synchronization/mutex, V).
  - `OSAllocatedUnfairLock` (iOS 16+, Sendable, https://developer.apple.com/documentation/os/osallocatedunfairlock, V).
  - Actor mocks and `@unchecked Sendable` with a lock are community conventions (no primary source).
- **Snapshot tests.** pointfreeco/swift-snapshot-testing has supported Swift Testing since 1.17.0 (2024-06-08). The README shows `@Suite(.snapshots(record: .failed))` (https://pointfree.co/blog/posts/146-swift-testing-support-for-snapshottesting, V; https://github.com/pointfreeco/swift-snapshot-testing, V).
- **Test plans.** These group configurations (env, sanitizers, localization, coverage) and run with `xcodebuild -testPlan` (https://developer.apple.com/documentation/xcode/organizing-tests-to-improve-feedback, V).

**Gap / recommendation**
- **Swift Testing everywhere for unit tests**, decoupled from the language-mode choice. XCTest only for UI and performance tests.
- **Idiomatic sample tests:**
  - `let` properties built in `init`;
  - descriptive names or display names, not `testX`;
  - `#expect(throws:)`;
  - one parameterized example.
- **Mocks.** Generate them as `Sendable`: either a `final class` guarded by `Mutex`/`OSAllocatedUnfairLock`, or an `actor`. Use typed stubs per method instead of `Any` plus casting.
- **Test plans.** Generate `.xctestplan` files: a unit plan and a UI plan, the latter including `performAccessibilityAudit` (§9).
- **UI tests.** Query by `accessibilityIdentifier`, not by copy.
- **Snapshot testing:** decide whether to include it (open decision).

**Community split?**
- Migrate existing XCTest now vs only new tests (irrelevant for greenfield).
- Lock-based vs actor mocks: actors force `await` in assertions.
- Snapshot testing's experimental trait API.

**Confidence:** High.

---

## 6. Code style tooling

**Generator today:** none. No `.swiftlint.yml`, `.swiftformat` or `.swift-format`, no build-phase plugin, no pre-commit hook. CLAUDE.md says "Follow Swift API Design Guidelines" (`claude_md.sh:198`) with nothing to enforce it.

**Current guidance**
- **SwiftLint** 0.65.1 (2026-08-21) recommends its plugins from SimplyDanny/SwiftLintPlugins and running it with the same toolchain you compile with (https://github.com/realm/SwiftLint/releases/tag/0.65.1, V; https://github.com/SimplyDanny/SwiftLintPlugins, V).
  - Explicit Swift 6.4 / Xcode 27 support is not stated: U.
- **SwiftFormat (nicklockwood)** 0.63.1 (2026-09-30). Its FAQ says SwiftLint finds problems while SwiftFormat fixes them, and that using both is fine (https://github.com/nicklockwood/SwiftFormat, V).
  - A build-phase integration needs `ENABLE_USER_SCRIPT_SANDBOXING=NO`.
- **swift-format (swiftlang)** 604.0.0 (2026-09-16): "Swift 6 (included with Xcode 16) and above include swift-format in the toolchain". It runs as `swift format`. https://github.com/swiftlang/swift-format (V)
  - Xcode's "Format File with 'swift-format'" menu item is documented by the community (https://troz.net/post/2024/swift_format/, 2024-11-06, V). I didn't find it in Apple's release notes (U).
- **Swift API Design Guidelines:** https://www.swift.org/documentation/api-design-guidelines/ (V). "Clarity at the point of use is your most important goal."

**Gap / recommendation**
- Ship one formatter config and one linter config, with a single command that runs both (a Makefile target or script) and a CI job that fails on violations.
- Prefer CI or pre-commit checks over build-phase scripts so builds stay fast and sandboxed. Agents also benefit from one deterministic `format` command.

**Community split?** Yes.
- The most common combination is SwiftFormat plus SwiftLint.
- The growing alternative is swift-format alone: zero dependencies, toolchain-bundled, and wired into Xcode.
- Never run two formatters.
- This is an open decision.

**Confidence:** High on tool facts.

---

## 7. Configuration and environments

**Generator today**
- **Environments as targets.** Environments are separate app targets (`<Name>`, `<Name>QA`, `<Name>Dev`). Each target maps **both** Debug and Release to one xcconfig (`project_yml.sh:43-45, 85-87, 123-125`).
- **xcconfig contents.** The xcconfigs hold only `API_BASE_URL` and `APP_DISPLAY_NAME` (`swift_sources.sh:627-664`).
- **Config reading.** `Configuration.baseURL` reads Info.plist and calls `fatalError` if the key is missing (`swift_sources.sh:681-692`). Consumers then force-unwrap `URL(string:)!` (`:453`).
- **Tests and schemes.** Test targets are tied to Production only (`project_yml.sh:150-177`), and QA/Dev schemes have no test action (`:206-243`).
- **Secrets.** `.gitignore` lists `*.xcconfig.local` (`gitignore.sh:43`), but nothing `#include?`s it.

**Current guidance**
- **Apple's xcconfig model.** Apple documents one xcconfig per build configuration, `#include`, `#include?` (suppresses the warning when the file is missing) and `[config=…]`/`[sdk=…]` conditionals (https://developer.apple.com/documentation/xcode/adding-a-build-configuration-file-to-your-project, V).
  - Apple gives no guidance on targets vs configurations, or on secrets.
- **NSHipster** (2020-02-27, https://nshipster.com/xcconfig, V, community):
  - "Don't use xcconfig files to store secrets like API keys."
  - Examples attach per-environment xcconfigs to configurations.
  - Its own runtime reader is fail-fast (`try!`).
- **thoughtbot** uses environment configurations that belong to no target (https://robots.thoughtbot.com/let-s-setup-your-ios-environments, S).
- **Secrets end up in the binary.** Anything passed through xcconfig → Info.plist is readable from the IPA. High-value secrets belong on a backend (community: https://nshipster.com/xcconfig, V; https://developer.apple.com/forums/thread/805449, S).
- **fatalError on missing config.** No authoritative source either way (U). Fail-fast at launch for a required key is a defensible community position.

**Gap / recommendation**
- **Configurations, not targets.** Prefer build configurations plus schemes (`Debug-Dev`, `Release-Dev`, `Debug-QA`, …) on one app target, or at minimum separate Debug/Release xcconfigs per environment.
  - That removes the 3× duplicated target block, makes tests run against every environment, and lets Debug-only settings differ.
  - XcodeGen supports this through `configs:` and `configFiles:`.
- **Validate config once.** Parse it into a typed `AppConfig` (with a real `URL`) at startup in the composition root. Fail fast there with a clear message, not lazily inside an endpoint getter.
- **Secrets.** Add `#include? "Secrets.xcconfig"` (gitignored) for local non-secret overrides, document CI injection, and state plainly that real secrets do not belong in the app.

**Community split?** Targets vs configurations: most guides favor configurations plus schemes. Gitignored include vs CI-injected values: both keep values out of git, not out of the binary.

**Confidence:** Medium-high. Apple documents the mechanism but doesn't prescribe a structure.

---

## 8. Design system

**Generator today**
- **Hard-coded M3 palette.** Material Design 3 tokens are generated from fixed palettes or a custom hex via a bundled JS engine run through `osascript` and `eval` (`lib/templates/colors.sh:815-827`).
- **Light/dark only.** Tokens are emitted as `UIColor { traits in ... }` dynamic providers with light and dark values only (`colors.sh:844-1061`), then wrapped as `Color.Theme.*` (`colors.sh:1075+`).
- **Asset catalog unused.** It only has an empty `AccentColor` (`swift_sources.sh:85-97`).
- **CLAUDE.md** mandates M3 tokens for all colors (`claude_md.sh:201`).
- No typography, Dynamic Type or spacing tokens.

**Current guidance**
- **HIG Color** (changelog: Liquid Glass guidance added 2025-06-09, updated 2025-12-16; https://developer.apple.com/design/human-interface-guidelines/color, V):
  - "If you define a custom color, make sure to supply light and dark variants, and an increased contrast option for each variant."
  - "Avoid hard-coding system color values."
  - "Even if your app ships in a single appearance mode, provide both light and dark colors to support Liquid Glass adaptivity."
  - "Apply color sparingly to the Liquid Glass material."
- **HIG Typography** (https://developer.apple.com/design/human-interface-guidelines/typography, V):
  - "Make sure your app's layout adapts to all font sizes."
  - Custom fonts must support Dynamic Type.
- **Adopting Liquid Glass** (https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass, V):
  - "Reduce your use of custom backgrounds in controls and navigation elements… Prefer to remove custom effects and let the system determine the background appearance." This covers `NavigationStack`, `.toolbar`, tab bars and similar.
  - `glassEffect` is iOS 26+ (V).
- **`UIDesignRequiresCompatibility`.** "The system ignores this key when you build for iOS 27 or later" (https://developer.apple.com/documentation/bundleresources/information-property-list/uidesignrequirescompatibility, V).
  - Inference: with Xcode 27 shipped, the opt-out is gone for new builds.
- **Material on iOS.** Google's Material Components for iOS has been in maintenance mode since 2021-07-15 and points to "modern UIKit components or SwiftUI instead" (https://github.com/material-components/material-components-ios, V).
  - Current M3 guidance on iOS adaptation could not be read, because m3.material.io is JS-rendered (U).

**Gap / recommendation**
- **Separate token names from color sources.** Keep a semantic token layer, using the company's own role names or M3 role names if Android parity matters. Back the tokens with **asset-catalog named colors** carrying Any/Dark × normal/High Contrast variants, or with system semantic colors, instead of hard-coded `UIColor` closures.
- **Add tokens beyond color:** typography via `Font.TextStyle` and Dynamic Type, plus spacing.
- **Liquid Glass rule.** Don't theme navigation, tab or tool bars. Let Liquid Glass render them.
- **CLAUDE.md fixes.** Remove the "No UIKit" contradiction. Replace "M3 tokens for all colors" with "semantic tokens only; no literal colors in views".

**Community split?** Yes.
- Brand-driven, cross-platform design systems (M3 naming shared with Android) vs iOS-native styling.
- Liquid Glass widens the gap: Apple explicitly asks you to remove custom bar backgrounds.
- This is an open decision.

**Confidence:** High on Apple guidance. The M3-on-iOS judgement is partly inference.

---

## 9. Localization, accessibility, privacy manifest, App Store requirements

**Generator today**
- `SWIFT_EMIT_LOC_STRINGS: YES` (`project_yml.sh:22`), but no `Localizable.xcstrings`, and `NetworkError` messages are English literals (`swift_sources.sh:210-229`).
- No accessibility identifiers or guidance, and no audit test.
- No `PrivacyInfo.xcprivacy`.

**Current guidance**
- **String Catalogs** (https://developer.apple.com/documentation/xcode/localizing-and-varying-text-with-a-string-catalog, V). Xcode 26 adds generated type-safe symbols (`Text(.introductionTitle)`), on by default for new projects, plus AI-generated translator comments (WWDC25 "Code-along: Explore localization with Xcode", https://developer.apple.com/videos/play/wwdc2025/225/, V).
- **Accessibility Nutrition Labels** are voluntary for now. Apple says that "over time, you'll be required to share accessibility support details to submit new apps and app updates" (https://developer.apple.com/help/app-store-connect/manage-app-accessibility/overview-of-accessibility-nutrition-labels, V). There is no date (U).
- **`XCUIApplication.performAccessibilityAudit`** is available from iOS 17 (https://developer.apple.com/documentation/xcuiautomation/xcuiapplication/performaccessibilityaudit(for:_:), V). See also WWDC23 "Perform accessibility audits for your app" (https://developer.apple.com/videos/play/wwdc2023/10035/, S).
- **Privacy manifests:**
  - Since 2024-05-01, apps that don't declare required-reason API use are not accepted (https://developer.apple.com/documentation/bundleresources/describing-use-of-required-reason-api, V).
  - The categories include UserDefaults (CA92.1 for app-only access) and FileTimestamp, SystemBootTime, DiskSpace and ActiveKeyboards (https://developer.apple.com/documentation/bundleresources/app-privacy-configuration/nsprivacyaccessedapitypes/nsprivacyaccessedapitype, V).
  - File format: https://developer.apple.com/documentation/bundleresources/privacy-manifest-files (V).
  - Listed third-party SDKs (Alamofire, Firebase, Kingfisher, …) need their own manifests and signatures (https://developer.apple.com/support/third-party-SDK-requirements/, V).
- **App Store upload requirements** (https://developer.apple.com/news/upcoming-requirements/, V):
  - since 2026-04-28, apps must be built with Xcode 26+ and the iOS 26 SDK;
  - since 2026-09-09, apps must target iOS 13+;
  - age-rating questionnaire answers were due 2026-01-31, with new tiers 4+/9+/13+/16+/18+.

**Gap / recommendation**
- **Privacy manifest.** Generate `PrivacyInfo.xcprivacy` with `NSPrivacyTracking=false` and a UserDefaults/CA92.1 entry, flagged for review.
- **Localization.** Generate an empty `Localizable.xcstrings` and a convention: user-facing strings via `LocalizedStringResource` or generated symbols. Errors expose localized messages.
- **Accessibility baseline:**
  - Dynamic Type through text styles;
  - `accessibilityIdentifier` on testable elements;
  - one UI test calling `performAccessibilityAudit()`;
  - CLAUDE.md rules on labels and hit targets.
- **Tooling floor.** Pin Xcode 26 as the minimum (README says Xcode 15/16, `README.md:23`) and recommend Xcode 27.

**Community split?** Mostly none; the privacy manifest is mandatory. There is debate on gating PRs on accessibility audits because of false positives.

**Confidence:** High.

---

## 10. Logging and observability

**Generator today:** nothing. No `Logger`, no signposts, no MetricKit, no crash reporting.

**Current guidance**
- **`os.Logger`** (iOS 14) takes a subsystem (bundle ID) and a category.
  - Interpolations are private by default; mark them `privacy: .public` or `.private(mask: .hash)`.
  - Debug-level messages cost nothing when discarded.
  - Source: WWDC20 "Explore logging in Swift", https://developer.apple.com/videos/play/wwdc2020/10168/ (V).
- **Apple on print():** "'print' should rarely be used". Apple recommends a log handle per component, and Xcode 15 added structured console filtering (WWDC23 "Debug with structured logging", https://developer.apple.com/videos/play/wwdc2023/10226/, V).
- **`OSSignposter`** (iOS 15) measures intervals in Instruments (https://developer.apple.com/documentation/os/ossignposter, V).
- **MetricKit:**
  - `MXMetricManager` is deprecated in iOS 27.2 ("Use MetricManager instead"), and `MetricManager` is iOS 27.0+ (https://developer.apple.com/documentation/metrickit/mxmetricmanager, V; https://developer.apple.com/documentation/metrickit/metricmanager, V).
  - Diagnostics are delivered immediately on iOS 15+.
  - `MXCrashDiagnostic` is also deprecated in favor of `DiagnosticResult` (V).
- **swift-log** is an API package that needs a backend, mainly useful for code shared with servers (https://github.com/apple/swift-log, V).

**Gap / recommendation**
- **Logging.** Generate `Logger` extensions with categories (`network`, `ui`, …). The network client logs method, path and status with privacy-annotated values.
- **Performance.** Add an `OSSignposter` example on request latency.
- **Diagnostics.** Optionally add a MetricKit subscriber with an `if #available(iOS 27, *)` branch.
- **Crash reporting and CLAUDE.md.** Document the crash-reporter decision (Sentry, Crashlytics or none) and add "no `print`" to CLAUDE.md.

**Community split?**
- MetricKit alone vs a third-party crash SDK: MetricKit gives no real-time alerts.
- Plain `Logger` vs swift-log.

**Confidence:** High on APIs.

---

## 11. CI/CD and release

**Generator today** (`lib/templates/fastlane.sh`)
- **Gemfile.** `gem "fastlane"` is unpinned, with no lockfile guidance (`:22-26`).
- **Appfile.** Uses `apple_id("")`, i.e. Apple ID auth (`:35-37`).
- **match.**
  - git storage, `type("appstore")` (`:46-51`).
  - The `certificates` lane runs match without `readonly` (`:87-90`).
- **Build and upload.**
  - The `build` lane runs `xcodegen generate` then `build_app(export_method: "app-store", clean: true)` (`:95-109`).
  - `beta` uploads with `skip_waiting_for_build_processing` (`:114-122`).
- No CI workflow, no test lane, no version or build-number bump.

**Current guidance**
- **fastlane is actively maintained by volunteers:** 2.241.1 was released 15 Oct (https://github.com/fastlane/fastlane/releases, V).
  - It moved under the Mobile Native Foundation, with the transfer "fully completed" in early 2026 and an OpenCollective for funding (https://opencollective.com/fastlane/updates/monthly-releases, 2026-04-16, V).
  - 2.239.0 made Ruby 3.1 the minimum (V).
- **Pin fastlane.** "Add both the ./Gemfile and the ./Gemfile.lock to version control", and run via `bundle exec` (https://docs.fastlane.tools/getting-started/ios/setup/, V).
- **Use an API key.** "It is recommended to use the API Key authentication when you are able to": no 2FA, and more reliable (https://docs.fastlane.tools/app-store-connect-api/, V).
- **match.** Storage can be git, Google Cloud, S3 or GitLab. Use `readonly: is_ci` on CI, and `api_key` is supported (https://docs.fastlane.tools/actions/match/, V).
- **Alternative to match: Xcode cloud signing on CI** with `xcodebuild -allowProvisioningUpdates -authenticationKeyPath …`. Forum reports show keychain problems on clean machines (https://developer.apple.com/forums/thread/764554, S). I found no Apple doc page (U).
- **Upload paths.** Xcode, `altool --upload-app`, Transporter and the App Store Connect API are all supported; altool upload is not deprecated (https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds, V).
- **The `export_method: "app-store"` value.** From memory, Xcode 15.3+ renamed this export method to `app-store-connect`. **Unverified**; check the fastlane `build_app` docs before acting.
- **Xcode Cloud.** 25 compute hours per month are included with membership; add-ons start at 100 h for $49.99 (https://developer.apple.com/xcode-cloud/, V).
- **GitHub Actions.** `macos-26` is `macos-latest`, `macos-15` is available, `macos-14` is deprecated, and an Xcode 27 image is in preview (https://github.com/actions/runner-images, V).
- Bitrise and Codemagic were not researched (U).

**Gap / recommendation**
- **Pin and authenticate.** Pin fastlane and commit `Gemfile.lock`. Switch to `app_store_connect_api_key` with key ID, issuer and `.p8` from CI secrets.
- **Signing.** Make match `readonly: is_ci`, or adopt cloud signing (open decision).
- **Lanes.** Add `test` and `lint` lanes and build-number automation (e.g. `latest_testflight_build_number + 1`).
- **CI workflow.** Generate one, for GitHub Actions or Xcode Cloud (open decision), running lint → unit tests → UI tests on PRs and TestFlight on main or tags.

**Community split?**
- match vs cloud signing.
- fastlane vs plain `xcodebuild` scripts, given fastlane's thin funding.
- Xcode Cloud (cheap and integrated) vs GitHub Actions (flexible, but macOS minutes are expensive).

**Confidence:** High on fastlane docs. Medium on cloud signing (forum-only).

---

## 12. Minimum deployment target

**Generator today:** default iOS 17.0, minimum allowed 15.0 (`lib/validation.sh:36`, `lib/prompts.sh:169`). There is a `PreviewProvider` branch below 17 (`swift_sources.sh:45-66`).

**Current guidance**
- **Apple adoption figures** (measured 2026-06-07, https://developer.apple.com/support/app-store/, V):
  - iOS 26 runs on 86% of iPhones introduced in the last four years and 79% of all iPhones.
  - iPadOS 26 runs on 79% of recent iPads and 68% of all iPads.
- **TelemetryDeck:** iOS 18 was about 12% at the end of May 2026 (https://telemetrydeck.com/survey/apple/iOS/majorSystemVersions/, S).
- **Xcode support** (https://developer.apple.com/support/xcode/, V):
  - Xcode 26 and 27 allow deployment targets from iOS 15.
  - Xcode 27 device/simulator debugging starts at iOS 17, so iOS 15 and 16 can be shipped but not debugged with Xcode 27.
- **API gates (DocC availability, V):**
  - iOS 17: Observation and SwiftData.
  - iOS 18: `Mutex`.
  - iOS 26: Liquid Glass APIs.
  - `@Entry` back-deploys to iOS 13.
- **Community heuristic.** Hacking with Swift calls "current minus one" a sensible default (https://www.hackingwithswift.com/interview-questions/what-do-you-think-is-a-sensible-minimum-ios-deployment-target, V). After the 18 → 26 renumbering, that means iOS 26 with 27 current.

**Gap / recommendation**
- **Raise the floor.** Allow iOS 17 as the minimum, because of Observation and Xcode 27 debuggability. Drop the 15/16 branch and the `PreviewProvider` path.
- **Default.** Make the default a company decision between 17, 18 and 26.

**Community split?** Yes. Revenue-sensitive apps keep wider floors; others jump to 17 (Observation) or 18 (`Mutex`). Open decision.

**Confidence:** High on the data.

---

## 13. AI-agent repo files

**Generator today**
- A single generated `CLAUDE.md` of roughly 150–200 lines, dominated by the network-layer tutorial and the conventional-commits guide (`lib/templates/claude_md.sh`).
- It contains invalid Swift (`:72-75`), the UIKit contradiction (`:14` vs §8), and an XCTest assertion in the Swift Testing path (`:121`).
- It hard-codes the `iPhone 16` simulator (`:168`).
- There is no `AGENTS.md`.

**Current guidance**
- **AGENTS.md** is an open format, "stewarded by the Agentic AI Foundation under the Linux Foundation", and read by Codex, Cursor, Copilot, Gemini CLI, Zed and others. The closest file wins (https://agents.md/, V).
- **Claude Code** (https://code.claude.com/docs/en/memory, V):
  - It recommends keeping CLAUDE.md under 200 lines.
  - `@path` imports work, and `.claude/rules/*.md` can be path-scoped.
  - Recent versions read AGENTS.md when there is no CLAUDE.md, or a CLAUDE.md can import `@AGENTS.md`.
- **Xcode 26.3+** runs Claude Agent and Codex in-IDE and exposes Xcode tools over MCP (`xcrun mcpbridge`).
  - Apple suggests putting hints "to configuration files, such as the AGENTS.md or CLAUDE.md files" (https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode, V).
  - Xcode 27 adds plan mode and sub-agents (WWDC26, https://developer.apple.com/videos/play/wwdc2026/259/, V).
- **XcodeBuildMCP** (now "MobileBuildMCP", https://github.com/getsentry/XcodeBuildMCP, V) gives agents headless build, test and run.

**Gap / recommendation**
- **Make AGENTS.md canonical.** Generate a short `AGENTS.md` as the source of truth, plus a `CLAUDE.md` that imports it.
- **Content: commands and hard rules, not tutorials:**
  - build, test, lint and format commands that are verified to work;
  - the simulator chosen dynamically or documented;
  - architecture rules;
  - "no literal colors";
  - "no print";
  - "update PrivacyInfo when adding SDKs".
- **Move walkthroughs elsewhere.** Put long walkthroughs (adding a service) in `docs/`.

**Community split?** One AGENTS.md vs tool-specific files, and how much to include. Anthropic's docs favor short.

**Confidence:** High.

---

## 14. Additional findings (outside the 13 areas)

- **License.** The generator is GPL-3.0 (`LICENSE`). Whether its templated output counts as a derivative work, and so whether GPL obligations would attach to company apps, is a legal question I can't answer (**U**). Ask legal, or have the company own a clean-room rewrite of the templates.
- **Dependencies.** README says "Zero dependencies: No Node, Python, Ruby" (`README.md:506`). In practice custom colors run a 3,228-line JS file through `osascript` and `eval` its output (`colors.sh:819-826`), and fastlane needs Ruby.
- **Destructive overwrite.** `rm -rf` of an existing directory after a y/N prompt (`lib/scaffold.sh:7-16`). Acceptable, but worth noting for a company tool.
- **No version or build-number strategy.** `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION` are hard-coded (`project_yml.sh:19-20`).
- **Device orientation.** iPhone orientations include landscape by default (`project_yml.sh:53`). That is a product decision, not a default.

---

## Candidate criteria (accept or reject each)

Each item is a verifiable rule for a generated project.

**Language and concurrency**
- [ ] **C1.** Swift 6 language mode. Verify: `SWIFT_VERSION: "6"` in `project.yml` `settings.base`, with no "5" option in the generator.
- [ ] **C2.** The app target uses MainActor default isolation. Verify: `SWIFT_DEFAULT_ACTOR_ISOLATION: MainActor` on the app target.
- [ ] **C3.** Approachable Concurrency is on. Verify: `SWIFT_APPROACHABLE_CONCURRENCY: YES` (confirm the flag set in Xcode 27 Quick Help first).
- [ ] **C4.** Every type crossing a module or actor boundary is `Sendable`. Verify: a clean build with zero concurrency warnings: `xcodebuild build 2>&1 | grep -c 'warning:.*Sendable'` == 0.

**Project and modules**
- [ ] **C5.** Synchronized folders. Verify: `options.defaultSourceDirectoryType: syncedFolder` in `project.yml` (or a non-XcodeGen equivalent).
- [ ] **C6.** Non-UI code (networking, models) lives in a local Swift package. Verify: `Packages/*/Package.swift` exists, and the app target depends on it.
- [ ] **C7.** Dependency versions are pinned in git. Verify: `git ls-files | grep Package.resolved` is non-empty, and `.gitignore` does not list `Package.resolved`.

**Architecture**
- [ ] **C8.** State uses Observation. Verify: `rg 'ObservableObject|@Published' Sources` returns nothing.
- [ ] **C9.** Navigation uses `NavigationStack` with typed routes. Verify: `rg NavigationView Sources` returns nothing, and a `Route` enum is used with `navigationDestination(for:)`.
- [ ] **C10.** One composition root wires production dependencies. Verify: no `= NetworkClient()` or `= .shared` default arguments in `Sources` (`rg '= *NetworkClient\(\)'` returns nothing).

**Networking**
- [ ] **C11.** No `[String: Any]` or existential `Encodable` in networking. Verify: `rg '\[String: Any\]|any Encodable|AnyEncodable' Sources Packages` returns nothing.
- [ ] **C12.** No force-unwrapped URLs. Verify: `rg 'URL\(string:.*\)!' Sources Packages` returns nothing.
- [ ] **C13.** The network client has no unused parameters. Verify: the progress parameters are implemented and tested, or absent.
- [ ] **C14.** Public networking APIs use untyped `throws` (SE-0413 guidance). Verify: `rg 'throws\(' Packages/*/Sources` returns only internal uses.

**Testing**
- [ ] **C15.** Unit tests use Swift Testing. XCTest is only for UI and performance tests. Verify: `rg 'XCTestCase' Tests` returns nothing; `UITests` may use it.
- [ ] **C16.** Test doubles are `Sendable` with no `Any` storage. Verify: mocks conform to `Sendable` without `@unchecked` (or with `@unchecked` plus a `Mutex` or lock), and `rg 'mockResponse: Any' Tests` returns nothing.
- [ ] **C17.** Test plans exist and every environment's scheme runs them. Verify: `*.xctestplan` committed, and each scheme's `test` action references a plan.
- [ ] **C18.** UI tests query by accessibility identifier. Verify: `rg 'staticTexts\["' UITests` returns nothing.

**Style**
- [ ] **C19.** One formatter and one linter are configured, and CI fails on violations. Verify: a config file exists (`.swiftformat` or `.swift-format`, plus `.swiftlint.yml`), and the CI workflow has a lint step that exits non-zero on violations.

**Configuration**
- [ ] **C20.** Environments are build configurations on one app target. Verify: `project.yml` has a single `type: application` target and N `configs`.
- [ ] **C21.** Config is parsed once into a typed struct at launch. Verify: `rg 'object\(forInfoDictionaryKey' Sources` appears in exactly one file.
- [ ] **C22.** No secrets in tracked xcconfigs. Verify: tracked xcconfigs contain no keys, and `#include? "Secrets.xcconfig"` is present and gitignored.

**Design and accessibility**
- [ ] **C23.** Views contain no literal colors. Verify: `rg 'Color\(red:|UIColor\(red:|#colorLiteral' Sources/Features` returns nothing.
- [ ] **C24.** Custom colors live in the asset catalog with dark and high-contrast variants. Verify: each `*.colorset/Contents.json` has `"appearance":"luminosity"` and `"contrast"` entries.
- [ ] **C25.** Navigation, tab and tool bars get no custom backgrounds. Verify: `rg 'toolbarBackground|UINavigationBarAppearance|UITabBarAppearance' Sources` returns nothing, or each use is justified.
- [ ] **C26.** Text uses Dynamic Type styles. Verify: `rg '\.font\(\.system\(size:' Sources` returns nothing.
- [ ] **C27.** An accessibility audit runs in UI tests. Verify: `rg performAccessibilityAudit UITests` is non-empty.

**Localization, privacy and platform**
- [ ] **C28.** User-facing strings live in a String Catalog. Verify: `Localizable.xcstrings` exists, and the build has no stale or missing-string warnings.
- [ ] **C29.** A privacy manifest is present and covers UserDefaults. Verify: `PrivacyInfo.xcprivacy` is in the app target resources and contains `NSPrivacyAccessedAPICategoryUserDefaults` with `CA92.1`.
- [ ] **C30.** The minimum deployment target is at least the company floor. Verify: `IPHONEOS_DEPLOYMENT_TARGET` >= the agreed value, and the generator rejects lower values.
- [ ] **C31.** The project builds with the current Xcode. Verify: CI uses Xcode 26 or later (App Store requirement since 2026-04-28) and the README states it.

**Logging**
- [ ] **C32.** No `print` in app code; logging goes through `os.Logger` categories. Verify: `rg '\bprint\(' Sources Packages` returns nothing.

**CI and release**
- [ ] **C33.** fastlane is pinned. Verify: `Gemfile.lock` is tracked.
- [ ] **C34.** Release automation authenticates with an App Store Connect API key, not an Apple ID. Verify: `rg apple_id fastlane/` returns nothing, and `app_store_connect_api_key` is used.
- [ ] **C35.** CI fetches signing read-only. Verify: match is called with `readonly: is_ci`, or cloud signing is used.
- [ ] **C36.** A CI workflow runs lint, unit and UI tests on every PR. Verify: a workflow file exists (`.github/workflows/*.yml` or `ci_scripts/`) and its triggers include `pull_request`.

**Agent files**
- [ ] **C37.** Agent instructions are short and tool-neutral. Verify: `AGENTS.md` exists, `CLAUDE.md` imports it, and both are under 200 lines (`wc -l`).
- [ ] **C38.** Every command in AGENTS.md is executed successfully by the generator's own self-test. Verify: a script runs each command in a freshly generated project and exits 0.

---

## Open decisions (for the grilling session)

1. **Architecture pattern.**
   - *MVVM with `@Observable`:* familiar and testable, with some boilerplate.
   - *MV (models + environment, no view models):* less code, closest to Apple samples, but harder to unit-test view logic.
   - *TCA:* strongest testing and structure, but a steep learning curve and third-party lock-in. 2.0 is in a paid-tier beta.
2. **Minimum iOS version.**
   - *17:* Observation, debuggable with Xcode 27, about 90%+ reach (inference from the adoption data).
   - *18:* adds `Mutex` and newer SwiftUI APIs.
   - *26:* Liquid Glass APIs without availability checks, about 79–86% of iPhones per Apple's June 2026 data.
   - Tradeoff: reach vs `#available` noise.
3. **Design system.**
   - *M3 token names backed by iOS asset colors:* Android parity and a shared vocabulary with designers.
   - *Apple semantic colors plus brand accents:* platform-native, least friction with Liquid Glass.
   - *Full M3 visuals:* not recommended. Google no longer supports Material on iOS, and it conflicts with Liquid Glass.
   - Depends on how designers work in Figma.
4. **Project generation.**
   - *XcodeGen with synced folders:* the current investment, simple, slower upstream.
   - *Tuist:* modular graphs and caching; caching is paid, and the toolchain is heavier.
   - *Plain Xcode with buildable folders plus local SPM packages:* zero tools; more manual project-settings drift.
5. **Modularization depth.**
   - Single app target.
   - App plus one Core package.
   - Full feature packages.
   - Tradeoff: build-time isolation and boundaries vs ceremony.
6. **Environment modeling.** Configurations plus schemes on one target (recommended by most guides) vs separate targets (independent icons and entitlements are easier). How many environments: 2 or 3?
7. **Networking approach.**
   - Hand-rolled `Request<Response>` client.
   - Keep the enum `Endpoint` style, made Sendable.
   - swift-openapi-generator: requires the backend to publish an accurate OpenAPI spec.
8. **DI approach.** Plain init plus `@Entry` (no dependency) vs swift-dependencies (test and preview overrides, Point-Free ecosystem) vs Factory (container, property wrappers).
9. **Formatter and linter.** swift-format alone (toolchain-bundled) vs SwiftFormat plus SwiftLint (richer rules, two third-party tools). Also: run at build time vs only in pre-commit and CI.
10. **CI platform.** GitHub Actions (flexible, paid macOS minutes) vs Xcode Cloud (25 h/month included, less flexible) vs Bitrise/Codemagic (not researched).
11. **Signing.** fastlane match (git, GCS or S3 storage) vs Xcode cloud signing with an API key (fewer moving parts, flakier on clean CI per forum reports).
12. **Keep fastlane?** Keep it (mature, volunteer-maintained) vs plain `xcodebuild` and `altool` scripts (fewer dependencies, more code you own).
13. **Snapshot testing.** Include pointfreeco/swift-snapshot-testing by default or not. The tradeoff is regression safety vs baseline churn on OS and Xcode updates.
14. **Crash reporting.** MetricKit only vs Sentry vs Crashlytics. Each third-party SDK adds privacy-manifest obligations.
15. **Swift 5 option.** Drop it (recommended) or keep it for legacy ports.
16. **Generator ownership and license.** Fork and maintain the GPL-3.0 generator, or rewrite the templates in-house. This needs legal input.
17. **UIKit policy.** Keep "no UIKit" as an absolute rule, or allow targeted UIKit interop (`UIViewRepresentable`, appearance APIs) with a stated rule.

---

## Sources

Primary (Apple, swift.org, Swift Evolution, official repos and docs):
- https://swift.org/blog/swift-6.4-released/
- https://developer.apple.com/documentation/xcode-release-notes/xcode-27-release-notes
- https://developer.apple.com/documentation/xcode-release-notes/xcode-26-release-notes
- https://developer.apple.com/documentation/xcode-release-notes/xcode-16-release-notes
- https://developer.apple.com/support/xcode/
- https://www.swift.org/migration/documentation/migrationguide/
- https://github.com/swiftlang/swift-evolution/blob/main/proposals/0466-control-default-actor-isolation.md
- https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md
- https://github.com/swiftlang/swift-evolution/blob/main/proposals/0470-isolated-conformances.md
- https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md
- https://developer.apple.com/videos/play/wwdc2025/268/ (Embracing Swift concurrency)
- https://developer.apple.com/forums/thread/793791 (Approachable Concurrency, Apple DTS)
- https://developer.apple.com/videos/play/wwdc2021/10095/ (async/await with URLSession)
- https://developer.apple.com/documentation/foundation/urlsession/data(for:delegate:)
- https://developer.apple.com/documentation/foundation/urlsessiontaskdelegate/urlsession(_:didcreatetask:)
- https://github.com/apple/swift-openapi-generator/releases/tag/1.0.0
- https://github.com/apple/swift-openapi-generator/releases
- https://github.com/apple/swift-openapi-urlsession
- https://github.com/swiftlang/swift-testing
- https://forums.swift.org/t/an-update-on-swift-testing-progress-and-stable-release-plans/71455
- https://developer.apple.com/videos/play/wwdc2025/245/
- https://developer.apple.com/videos/play/wwdc2026/262/
- https://developer.apple.com/videos/play/wwdc2026/267/
- https://developer.apple.com/xcode/swift-testing/
- https://developer.apple.com/documentation/testing/migratingfromxctest
- https://developer.apple.com/documentation/testing/trait/timelimit(_:)
- https://developer.apple.com/documentation/synchronization/mutex
- https://developer.apple.com/documentation/os/osallocatedunfairlock
- https://developer.apple.com/documentation/xcode/organizing-tests-to-improve-feedback
- https://github.com/pointfreeco/swift-snapshot-testing
- https://github.com/yonaskolb/XcodeGen/releases/tag/2.46.0
- https://raw.githubusercontent.com/yonaskolb/XcodeGen/master/CHANGELOG.md
- https://github.com/tuist/tuist/releases
- https://tuist.dev/en/docs/guides/features/projects
- https://community.tuist.dev/t/support-for-buildable-folders-in-tuist-4-62-0/723
- https://github.com/pointfreeco/isowords
- https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro
- https://developer.apple.com/videos/play/wwdc2023/10149/
- https://developer.apple.com/documentation/swiftui/migrating-to-new-navigation-types
- https://developer.apple.com/documentation/swiftui/navigationstack
- https://developer.apple.com/documentation/swiftui/entry()
- https://github.com/pointfreeco/swift-dependencies
- https://github.com/hmlongco/Factory
- https://github.com/realm/SwiftLint/releases/tag/0.65.1
- https://github.com/SimplyDanny/SwiftLintPlugins
- https://github.com/nicklockwood/SwiftFormat
- https://github.com/swiftlang/swift-format
- https://www.swift.org/documentation/api-design-guidelines/
- https://developer.apple.com/documentation/xcode/adding-a-build-configuration-file-to-your-project
- https://developer.apple.com/design/human-interface-guidelines/color
- https://developer.apple.com/design/human-interface-guidelines/typography
- https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass
- https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)
- https://developer.apple.com/documentation/bundleresources/information-property-list/uidesignrequirescompatibility
- https://github.com/material-components/material-components-ios
- https://developer.apple.com/documentation/xcode/localizing-and-varying-text-with-a-string-catalog
- https://developer.apple.com/videos/play/wwdc2025/225/
- https://developer.apple.com/help/app-store-connect/manage-app-accessibility/overview-of-accessibility-nutrition-labels
- https://developer.apple.com/documentation/xcuiautomation/xcuiapplication/performaccessibilityaudit(for:_:)
- https://developer.apple.com/videos/play/wwdc2023/10035/
- https://developer.apple.com/documentation/bundleresources/describing-use-of-required-reason-api
- https://developer.apple.com/documentation/bundleresources/app-privacy-configuration/nsprivacyaccessedapitypes/nsprivacyaccessedapitype
- https://developer.apple.com/documentation/bundleresources/privacy-manifest-files
- https://developer.apple.com/support/third-party-SDK-requirements/
- https://developer.apple.com/news/upcoming-requirements/
- https://developer.apple.com/support/app-store/
- https://developer.apple.com/videos/play/wwdc2020/10168/
- https://developer.apple.com/videos/play/wwdc2023/10226/
- https://developer.apple.com/documentation/os/ossignposter
- https://developer.apple.com/documentation/metrickit/mxmetricmanager
- https://developer.apple.com/documentation/metrickit/metricmanager
- https://github.com/apple/swift-log
- https://github.com/fastlane/fastlane/releases
- https://opencollective.com/fastlane/updates/monthly-releases
- https://docs.fastlane.tools/getting-started/ios/setup/
- https://docs.fastlane.tools/app-store-connect-api/
- https://docs.fastlane.tools/actions/match/
- https://developer.apple.com/help/app-store-connect/manage-builds/upload-builds
- https://developer.apple.com/xcode-cloud/
- https://github.com/actions/runner-images
- https://agents.md/
- https://code.claude.com/docs/en/memory
- https://developer.apple.com/documentation/xcode/giving-external-agents-access-to-xcode
- https://developer.apple.com/videos/play/wwdc2026/259/
- https://github.com/getsentry/XcodeBuildMCP

Community / secondary (opinion signals):
- https://www.hackingwithswift.com/forums/swift/new-xcode-16-4-projects-default-to-swift-5-why/29888
- https://www.donnywals.com/should-you-opt-in-to-swift-6-2s-main-actor-isolation/
- https://fatbobman.com/en/posts/default-actor-isolation/
- https://www.donnywals.com/building-a-token-refresh-flow-with-async-await-and-swift-concurrency/
- https://blog.jacobstechtavern.com/p/advanced-swift-actors-re-entrancy
- https://github.com/Moya/Moya/releases
- https://swiftpackageindex.com/Moya/Moya
- https://github.com/kean/Get
- https://pointfree.co/blog/posts/146-swift-testing-support-for-snapshottesting
- https://pointfree.co/blog/posts/206-beta-preview-composablearchitecture-2-0
- https://dimillian.medium.com/swiftui-in-2025-forget-mvvm-262ff2bbd2ed
- https://developer.apple.com/forums/thread/710501
- https://techconcepts.org/blog/swiftui-navigation-2026
- https://swiftwithmajid.com/2022/01/12/microapps-architecture-in-swift-spm-basics/
- https://swiftwithmajid.com/2025/05/27/microapps-architecture-in-swift-scaling/
- https://pepicrft.me/blog/cache-for-any-xcode-project/
- https://tuist.dev/blog/2025/03/21/git-conflicts
- https://troz.net/post/2024/swift_format/
- https://nshipster.com/xcconfig
- https://robots.thoughtbot.com/let-s-setup-your-ios-environments
- https://developer.apple.com/forums/thread/805449
- https://www.hackingwithswift.com/interview-questions/what-do-you-think-is-a-sensible-minimum-ios-deployment-target
- https://telemetrydeck.com/survey/apple/iOS/majorSystemVersions/
- https://developer.apple.com/forums/thread/764554
