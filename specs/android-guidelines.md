# Spec: Android guidelines skill (agent-agnostic)

**Date:** 2026-10-19 · **Status:** Settled (grilling session)
**Context:** A portable Agent Skill that tells coding agents how to write and review Android (Kotlin, Jetpack Compose) code in any project Jaim works on, including client projects. It is derived from the gap analysis of EdwinLovo/android-claude-toolkit (`research/android/2026-10-android-toolkit-gap-analysis.md`), used as inspiration only: the toolkit has no LICENSE, so no text or code is copied from it.

## Decisions

### Scope and strength (carried over from `specs/ios-guidelines.md`)
- **Two strengths of rule.** *Must* rules apply to every project. *Default* rules apply only when the project has no established convention for that area, and each default says so in its own text.
- **Project conventions win over defaults.** Why: client projects (e.g. one already on the toolkit's conventions) keep their patterns.
- **Infra is follow-only.** CI, signing, Play publishing and crash reporting: the agent follows what the project has and never introduces or replaces it.
- **Delivery:** one Agent Skill, `android/SKILL.md` (musts, under 150 lines) plus `android/references/*.md` (defaults). Nothing Claude-only: no path-scoped `.claude/rules`, no commands, no `context: fork`, no placeholder `sed`.
- **No global install.** Copied into a project when needed, as with iOS.
- **No copied text.** Rules are written from primary sources.

### Toolchain and SDK levels
- **Default:** minSdk 26; compileSdk and targetSdk at the latest stable API (36 today). Latest stable Kotlin and AGP. Why: `java.time` without desugaring and no compat noise, matching MyWalkup.
- **Must:** never change Kotlin, AGP, Gradle, minSdk, compileSdk or targetSdk without asking. Raising targetSdk handles that level's behavior changes in the same change.
- **Must:** dependencies live in the version catalog, never hard-coded in module build files.

### Module structure
- **Default:** new projects start as a single `:app` module. Why: some apps are small and don't need modules.
- **Growth path:** `:app` + `:core` + `:data`.
  - `:core`: domain models, repository interfaces, use cases, design system, utilities. No network or database dependencies.
  - `:data`: network client, DTOs, mappers, repository implementations, local storage, data-layer Hilt bindings. Depends on `:core`.
  - `:app`: UI layer (screens, ViewModels, navigation, Application). Depends on both.
- **When:** the agent proposes the split only when asked or when a second module is needed anyway (widget, Wear, shared SDK). Convention plugins in `build-logic/` arrive with the split, not before.
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
- **Default:** `SavedStateHandle` for small input that must survive process death; navigation arguments come from the route key.
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

### Data layer and networking
- **Default:** Retrofit 3 + OkHttp + kotlinx.serialization converter. Apollo Kotlin for GraphQL, Ktor only for KMP. Existing projects keep their library.
- **Default:** repositories call the service directly. A data source class is added only when coordinating several sources (remote + local). Offline-first when the app has a local cache: the local store is the source of truth.
- **Default:** repositories return a sealed result, e.g. `ApiResult<T>` = `Success(data)` / `Failure(error: DataError)`, with `DataError` a sealed type (Network, Unauthorized, Server(code), Serialization, Unknown). No `Loading` case. Composition over a `BaseRepository` base class.
- **Must:** never swallow `CancellationException`; serialization failures are caught and typed; a 204 is a success; raw `e.message` or `errorBody()` text never reaches users (the UI maps `DataError` to string resources).
- **Must:** DTOs are `@Serializable` data-layer types mapped to domain models, never in domain or UI signatures. No Gson in new code.
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
- **Must:** lazy list items pass a stable `key`.
- **Must:** lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`, and effects calling lambda params use `rememberUpdatedState`.
- **Default:** small `private` helper composables may share a file; a composable used from two places gets its own file and becomes `internal`. No one-component-per-file must.

### Previews
- **Default:** previews required for reusable components and screen content, `private`, at the bottom of the file.
- **Default:** `@PreviewLightDark` plus `@PreviewFontScales` on screen content, or the project's wrapper annotation.
- **Default:** `PreviewParameterProvider` only for real state variants, in the same file unless the project uses sibling files.

### Theme and design system
- **No default design system.** Tokens come from the client's Figma.
- **Default:** extend MaterialTheme: map the Figma palette onto M3 `ColorScheme`, typography and shapes; extra tokens (spacing, extended colors) in a `staticCompositionLocalOf` of `@Immutable` classes behind an `AppTheme.*` accessor. `MaterialTheme.colorScheme` stays allowed. A fully custom theme object only when the design isn't Material-based.
- **Must:** no literal colors (`Color(0x…)`, `Color.Red`) or hard-coded `.dp` design values in composables; values come from theme tokens. Repeated values become tokens.
- **Default:** dynamic color is a per-project decision; no Material 3 Expressive APIs unless the project is on material3 1.5+ and opts in.

### Icons
- **Default:** Material Symbols vector drawables referenced as `R.drawable.ic_*`. A project's existing icon registry object is kept.
- **Must:** no `androidx.compose.material.icons` (`Icons.*`) in new code.

### Strings, localization and accessibility
- **Must:** no hard-coded user-visible strings; plurals via `pluralStringResource`; whole sentences with placeholders, no concatenation.
- **Must:** numbers, currency and dates formatted locale-aware.
- **Must:** touch targets at least 48dp; every icon-only control has a `contentDescription` from resources (or `null` when decorative) or a click label.
- **Default:** per-app language support via `generateLocaleConfig` in apps with more than one locale.

### Platform musts
- **Must:** edge-to-edge enabled (`enableEdgeToEdge()`), insets applied once, never `systemBarsPadding()` on a NavHost or root.
- **Must:** back handling only through `BackHandler`/`PredictiveBackHandler` or navigation; no `onBackPressed` overrides.
- **Must:** never add `screenOrientation` locks or `resizeableActivity="false"` without asking; when a client wants a lock, phones only. Layouts adapt through window size classes, never device-type checks.
- **Must:** UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
- **Must:** runtime permissions (including `POST_NOTIFICATIONS` on API 33+) are requested in context with a handled denial.
- **Must:** apps that ship native libraries support 16 KB pages.
- **Must:** a library that uses reflection or serialization comes with verified R8 keep rules and a tested minified release build.

### Environments and configuration
- **Default:** one `environment` flavor dimension with `dev`, `qa`, `prod` product flavors; `applicationIdSuffix` `.dev`/`.qa`; base URLs via `buildConfigField`; app name and launcher icon per flavor in `src/<flavor>/res`; `debug`/`release` build types, R8 on for release.
- **Must:** required config is read once at startup and fails fast with a clear message.
- Existing projects keep their flavor and build-type setup.

### Testing
- **Default:** JUnit 5 for new projects; existing JUnit 4 suites stay.
- **Default:** hand-written fakes for types we own; MockK only for hard-to-fake third-party types.
- **Default:** `runTest`, a main-dispatcher rule or extension, Turbine for genuine streams. `StateFlow` assertions check the final `uiState.value` or latest item, never an intermediate emission (conflation).
- **Default:** backtick sentence names in JVM tests; `subject_condition_expected` in `androidTest`.
- **Default:** one Compose UI test per main screen against `ScreenContent`, with `enableAccessibilityChecks()`.
- **Default:** screenshot tests off; Roborazzi when a project opts in.
- **Must:** test doubles live in test source sets, never in `main`. `runTest`, not `runBlocking`.

### Tooling
- **Default:** ktlint 1.x plus Android Lint with Slack's `compose-lints`. No Detekt until 2.0 is stable.
- **Must:** before finishing, run the project's formatter, linter and static analysis plus the affected unit tests, and fix findings. Never disable, suppress or baseline a finding to get a clean run.

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
- Unverified research claims, recheck before relying on them: the scope of the Navigation "maintenance mode" caution, Android 17 not recreating activities on configuration changes, the 16 KB deadline dates, the old `hiltViewModel` import's deprecation status, and the minSdk 26 device share.
