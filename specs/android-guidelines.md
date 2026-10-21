# Spec: Android guidelines skill (agent-agnostic)

**Date:** 2026-10-19 · **Status:** Settled (grilling session)
**Context:** A portable Agent Skill that tells coding agents how to write and review Android (Kotlin, Jetpack Compose) code in any project Jaim works on, including client projects. It is derived from the gap analysis of EdwinLovo/android-claude-toolkit (`research/android/2026-10-android-toolkit-gap-analysis.md`), used as inspiration only: the toolkit has no LICENSE, so no text or code is copied from it.
**Revised 2026-10-19** against Google's official skills (github.com/android/skills @ `ed0a272`, Apache-2.0): JUnit 4 default, AGP 9 build rules, Navigation 3 and back-handling updates, security and permissions, edge-to-edge and Compose performance gaps. Rules are still written in our own words, citing the skill path as the source.
**Revised 2026-10-21** after an independent review of seven rules against current Android, Compose, Play and library sources (`research/android/portable-rules-evaluation-review.md`, `…-verdict.md`): stability annotations, `rememberUpdatedState`, root insets, Photo Picker, consumer keep rules, version catalog and serializer rules narrowed or reclassified.

## Decisions

### Scope and strength (carried over from `specs/ios-guidelines.md`)
- **Two strengths of rule.** *Must* rules apply to every project. *Default* rules apply only when the project has no established convention for that area, and each default says so in its own text.
- **Project conventions win over defaults.** Why: client projects (e.g. one already on the toolkit's conventions) keep their patterns.
- **Infra is follow-only.** CI, signing, Play publishing and crash reporting: the agent follows what the project has and never introduces or replaces it.
- **Delivery:** one Agent Skill, `android/SKILL.md` (musts, under 150 lines) plus `android/references/*.md` (defaults). Nothing Claude-only: no path-scoped `.claude/rules`, no commands, no `context: fork`, no placeholder `sed`.
- **No global install.** Copied into a project when needed, as with iOS.
- **No copied text.** Rules are written from primary sources.

### Toolchain and SDK levels
- **Default:** minSdk 26; compileSdk and targetSdk at the latest stable API (37 since Android 17 went stable on 2026-06-16; the skill says to check rather than naming a number). Latest stable Kotlin and AGP. Why: `java.time` without desugaring and no compat noise, matching MyWalkup.
- **Must:** never change Kotlin, KSP, AGP, Gradle, minSdk, compileSdk or targetSdk without asking, including a Kotlin version pinned through the `buildscript` classpath. Raising targetSdk handles that level's behavior changes in the same change.
- **Must (AGP 9+):** use AGP's built-in Kotlin. New modules don't apply `org.jetbrains.kotlin.android`. Use KSP; `com.android.legacy-kapt` only for a processor without KSP support. Compiler options go in `kotlin { compilerOptions }`, not `android.kotlinOptions`. Never add the opt-out flags `android.builtInKotlin=false`, `android.newDsl=false` or `android.disallowKotlinSourceSets=false`. Verify: grep build files and `gradle.properties`.
- **Must:** declare dependencies through the project's existing mechanism; never hard-code versions in module build files when the project centralizes them elsewhere.
- **Default:** new projects use a version catalog.

### Module structure
- **Default:** new projects start as a single `:app` module. Why: some apps are small and don't need modules.
- **Growth path:** `:app` + `:core` + `:data`.
  - `:core`: domain models, repository interfaces, use cases, design system, utilities. No network or database dependencies.
  - `:data`: network client, DTOs, mappers, repository implementations, local storage, data-layer Hilt bindings. Depends on `:core`.
  - `:app`: UI layer (screens, ViewModels, navigation, Application). Depends on both.
- **When:** the agent proposes the split only when asked or when a second module is needed anyway (widget, Wear, shared SDK). Convention plugins in `build-logic/` arrive with the split, not before. They use only public `com.android.build.api.*` types and `androidComponents.onVariants`/`beforeVariants`; `applicationVariants` and `variantFilter` are gone in AGP 9.
- **Must:** new code goes in the module the structure implies; never create a module without asking.

### Screen file layout
- **Default:** package by feature. Per screen:
  - `<Screen>Route.kt` (`@Serializable` key)
  - `<Screen>Contract.kt` (`<Screen>UiState` + `sealed interface <Screen>Event`)
  - `<Screen>ViewModel.kt`
  - `<Screen>Screen.kt` (entry composable + `internal` `<Screen>ScreenContent` + `private` previews)
  - `components/` only when the feature has its own components; `delegates/` only per the delegates rule.
- No `contracts/` subfolder. UI models live in the UI layer, not in `domain/`.

### Architecture and state
- **Default:** MVVM with UDF. One immutable `<Screen>UiState` data class with defaults per screen.
- **Default:** events go through `handleEvent(event)` over a `sealed interface <Screen>Event`, exhaustive `when`; `ScreenContent` takes a single `onEvent` lambda. Why: the team's existing style, uniform for agents, compile-checked.
- **Default:** explicit backing fields on Kotlin 2.4+ (stable, no flag); below 2.4, `private val _uiState` + `asStateFlow()`. Never add an experimental compiler flag or upgrade Kotlin for this without asking.
- **Default:** updates use `MutableStateFlow.update { it.copy(...) }`; no custom `reduce` extension. Derived state uses `stateIn(viewModelScope, WhileSubscribed(5_000), initial)`.
- **Default:** prefer `stateIn` or an idempotent load over async work in `init`.
- **Default:** `SavedStateHandle` for small input that must survive process death; navigation arguments come from the route key. On Navigation 3 the key does not reach `SavedStateHandle`: pass it through an assisted Hilt factory (`hiltViewModel(creationCallback = { it.create(key) })`).
- **Must:** `MutableStateFlow`/`MutableState` never public. ViewModels take no `Context`, `Activity` or `Resources` and are not `AndroidViewModel`.
- **Must:** ViewModels don't send one-off events through `Channel` or `SharedFlow`; outcomes are UI state that the UI clears.

### Delegates
- **Optional pattern**, allowed only for a stateful sub-flow (own loading/error/data state) used where a use case alone would duplicate state handling.
  - Injected into the ViewModel, never exposed to the UI.
  - The ViewModel combines their state into its `uiState`.
  - No `init(scope)`: the scope is passed through a method that returns the state, or the delegate is a plain class the ViewModel creates.
- Stateless shared logic goes into use cases instead.

### Use cases
- **Default:** only when logic is non-trivial or reused across ViewModels. `operator fun invoke`, stateless, main-safe, may call other use cases. Domain code has no Android framework or UI imports.

### Messages and errors in the UI
- **Default:** per-screen errors are state: `uiState.userMessage: UiText?`, shown by the screen, cleared with a `MessageShown` event.
- **Default:** app-wide messages (session expired, offline) go through one injected `@Singleton` holder exposing `StateFlow<List<UserMessage>>`, drained by the root composable with show-then-mark-shown. No Channel, no global `object` bus.
- **Default:** `UiText` (string resource with format args, or dynamic string) carries text produced outside composables.

### Navigation
- **Default:** Navigation 3 for new Compose projects. Existing projects keep their library (Navigation Compose 2.x, Fragment/XML); never migrate without asking.
- **Must:** routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x); no string routes in new code.
- **Must:** screens expose navigation callbacks (`onBack`, `onOpenDetail: (Id) -> Unit`); the nav graph wires them. Composables never receive `NavController` or the back stack.
- **Default:** navigation that depends on logic is a UI-state flag the screen observes, acts on through its callback, and clears. No `ViewModelNav`/`NavigationAction` delegate as a default.
- **Must (Navigation 3):** `NavDisplay` installs `rememberSaveableStateHolderNavEntryDecorator()` and `rememberViewModelStoreNavEntryDecorator()`, so ViewModels and saved state are scoped per entry.
- **Default:** results returned to a previous screen travel as state (Google's results-as-state recipe), not through the in-memory `ResultEventBus`, which loses results on process death.
- **Default:** top-level destinations use `NavigationSuiteScaffold`. On Navigation 3, multi-pane layouts use `ListDetailSceneStrategy`/`SupportingPaneSceneStrategy` with entry metadata, not `ListDetailPaneScaffold`.

### Data layer and networking
- **Default:** Retrofit 3 + OkHttp + kotlinx.serialization converter. Apollo Kotlin for GraphQL, Ktor only for KMP. Existing projects keep their networking library and serializer.
- **Default:** repositories call the service directly. A data source class is added only when coordinating several sources (remote + local). Offline-first when the app has a local cache: the local store is the source of truth.
- **Default:** repositories return a sealed result, e.g. `ApiResult<T>` = `Success(data)` / `Failure(error: DataError)`, with `DataError` a sealed type (Network, Unauthorized, Server(code), Serialization, Unknown). No `Loading` case. Composition over a `BaseRepository` base class.
- **Must:** never swallow `CancellationException`; serialization failures are caught and typed; a 204 is a success; raw `e.message` or `errorBody()` text never reaches users (the UI maps `DataError` to string resources).
- **Must:** DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer; never introduce a second serializer or migrate without asking.
- **Must:** no hard-coded base URLs or keys in source; no secrets in source or committed `gradle.properties`.

### Dependency injection
- **Default:** Hilt. Constructor injection, `@Binds` for interfaces, `@Provides` for builders and third-party types, custom `@Qualifier` over `@Named`, scopes only for shared state. Hilt modules sit next to what they bind.
- **Must:** never introduce a second DI library or migrate away from Hilt without asking. A project already on another DI library keeps it.
- **Audit:** flag composables that resolve dependencies directly.

### Concurrency
- **Must:** dispatchers are injected, not hard-coded outside DI modules. No `GlobalScope`.
- **Default:** combine flows with `combine`/`stateIn`, `flatMapLatest` for query-driven loads, no `runBlocking` outside `main` and tests.

### Compose UI
- **Default:** stateful entry composable + stateless `internal` `<Screen>ScreenContent`; the ViewModel exists only at screen level.
- **Must:** collect with `collectAsStateWithLifecycle()`.
- **Must:** composables below the screen entry receive state and lambdas, never a ViewModel, `NavController` or back stack.
- **Must:** reusable UI composables take `modifier: Modifier = Modifier` as the first optional parameter and apply it to the root.
- **Must:** lazy list items pass a stable `key` and, for mixed item types, a `contentType`.
- **Must:** lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. For keyed effects, a lambda parameter is read through `rememberUpdatedState` when changing it must not restart the effect, and used as a key when changing it should.
- **Must:** with strong skipping (default since Kotlin 2.0.20), don't add `@Immutable`, `@Stable` or `ImmutableList` to collection-bearing models; preserve collection instances instead. Why: making an unstable parameter stable switches instance comparison to `equals`, which can walk the collection whenever a different instance arrives. Exception only when the stability contract is valid, the reference cannot be preserved and benchmarks show structural comparison beats recomposition. Existing annotations are not changed merely to comply. Collection-free classes may keep valid `@Immutable`.
- **Must:** custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.
- **Must:** no experimental Compose APIs requiring an opt-in, alpha Compose APIs, new `@OptIn` annotations or module-wide opt-in flags without asking. Check the project's Compose version before using an API that needs a newer one.
- **Default:** small `private` helper composables may share a file; a composable used from two places gets its own file and becomes `internal`. No one-component-per-file must.

### Previews
- **Default:** previews required for reusable components and screen content, `private`, at the bottom of the file.
- **Default:** `@PreviewLightDark` plus `@PreviewFontScales` on screen content, or the project's wrapper annotation.
- **Default:** `PreviewParameterProvider` only for real state variants, in the same file unless the project uses sibling files.

### Theme and design system
- **No default design system.** Tokens come from the client's Figma.
- **Default:** extend MaterialTheme: map the Figma palette onto M3 `ColorScheme`, typography and shapes; extra tokens (spacing, extended colors) in a `staticCompositionLocalOf` of `@Immutable` classes behind an `AppTheme.*` accessor. `MaterialTheme.colorScheme` stays allowed. A fully custom theme object only when the design isn't Material-based.
- **Must:** no literal colors (`Color(0x…)`, `Color.Red`) or hard-coded `.dp` design values in composables; values come from theme tokens. Repeated values become tokens.
- **Default:** dynamic color is a per-project decision. Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).

### Icons
- **Default:** Material Symbols vector drawables referenced as `R.drawable.ic_*`. A project's existing icon registry object is kept.
- **Must:** no `androidx.compose.material.icons` (`Icons.*`) in new code.

### Strings, localization and accessibility
- **Must:** no hard-coded user-visible strings; plurals via `pluralStringResource`; whole sentences with placeholders, no concatenation.
- **Must:** numbers, currency and dates formatted locale-aware.
- **Must:** touch targets at least 48dp; every icon-only control has a `contentDescription` from resources (or `null` when decorative) or a click label.
- **Default:** per-app language support via `generateLocaleConfig` in apps with more than one locale.

### Platform musts
- **Must:** edge-to-edge enabled (`enableEdgeToEdge()`), each inset applied once. When screens should draw behind system bars, no system-bar padding on the whole NavHost or root; insets are applied per screen or component. Root padding is valid when keeping all content within safe bounds is intentional.
- **Must:** lists take insets through `contentPadding`, never `Modifier.padding`. `NavigationSuiteScaffold` and `ListDetailPaneScaffold` don't pass insets down: each screen applies its own, never `safeDrawingPadding` on the scaffold.
- **Must:** IME insets applied exactly once: `adjustResize` in the manifest, `imePadding()` before `verticalScroll`, and no `imePadding()` under a Scaffold whose `contentWindowInsets` already include the IME.
- **Must:** back handling only through navigation (Navigation 3's built-in back on Nav3), `NavigationBackHandler` (`androidx.navigationevent`), or `BackHandler`/`PredictiveBackHandler` where the project already uses them. No `onBackPressed` overrides; never set `enableOnBackInvokedCallback="false"`.
- **Default:** `NavigationBackHandler` for new back handling outside navigation.
- **Must:** never add `screenOrientation` locks or `resizeableActivity="false"` without asking; when a client wants a lock, phones only. Layouts adapt through window size classes, never device-type checks.
- **Must:** UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
- **Must:** apps that ship native libraries support 16 KB pages.

### Security and permissions
- **Must:** runtime permissions (including `POST_NOTIFICATIONS` on API 33+) are requested in context with a handled denial: show a rationale when `shouldShowRequestPermissionRationale` says so, send the user to app settings after a permanent denial, never loop requests or cache grant state. Background location is a separate request after foreground is granted.
- **Must:** the Photo Picker (`PickVisualMedia`) for user-selected photos and videos. `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only when broad library access is essential to core functionality and system pickers are insufficient, with Play's declaration and review requirements met. A permission-based gallery on Android 14+ targeting API 34+ handles partial access and reselection with `READ_MEDIA_VISUAL_USER_SELECTED`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`. Verify: review manifest grep hits.
- **Must:** components are `exported="false"` unless another app needs them. `PendingIntent`s use `FLAG_IMMUTABLE`; a mutable one gets an explicit component. Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.
- **Must:** never launch an Intent taken from extras without validating it (`IntentSanitizer` or package/exported checks). `onNewIntent` calls `setIntent` and validates like `onCreate`.

### R8
- **Must:** a library that uses reflection or serialization comes with verified R8 keep rules and a tested minified release build.
- **Must:** keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. Don't duplicate consumer rules the exact artifact/version already bundles (check `META-INF/proguard/`, `META-INF/com.android.tools/`, `proguard.txt`); add only missing, verified narrow rules. AndroidX, kotlinx.coroutines, kotlinx.serialization (except named companions), Retrofit 2.10+ and Gson 2.11+ bundle theirs. No rules for Activity and View classes. Under full mode, `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.

### Environments and configuration
- **Default:** one `environment` flavor dimension (`dimension = "environment"`; `setDimension` is gone in AGP 9) with `dev`, `qa`, `prod` product flavors; `applicationIdSuffix` `.dev`/`.qa`; base URLs via `buildConfigField` (needs `buildFeatures { buildConfig = true }`; String values carry escaped quotes); app name and launcher icon per flavor in `src/<flavor>/res`; `debug`/`release` build types. Release sets `isMinifyEnabled`, `isShrinkResources` and `proguard-android-optimize.txt` explicitly (AGP 9 rejects `proguard-android.txt`).
- **Must:** required config is read once at startup and fails fast with a clear message.
- Existing projects keep their flavor and build-type setup.

### Testing
- **Default:** JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 forces the vintage engine and two test styles. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
- **Default:** hand-written fakes for types we own; MockK only for hard-to-fake third-party types.
- **Default:** `runTest`, a main-dispatcher rule or extension, Turbine for genuine streams. `StateFlow` assertions check the final `uiState.value` or latest item, never an intermediate emission (conflation).
- **Default:** backtick sentence names in JVM tests (Robolectric included); `subject_condition_expected` in `androidTest`.
- **Default:** one Compose UI test per main screen against `ScreenContent`, with `enableAccessibilityChecks()`, in `test` on Robolectric unless it needs a device. Find nodes by semantics (text, role, content description); `testTag` only when semantics can't single the node out. Screens with saved state get a `StateRestorationTester` check. `HiltAndroidRule` is `order = 0`.
- **Default:** screenshot tests off. When a project opts in: Roborazzi (1.56.0+ on AGP 9), or Compose Preview Screenshot Testing (experimental; AGP 9.0+ for IDE support). Never upgrade AGP for it. On AGP 9, Paparazzi only from 2.0.0-alpha05.1.
- **Must:** test doubles live in test source sets, never in `main`. `runTest`, not `runBlocking`.

### Tooling
- **Default:** ktlint 1.x plus Android Lint with Slack's `compose-lints`. No Detekt until 2.0 is stable.
- **Must:** before finishing, run the project's formatter, linter and static analysis plus the affected unit tests, and fix findings. Never disable, suppress or baseline a finding to get a clean run. Never run `clean` to verify; `./gradlew help` or `--dry-run` checks configuration cheaply.

### Logging and crash reporting
- **Default:** Timber. `DebugTree` in debug only; release forwards warnings and errors to the crash reporter or logs nothing.
- **Must:** no `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.
- **No crash-reporting default.** Follow the project's SDK.

## Acceptance criteria
1. `rules/android/SKILL.md` has valid Agent Skill frontmatter (`name: android-guidelines` and a quoted `description` that names Kotlin/Android/Compose work).
2. `SKILL.md` lists every **must** rule above, states the must/default distinction and the "project conventions win" clause, indexes the reference files, and stays under 150 lines.
3. Every **default** rule lives in a reference file, and each one's text includes "unless the project already uses X" or equivalent.
4. Traceability: every decision in this spec maps to at least one rule. Check with a table or grep in the slice PR description.
5. No text or code is copied from EdwinLovo/android-claude-toolkit, and no client name, package, module or scheme appears in the skill.
6. Each must that the research paired with a check (grep or lint rule) keeps a one-line "Verify:" hint, so an audit can confirm it.
7. Research claims flagged U or S are not stated as fact in the skill unless rechecked against the primary source.

## Slices
Not split. Estimate: about 11 files and 400 lines (SKILL.md plus about 10 reference files), scaled from the iOS skill's measured 9 files and 258 lines. Under the 12-file / 480-line trigger. Covers criteria 1–7.

## Non-goals / Deferred
- Global install, trigger lines and per-tool load checks (dropped for iOS too).
- Changing, forking or upstreaming fixes to EdwinLovo/android-claude-toolkit.
- Migrating client projects (MyWalkup's toolkit-style rules, Nelnet's Fragment/XML stack) toward the defaults.
- A new-project generator, template or scaffolding skills (the toolkit's "primitives").
- Choosing CI, signing, Play publishing or crash-reporting tools.
- Kotlin Multiplatform, Wear, TV, Auto and widgets.
- Detekt, revisited when 2.0 is stable.
- React Native rules: same repo, separate spec.
- Research claims, rechecked against primary sources on 2026-10-19:
  - Latest stable API: 37 (Android 17, stable 2026-06-16; android-developers.googleblog.com/2026/06/Android-17.html). Play requires target API 36 for new apps and updates since 2026-08-31, extendable to 2026-11-01 (developer.android.com/google/play/requirements/target-sdk).
  - 16 KB pages: from 2027-02-01, Play rejects updates without 16 KB support for apps targeting API 35+ on 64-bit devices (developer.android.com/guide/practices/page-sizes). The older 2025/2026 dates are stale.
  - minSdk 26 device share: unconfirmed. Google no longer publishes a version table; use Play Console "Reach and devices" or the Android Studio new-project wizard.
  - Android 17 configuration changes: activities no longer restart for keyboard, navigation, touchscreen and color-mode changes (opt back in with `android:recreateOnConfigChanges`); rotation and resizing still recreate (developer.android.com/guide/topics/resources/runtime-changes). Separately, apps targeting 37 have orientation and resize locks ignored on displays with smallest width ≥ 600dp (developer.android.com/about/versions/17/changes/ff-restrictions-ignored).
  - Navigation "maintenance mode": the caution covers the whole `androidx.navigation` library, Navigation Compose included (developer.android.com/jetpack/androidx/releases/navigation).
  - `hiltViewModel`: Hilt 1.3.0-alpha02 moved it to `androidx.hilt.lifecycle.viewmodel.compose` and deprecated the `androidx.hilt.navigation.compose` one (developer.android.com/jetpack/androidx/releases/hilt; `@Deprecated` in androidx source).
  - Screenshot tools on AGP 9: Compose Preview Screenshot Testing needs AGP 8.5+ (9.0+ for IDE support) and is experimental (developer.android.com/studio/preview/compose-screenshot-testing); Paparazzi supports AGP 9 from 2.0.0-alpha05.1 (cashapp/paparazzi CHANGELOG); Roborazzi from 1.56.0 (takahirom/roborazzi releases).
  - Retrofit's bundled R8 full-mode rules for generic signatures of `Call`, `Response` and service return types first shipped in 2.10.0 (2024-03-18); 2.9.0 (2020-05-20) bundled rules but lacked them (square/retrofit CHANGELOG and `retrofit2.pro` history).
  - Google's stability pages now caution against `@Immutable`/`@Stable`/`ImmutableList` on collection-holding models and allow them only when the instance reference cannot be preserved and benchmarks confirm a gain (developer.android.com/develop/ui/compose/performance/stability/fix, 2026-10-21).
  - Play's Photo and Video Permissions policy covers `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only, allows them when system pickers cannot serve core functionality, and exempts private and enterprise device-management apps (support.google.com/googleplay/android-developer/answer/16935362).
