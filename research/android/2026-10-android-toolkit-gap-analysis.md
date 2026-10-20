# Android Claude toolkit: gap analysis against October 2026 practice

- Researched 2026-10-19.
- Subject: https://github.com/EdwinLovo/android-claude-toolkit at commit `7004bb1` (2026-07-03). It was read-only and nothing in it was executed.
- File references (`.claude/rules/viewmodels.md:12`) are relative to that clone. `CLAUDE.md:N` and `README.md:N` are the root files.
- Context: Kotlin 2.4.21 (2026-10-08) is current, AGP 9.4.0 (September 2026) with Gradle 9.6+ (9.8.2 is latest), Android 17 (API 37) shipped 2026-06-16, and Play requires targetSdk 36 since 2026-08-31. The toolkit is a few weeks newer than Kotlin 2.4.0 but still documents 2.3-era flags.

Source flags:
- **V**: the page was opened and read. Pages were read through a summarizing fetch tool. Spot-check quoted wording before reusing it.
- **S**: seen only in a search snippet or secondary summary.
- **U**: unverified.

Labels:
- "Community" marks opinion from blogs, forums or third-party maintainers, not Google, JetBrains or a library's own docs.
- "Inference" marks my own reasoning from the cited facts.

Tags on recommendations: **KEEP**, **CHANGE**, **DROP**, **OPTIONAL** (team preference, make it a default the project can override), **ADD** (missing entirely).

---

## Summary: top 10 gaps, ranked by impact

1. **No platform musts at all.** There is nothing on edge-to-edge (enforced at targetSdk 35, no opt-out at 36), predictive back (`onBackPressed` no longer called at 36), large-screen orientation and resizability (ignored on sw600dp+ at 36, opt-out gone at 37), runtime permissions, `POST_NOTIFICATIONS`, per-app languages or the Play target-API deadline. The scaffold even applies `Modifier.systemBarsPadding()` to the whole `NavHost` (`.claude/skills/project-scaffold/SKILL.md:266`) and never calls `enableEdgeToEdge()`.
2. **One-off events contradict Google.** A global `object ErrorEventBus` on a `Channel` (`.claude/rules/error-handling.md:34-46`) plus a VM-driven navigation action stream. Google's UI events page lists Channels as an antipattern ("don't guarantee the delivery and processing of those events") and says ViewModel events "should always result in a UI state update". The architecture recommendations say "Do not send events from the ViewModel to the UI" (Strongly recommended).
3. **Navigation is built around passing `NavController` into screens** (`.claude/rules/screens.md:22-34`), which Google's navigation docs warn against in so many words. The toolkit also ignores Navigation 3, which is stable (1.2.0) and is what the architecture recommendations now name for single-activity apps.
4. **The toolkit contradicts itself in several places.** Event types (`interface` vs `sealed interface`), the event bus (`SharedFlow` vs `Channel`), preview provider placement, test naming, the fallback string key and the data-source layer each have two opposite rules. An agent loading both rules will follow either one.
5. **Error handling hides failures and leaks raw messages to users.** `safeCallSuspend` catches only `IOException`/`HttpException`, so serialization errors escape a layer that "never throws". It treats a 204 as an error ("Empty body"), and pushes `e.message` or the raw `errorBody()` into the user-facing dialog through `toUiText()`.
6. **No tooling is configured.** No lint config, Detekt, ktlint/ktfmt or Compose lint rules, and no baseline profiles, R8 keep-rule guidance or CI. The data-feature checklist tells the agent to run `./gradlew ktlintCheck` (`.claude/skills/data-feature/SKILL.md:252`), a task nothing sets up.
7. **Accessibility and adaptive UI are almost absent.** Only `contentDescription` is covered. There is no 48dp touch target, no semantics merging, no font scaling, no `enableAccessibilityChecks()` in tests, and no window size classes. The dialog wrapper hard-codes 62% width on every window (`.claude/skills/component-primitives/SKILL.md:94`).
8. **Testing guidance will produce a flaky or failing ViewModel test.** The template awaits an intermediate `isLoading = true` emission from a `StateFlow` (`.claude/rules/testing.md:55-69`), which Google warns is conflated. There is no Compose UI test pattern beyond one line, and no screenshot testing. `ScreenContent` is `private`, so tests can't reach it.
9. **Single module with no Gradle conventions.** There is no version-catalog content, no convention plugins, no modularization path and no AGP 9 notes (built-in Kotlin, new DSL). Now in Android, the official reference app, uses `build-logic` convention plugins and feature/core modules.
10. **Agent mechanics are Claude-only, and some don't work as written.** `.claude/rules` with `name`/`description` frontmatter (Claude Code reads only `paths`), commands that duplicate skills, a forked audit skill told to "Enter plan mode" (subagents lose `EnterPlanMode`), `AskUserQuestion` everywhere, and a README `sed` that turns `@<PREVIEW>` into `@@AppPreview`. None of it carries into a portable Agent Skill unchanged.

**Correctness issues in the toolkit.** These are smaller than the gaps above but should be fixed or avoided regardless:
- **Event type contradiction.** `CLAUDE.md:191`, `README.md:131` and `.claude/commands/new-screen.md:129` say `interface <Screen>Event`. `.claude/rules/contracts.md:41,105` says "always `sealed interface`" and lists plain `interface` as a violation.
- **Event bus contradiction.** `.claude/rules/utils.md:20` says an event bus "exposes a `SharedFlow<T>`". `.claude/rules/error-handling.md:156` lists `SharedFlow` as a violation.
- **Preview provider placement, three conflicting rules.**
  - `CLAUDE.md:123` keeps trivial providers in the same file. `.claude/rules/previews.md:89,149` says "always, no in-file exceptions".
  - `.claude/rules/previews.md:123,150,162` says screens never use `@PreviewParameter`, yet `.claude/rules/screens.md:108-118` and `.claude/skills/screen-anatomy/SKILL.md:261-273` generate an in-file `PreviewParameterProvider` for every screen.
  - `CLAUDE.md:124` puts screen-level providers in `<Screen>PreviewParams.kt`.
- **Test naming contradiction.** `.claude/rules/testing.md:15` says camelCase with no underscores. `.claude/commands/add-tests.md:59,129` says `subjectOrMethod_condition_expectedBehavior`.
- **Fallback string key mismatch.** `.claude/rules/error-handling.md:70` uses `R.string.error_generic`, while `.claude/skills/misc-primitives/SKILL.md:60,86` and `.claude/skills/project-scaffold/SKILL.md:429` use `generic_error`. One of the two won't compile.
- **Wrong helper name.** `CLAUDE.md:164` says paging sources use `BasePagingSource.toLoadResult { }`. The shipped helper is `toPage` (`.claude/skills/repository-primitives/SKILL.md:238`).
- **Data-source layer contradiction.** `CLAUDE.md:164,167` assumes a `RemoteDataSource` and a `DataSourceModule` always exist. `.claude/rules/repositories.md:41` and `.claude/rules/data-sources.md:14` say Retrofit projects have no data-source layer.
- **Deprecated import.** `ObserveAsEvents` imports `androidx.compose.ui.platform.LocalLifecycleOwner` (`.claude/rules/error-handling.md:79`, `.claude/skills/misc-primitives/SKILL.md:100`). That symbol is deprecated in favor of `androidx.lifecycle.compose.LocalLifecycleOwner`, which the toolkit's own lifecycle helpers already use (`.claude/rules/lifecycle.md:36`).
- **Token examples that won't compile.** `<THEME>.colors.primary` (`.claude/rules/theming-and-tokens.md:82`) and `<THEME>.colors.onSurface` (`.claude/rules/icons.md:41`) don't exist in the shipped `ExtendedColors` (`.claude/skills/theme-primitives/SKILL.md:264-298`), whose tokens are `textDefault`, `iconDefault` and so on.
- **Dark-mode dimensions contradiction.** `.claude/rules/theming-and-tokens.md:46` says to populate "light and dark values" for a new dimension token, and `:57` of the same file says dimensions have no dark variants.
- **Framework import in the domain layer.** Use cases return `Flow<PagingData<T>>` (`.claude/rules/use-cases.md:41,51`) while the same file bans framework imports in `domain/` (`:62`), and `.claude/rules/models.md:58` greps for `androidx.`.
- **Locale-blind formatting in a domain model.** `displayPrice` (`.claude/rules/models.md:46`) is `"$${priceCents / 100.0}"`. It renders `$1.5`, ignores locale and currency, and puts user-visible text in `domain/`, against `.claude/rules/string-resources.md`.
- **Wrong stability claim.** `.claude/rules/delegates.md:115` says a nested state data class "would mark the parent Compose-unstable". A `val`-only data class of stable types is stable, nested or not. Meanwhile the toolkit allows `List<T>` freely, which the Compose docs say is always unstable (§5).
- **Wrong reasoning in the DI rule.**
  - `.claude/rules/dependency-injection.md:75`: "`@param:` targets the constructor parameter, which is what Hilt's field-injection reads". This is constructor injection, not field injection.
  - `:95`: plain `@Inject` "would create a fresh delegate per VM". An unscoped binding is created per injection site, and `@ViewModelScoped` matters only when several consumers inside one ViewModel component must share the instance.
- **Lifecycle helpers capture stale callbacks.** `OnLifecycleResumed` and `OnLifecycleEvent` (`.claude/rules/lifecycle.md:40-75`) are keyed only on `lifecycleOwner` and don't use `rememberUpdatedState`, so they can run a callback from an earlier composition (inference). The built-in `LifecycleResumeEffect`/`LifecycleEventEffect` (Lifecycle 2.7+) already exist.
- **Fragile navigation primitives.**
  - `PopBackStackAndNavigate` uses `popUpTo(0)` (`.claude/skills/navigation-primitives/SKILL.md:118`), an undocumented destination id.
  - `PopWithResultUsingRouteName` reads `currentBackStack` under `@SuppressLint("RestrictedApi")` and matches route strings with `contains` (`:97-103`).
  - `PopResultKeyValue(val value: Any)` (`:125`) accepts values `SavedStateHandle` can't save. Inference: that crashes at save time.
- **Dropped navigation.** `ViewModelNavImpl.navigate` uses `compareAndSet(null, …)` (`.claude/skills/navigation-primitives/SKILL.md:186`), so a second navigation request while one is pending is silently dropped.
- **Imprecise sealed rule.** Sealed subtypes must be in the same module **and package** (kotlinlang sealed classes, V). `.claude/rules/contracts.md:42` and `.claude/skills/screen-anatomy/SKILL.md:87` say "closed at the module boundary, not the file boundary". It works only because child contracts share the `contracts` package.
- **Broken install `sed`.** `README.md:99` replaces `<PREVIEW>` with `@AppPreview`. The templates write `@<PREVIEW>` and `annotation class <PREVIEW>` (`.claude/rules/previews.md:64`, `.claude/skills/misc-primitives/SKILL.md:248`), so the result is `@@AppPreview` and `annotation class @AppPreview`, which is invalid Kotlin. The `sed` also never substitutes `<PKG_ROOT_PATH>` (`.claude/skills/screen-anatomy/SKILL.md:28`).
- **Missing-Main claim.** `.claude/rules/testing.md:143` says that without `MainDispatcherRule`, `viewModelScope` coroutines "never run". Inference: they throw because the Main dispatcher is missing.
- **PR command contradiction.** `.claude/commands/pr-template.md:36` says "Just create the draft", but `:31` runs `gh pr create --base dev` without `--draft`.

---

## 1. Toolchain baseline and project structure

**Toolkit today**
- No versions anywhere: Kotlin, AGP, Gradle, compileSdk, targetSdk and minSdk are all absent. `/init-project` deliberately leaves `build.gradle.kts` and `libs.versions.toml` untouched (`.claude/commands/init-project.md:24`).
- "Always add new dependencies" to `gradle/libs.versions.toml` (`CLAUDE.md:160`).
- Single module (`CLAUDE.md:9,128`, `.claude/rules/folder-structure.md:10`).
- It requires the `-Xexplicit-backing-fields` compiler flag (`.claude/rules/viewmodels.md:29-32`, `.claude/skills/project-scaffold/SKILL.md:420-422`).

**Current guidance**
- **Kotlin.**
  - 2.4.21 (2026-10-08) is the latest stable. 2.4.0 shipped 2026-06-03 and 2.3.0 on 2025-12-16 (https://kotlinlang.org/docs/releases.html, V).
  - Kotlin 2.4.0 "promotes context parameters, explicit backing fields, and annotation use-site targets features to Stable" (https://kotlinlang.org/docs/whatsnew24.html, V).
- **AGP and Gradle.**
  - AGP 9.4.0 (September 2026) needs Gradle 9.6.0+, JDK 17 and Build Tools 36.0.0, and supports up to API 37 (https://developer.android.com/build/releases/gradle-plugin, V).
  - AGP 9.0 changed several defaults: built-in Kotlin (`android.builtInKotlin=true`, so no `org.jetbrains.kotlin.android` plugin), the new DSL on, stricter R8 full-mode keep rules, and `resValues` off (https://developer.android.com/build/releases/past-releases/agp-9-0-0-release-notes, V).
  - AGP 9.4 notes say "Starting with AGP 10, the new Variant API is mandatory" (V).
- **SDK levels.**
  - Play requires targetSdk 36 for new apps and updates from 2026-08-31, with an extension to 2026-11-01. API 35 was required from 2025-08-31 (https://support.google.com/googleplay/android-developer/answer/11926878, V). No API 37 deadline is published.
  - Now in Android uses compileSdk 36, targetSdk 36 and minSdk 23 (`build-logic/convention`, read via GitHub, V).
  - Dagger 2.60 raised Hilt's minSdk to 23 (https://github.com/google/dagger/releases, V).
- **Gradle conventions.**
  - Now in Android uses `gradle/libs.versions.toml` plus convention plugins in `build-logic/convention` (`nowinandroid.android.application`, `.library.compose`, `.feature.impl`, `.hilt`, …). Its `gradle.properties` turns the configuration cache on with `problems=fail` (https://github.com/android/nowinandroid, V).
  - Gradle 9.0 calls the configuration cache "the preferred mode of execution", but it is still off by default (https://docs.gradle.org/9.0.0/release-notes.html, V; https://docs.gradle.org/current/userguide/configuration_cache_enabling.html, V).
- **Modularization.**
  - Google says "It doesn't always make sense to modularize your project". Codebase size is the main deciding factor, and modularization is a necessity for reuse, strict visibility or Play Feature Delivery (https://developer.android.com/topic/modularization, V).
  - The patterns page describes app, feature, data, core/common and test modules, and recommends version catalogs and convention plugins (https://developer.android.com/topic/modularization/patterns, V).

**Gap / recommendation**
- **ADD (must):** never change an existing project's Kotlin, AGP, Gradle, compileSdk, targetSdk or minSdk without asking. This mirrors the iOS "ask before changing language mode" must.
- **ADD (default):** for new projects, compileSdk and targetSdk 36 (the Play floor), the latest stable Kotlin and AGP, and minSdk as a project decision with 23 or higher as the documented floor (Hilt and NIA both use 23).
- **KEEP:** dependencies in the version catalog only, with no hard-coded versions in module build files.
- **ADD (default):** convention plugins in `build-logic/` for anything shared by two or more modules, and the configuration cache on for new projects.
- **CHANGE:** treat single-module as one option, not the assumption. Default to one `app` module for small apps, with a documented path to `core:*` and `feature:*` modules when a feature gets big or is shared (open decision). The rules should describe layering (UI, domain, data) independently of module layout, so they hold in both.
- **CHANGE:** the explicit backing fields flag is only needed on Kotlin 2.3.x. See §2.

**Community split?** Yes, on how early to modularize. Google's own guidance is "when the codebase size warrants it".

**Confidence:** High on versions and Play dates. Medium on the minSdk default (opinion).

---

## 2. Architecture and state

**Toolkit today**
- **UiState and events.**
  - `<Screen>UiState` is a plain `data class` with defaults (`.claude/rules/contracts.md:28-43`).
  - Events are a sealed interface, with child delegate events extending it (`.claude/rules/contracts.md:57-63`), contradicted by `interface <Screen>Event` (see Correctness).
  - One public `handleEvent(event)` per ViewModel with a `when` (`.claude/rules/viewmodels.md:81`), plus a separate `handle<Name>Event` per shared delegate (`:82`).
- **State exposure.**
  - Explicit backing field `val uiState: StateFlow<T>` with `field = MutableStateFlow(...)`, and `_uiState` banned (`.claude/rules/viewmodels.md:27-43,77`).
  - A custom `reduce` extension that wraps `update` (`.claude/skills/misc-primitives/SKILL.md:224`), and direct `MutableStateFlow.update {}` banned (`.claude/rules/viewmodels.md:80`).
- **Loading and structure.**
  - Loading starts in `init { loadItems() }` (`.claude/rules/viewmodels.md:45-47`).
  - Strict member ordering, with the `companion object` last (`.claude/rules/viewmodels.md:14-25`).
- **Missing:** `stateIn`, `SavedStateHandle` (except for navigation results) and process death.

**Current guidance**
- **UDF.** "ViewModels expose UI state using the observer pattern and receive actions from the UI through method calls" (Strongly recommended) (https://developer.android.com/topic/architecture/recommendations, V).
- **UiState.** Expose "a single property called `uiState`" as a `StateFlow`. "Create the `uiState` using the `stateIn` operator with the `WhileSubscribed(5000)` policy" when data comes as a stream, and a `MutableStateFlow` exposed as `StateFlow` is fine in simpler cases (Recommended). UiState may be a data class or a sealed class (same page, V).
- **One function per action.** All Google samples use one method per user action (`fetchArticles(category)`, `followAuthor(followed)`); none uses an `onEvent` dispatcher, and Google prescribes neither (https://developer.android.com/topic/architecture/ui-layer, V; https://developer.android.com/topic/architecture/ui-layer/stateholders, V).
  - Now in Android uses methods (`updateTopicSelection`, `dismissOnboarding`, `followTopic`) (V).
  - Inference: `handleEvent` (MVI-style) is a team preference, not a platform rule.
- **No async work in `init`.** "Warning: Don't launch asynchronous operations in the `init` block or constructor of a `ViewModel`… When the ViewModel holds Compose State fields, … an `IllegalStateException` can occur."
  - The alternative: "initialize the state production pipeline lazily … with the `started` argument in the `stateIn` method", otherwise "define an idempotent `initialize` function".
  - Source: https://developer.android.com/topic/architecture/ui-layer/state-production (V).
- **`update`.** "When using `MutableStateFlow`, use the `update` method as usual" (same page, V).
- **Process death.** "The `SavedStateHandle` object can be read from to provide the minimum amount of information required to restore the UI state from the data layer" (stateholders page, V).
- **Explicit backing fields.**
  - Experimental in 2.3.0 behind `-Xexplicit-backing-fields` (https://kotlinlang.org/docs/whatsnew23.html, V).
  - Stable in 2.4.0, and properties.html documents it with no flag (https://kotlinlang.org/docs/properties.html, V).
  - Constraints: a `val`, not `open`, not delegated, with a `private` field whose type is a subtype of the property type (V).
  - KEEP: https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0430-explicit-backing-fields.md (S). YouTrack: KT-14663 (V).
  - A Kotlin Slack thread reports a spurious "experimental" IDE warning on 2.4.0 (S, not primary).

**Gap / recommendation**
- **KEEP:** a single immutable `UiState` data class per screen, with defaults, exposed as `StateFlow`.
- **KEEP:** `MutableStateFlow` is never public.
- **CHANGE (default):** produce state with `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initial)` when it derives from repository flows. Use `MutableStateFlow` only for state the ViewModel owns outright (form input, selection).
- **CHANGE (must):** no async work in `init`. Start loading through `stateIn` or an idempotent `initialize`. This replaces `.claude/rules/viewmodels.md:45-47`.
- **CHANGE (default):** explicit backing fields when the project is on Kotlin 2.4+ (stable, no flag). On 2.3, use `private val _uiState` plus `asStateFlow()`, and never add an experimental flag to a client project without asking. **DROP** the "no `_uiState`" ban as a must: it fails on any project below 2.4.
- **DROP:** the `reduce` extension and the ban on `update {}`. It is a one-line alias that adds a project-specific verb with no behavior change. **OPTIONAL** if a project already has it.
- **OPTIONAL:** `handleEvent(event)` vs one function per action. Default to one function per action, which matches Google and NIA (open decision). If a project uses `handleEvent`, keep the event type `sealed interface` so `when` is exhaustive.
- **ADD (default):** `SavedStateHandle` for small user input that must survive process death (query text, selected tab, IDs), and navigation arguments via `savedStateHandle.toRoute<T>()` (§6).
- **OPTIONAL:** strict member ordering. Kotlin's conventions say "properties and initializer blocks, secondary constructors, methods, companion object", and "Do not sort the method declarations alphabetically or by visibility… put related stuff together" (https://kotlinlang.org/docs/coding-conventions.html, V). Keep "companion object last"; drop the rest.

**Community split?** Yes. MVI-style single dispatchers (Orbit, custom `onEvent`) are common in teams. Google's docs and samples show method calls.

**Confidence:** High on Google and Kotlin facts. The dispatcher choice is opinion.

---

## 3. One-off events and error handling in the UI

**Toolkit today**
- **The bus.** A process-wide `object ErrorEventBus` backed by `Channel<UiText>(BUFFERED)` with `receiveAsFlow()` (`.claude/rules/error-handling.md:34-46`). ViewModels call `ErrorEventBus.send(message.toUiText())`. `MainScreen` collects it with `ObserveAsEvents`, which uses `repeatOnLifecycle(STARTED)` plus `Dispatchers.Main.immediate`, and shows one dialog (`.claude/skills/project-scaffold/SKILL.md:222-228`).
- **The rule text.** "ViewModels never own an error dialog" (`.claude/rules/error-handling.md:141`). The bus is "exactly-once delivery" (`:128`). Tests must call `ErrorEventBus.drain()` (`:146`).
- **Mixed pattern.** Contextual errors and toasts use state flags, e.g. `showSavedToast` reset by `OnToastShown` (`.claude/rules/lifecycle.md:123-131`, `.claude/rules/error-handling.md:143`).
- **Error state is lost on rotation.** The dialog state lives in `remember { mutableStateOf<UiText?>(null) }` in `MainScreen` (`.claude/skills/project-scaffold/SKILL.md:222`). A configuration change drops the visible error, and a second error overwrites the first (inference).

**Current guidance**
- "ViewModel events should always result in a UI state update." The consumed message is "user input", so the ViewModel clears it, e.g. `userMessageShown()` (https://developer.android.com/topic/architecture/ui-layer/events, V).
- The same page lists as an antipattern: "ViewModel events being exposed to the UI using Kotlin Channels or other reactive streams. When the producer (the ViewModel) outlives the consumer (Compose UI), these solutions don't guarantee the delivery and processing of those events." "Handle such events immediately and reduce them to UI state." (V)
- Recommendations: "Do not send events from the ViewModel to the UI. Process the event immediately in the ViewModel and cause a state update with the result" (Strongly recommended) (https://developer.android.com/topic/architecture/recommendations, V).
- **Vivo's article.** Manuel Vivo (Android DevRel), "ViewModel: One-off event antipatterns", 2022-06-01.
  - "A Channel doesn't guarantee the delivery and processing of the events." `Dispatchers.Main.immediate` mitigates this but is error-prone because nothing enforces it.
  - Source: https://manuelvivo.dev/viewmodel-events-antipatterns (V, author's mirror). The Medium original returned 403 (U).
- **NIA has no global error bus.** It shares a `SnackbarHostState` through a CompositionLocal, drives an offline snackbar from state, and its reads come from local storage, so "errors are not expected when reading from Repository instances" (V).
- **Community:** `ObserveAsEvents` plus `Main.immediate` is a popular community pattern and is the mitigation Vivo describes. It is not Google's recommendation.

**Gap / recommendation**
- **CHANGE (default):** user-visible messages are state. Use a `userMessage: UiText?` (or a queue) field in `UiState`, render it from the screen, and call `onMessageShown()` to clear it.
  - App-wide messages (offline, session expired) come from an app-level state holder or a repository flow collected in the app shell, not a static `object`.
- **DROP (must for new code):** a global singleton event bus as the default error channel. **OPTIONAL** when a project already has one: then require `Main.immediate` on both ends and collection in an always-present host, as the toolkit does, and inject the bus instead of a Kotlin `object` so tests don't need `drain()`.
- **CHANGE (must):** never show raw exception or server messages to users. Map errors to typed domain errors and then to string resources (§7).
- **KEEP:** `UiText` (string resource or dynamic string) as the carrier for user-visible text produced outside composables. Add format args (`StringResource(id, args)`); the toolkit bans them (`.claude/rules/error-handling.md:155`).

**Community split?** Yes. Many teams use Channels for "effects" and accept the edge cases. Google's position is explicit and has been stable since 2022.

**Confidence:** High.

---

## 4. Delegates (`@ViewModelScoped` sub-flow holders)

**Toolkit today**
- Delegates own their own `StateFlow`, receive `viewModelScope` through `lateinit var scope` set in `init(scope)`, and are `@ViewModelScoped` (`.claude/rules/delegates.md:34-63,113-118`).
- There are feature-scoped and shared variants. Each shared delegate gets its own `handle<Name>Event` on the host and its own lambda parameter on `ScreenContent` (`:143-197`).
- Shared delegates take host callbacks through `init` (`:83-91`).

**Current guidance**
- Google: business logic and screen state belong in the ViewModel. "For shorter-lived UI state and UI logic, a plain class whose lifecycle is dependent solely on the UI should suffice." Plain state holders are for reusable UI pieces. "DO NOT pass a ViewModel instance to a plain state holder class" (https://developer.android.com/topic/architecture/ui-layer/stateholders, V).
- Reusable business logic across ViewModels belongs in the domain layer (use cases), which is "Recommended in big apps… if you need to reuse business logic… across multiple ViewModels" (https://developer.android.com/topic/architecture/recommendations, V).
- Google has no guidance on ViewModel-scoped helper classes that own state and borrow `viewModelScope` (U). Inference: `@ViewModelScoped` is a documented Hilt scope, so the mechanism is legitimate, but `lateinit var scope` plus `init(scope)` is two-phase initialization that fails at runtime if `init` is forgotten.

**Gap / recommendation**
- **OPTIONAL:** keep delegates as a documented pattern for large screens with independent sub-flows, but don't generate them by default (`/new-screen` asks about them on every screen, `.claude/commands/new-screen.md:59-76`).
- **CHANGE:** if kept, pass the scope through the constructor with assisted injection or a factory, or expose `fun bind(scope): StateFlow<…>`, so a missing `init` is a compile error rather than an `UninitializedPropertyAccessException` (inference).
  - **DROP** the bespoke routing rules: separate `handle<Name>Event` functions, the wrapper ban, contract co-location exceptions.
- **ADD (default):** first reach for a use case (shared business logic) or a plain state holder (UI logic) before a delegate.

**Community split?** Yes. ViewModel delegates and "child ViewModels" are a recurring community debate with no Google position.

**Confidence:** Medium. The Google text supports plain state holders and use cases, but says nothing about this exact pattern.

---

## 5. Compose UI: screens, components, previews, lifecycle, stability

**Toolkit today**
- **Screens.**
  - Every screen file has exactly two composables: a public entry that takes `navController` and `hiltViewModel()`, and a `private` stateless `<Screen>ScreenContent` (`.claude/rules/screens.md:14-45`).
  - Every screen wraps one app-wide `<SCAFFOLD>` (`:95-97`), and Material3 `Scaffold` is banned in screens (`:129`).
- **Components.**
  - One component per file. `private @Composable` is banned except previews and `ScreenContent` (`.claude/rules/composables.md:11-31`). Non-composable helpers can't live in component files (`:85-89`).
  - Modifier conventions follow the Compose guidelines (`.claude/rules/composables.md:33-61`), and "callers pass `modifier` positionally" (`:60`).
- **Previews.**
  - Every composable has a custom `@<PREVIEW>` light/dark preview at the bottom of its file (`.claude/rules/previews.md:13-28`).
  - Every provider goes in a sibling `*PreviewParams.kt` (`:89`).
- **Lifecycle.**
  - Custom `OnLifecycleResumed` and `OnLifecycleEvent` helpers (`.claude/rules/lifecycle.md:14-75`).
  - "`LaunchedEffect(Unit)` is the default" (`:121`).
- **Missing:** strong skipping, keys in lazy lists (mentioned only as an advisory audit smell, `.claude/skills/audit-branch/SKILL.md:115`) and `remember`/`derivedStateOf` guidance.

**Current guidance**
- **Compose API guidelines** (https://github.com/androidx/androidx/blob/androidx-main/compose/docs/compose-api-guidelines.md, V; https://github.com/androidx/androidx/blob/androidx-main/compose/docs/compose-component-api-guidelines.md, V):
  - The modifier "MUST appear as the first optional parameter", defaults to `Modifier`, is applied once to the root-most layout, and is the only `Modifier` parameter.
  - Order is: required params, `modifier`, optional params, trailing `@Composable` content lambda. Single slots are named `content`.
  - "Prefer stateless and controlled @Composable functions."
  - Neither document says anything about one component per file or banning private composables.
  - The guidelines are strictest for framework code and loosest for app code (https://developer.android.com/develop/ui/compose/api-guidelines, V).
- **Kotlin file conventions.** "Placing multiple declarations… in the same Kotlin source file is encouraged as long as these declarations are closely related to each other semantically, and the file size remains reasonable" (https://kotlinlang.org/docs/coding-conventions.html, V).
- **ViewModels at screen level only.** "Don't pass ViewModel instances down to other composable functions… pass down just the necessary state" (stateholders page, V).
- **Stability.**
  - A class is stable when all its properties are `val` and of immutable types (https://developer.android.com/develop/ui/compose/performance/stability/fix, V).
  - "the Compose compiler always marks a parameter of type `List` as unstable" (same page, V).
  - "Strong Skipping is enabled by default in Kotlin 2.0.20". Unstable params compare by `===` and lambdas are auto-remembered (https://developer.android.com/develop/ui/compose/performance/stability/strongskipping, V).
  - The docs say "you shouldn't attempt to make every composable skippable" (V).
  - Types from modules not built with the Compose compiler count as unstable (V).
- **Lifecycle effects.** `LifecycleStartEffect`, `LifecycleResumeEffect` and `LifecycleEventEffect` arrived in Lifecycle 2.7.0 (https://developer.android.com/topic/libraries/architecture/compose, V). The recommendations page tells you to use them instead of overriding Activity callbacks (Strongly recommended, V). Lifecycle 2.11.0 is stable (2026-06-17, V).
- **`LocalLifecycleOwner`.** It moved to lifecycle-runtime-compose in 2.8.0, and the old `androidx.compose.ui.platform` symbol is `@Deprecated` (androidx source, V).
- **Previews.**
  - Built-in multipreviews: `@PreviewLightDark`, `@PreviewScreenSizes`, `@PreviewFontScales`, `@PreviewDynamicColors` (ui-tooling-preview 1.6+) (https://developer.android.com/develop/ui/compose/tooling/previews, V).
  - Google says nothing official on private previews (U). slackhq compose-lints rule `ComposePreviewPublic` recommends `private` (Community, V).
- **Scaffold and insets.** "`Scaffold` does not apply the insets to content; this responsibility is yours." Avoid extra inset handling with Scaffold "to avoid applying too much padding" (https://developer.android.com/develop/ui/compose/system/material-insets, V).

**Gap / recommendation**
- **KEEP:** the stateful entry and stateless content split, with state hoisting and `collectAsStateWithLifecycle()`.
- **KEEP:** the ViewModel only at screen level, never passed down.
- **CHANGE:** `ScreenContent` should be `internal` (or public in a feature module), not `private`. Compose UI tests and screenshot tests need to call it with fake state (inference). The screen entry takes navigation lambdas, not `NavController` (§6).
- **KEEP:** the modifier conventions, citing the androidx guidelines. **DROP** "callers pass `modifier` positionally" (`.claude/rules/composables.md:60`): no guideline says it, and named arguments are clearer.
- **DROP:** one component per file and the private-composable ban.
  - Both contradict Kotlin's file conventions, and Google doesn't require either.
  - **ADD (default):** a public or internal reusable component gets its own file. Private sub-composables used by one parent stay in the parent's file.
- **OPTIONAL:** a project-wide `<SCAFFOLD>` wrapper. Fine if design demands it, but it must pass `contentWindowInsets` through and must not double-pad (§15).
- **CHANGE:** replace the custom lifecycle helpers with `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`.
  - **ADD (must):** any long-lived effect that calls a lambda parameter uses `rememberUpdatedState`.
  - **CHANGE:** "refresh on resume" is a UI trigger for a ViewModel call, not a data-loading strategy. Data that must stay fresh should come from a flow (§7).
- **CHANGE:** previews.
  - Previews are required for reusable components and screen content.
  - Use the built-in `@PreviewLightDark`/`@PreviewFontScales` plus a project wrapper only when the project has one.
  - `PreviewParameterProvider` for real state variants, in whichever file the project uses.
  - Pick one rule and delete the three conflicting ones.
- **ADD (default):** stability rules.
  - Don't chase skippability.
  - Under strong skipping, prefer stable `val`-only state types. Use `ImmutableList` or a stability config file only when profiling shows a problem.
  - Always provide `key` in `LazyColumn`/`LazyRow` items.
  - Read fast-changing state in lambdas or `derivedStateOf`.
- **DROP:** the "nested data class makes the parent unstable" claim (`.claude/rules/delegates.md:115`).

**Community split?** Yes, on file granularity and private previews. Slack's compose-lints wants private previews; Google is silent.

**Confidence:** High on Compose docs. File-granularity rules are opinion.

---

## 6. Navigation

**Toolkit today**
- Navigation Compose 2.x with `@Serializable object`/`data class` routes that implement a marker `NavigationRoute` (`.claude/rules/navigation.md:15-38`).
- ViewModels implement `ViewModelNav by ViewModelNavImpl()`. The impl holds a `StateFlow<NavigationAction?>`, and `HandleNavigation(viewModel, navController)` executes it in a `LaunchedEffect` (`.claude/skills/navigation-primitives/SKILL.md:161-226`).
- Every screen takes `navController: NavController` (`.claude/rules/screens.md:22-34`).
- Results go through `savedStateHandle` via `popBackStackWithResult` and `ObserveBooleanResult`. Direct `savedStateHandle` access elsewhere is banned (`.claude/rules/navigation.md:117-122`).

**Current guidance**
- **Navigation 2.x.**
  - 2.10.2 (2026-09-23) is stable; type-safe routes arrived in 2.8.0 (https://developer.android.com/jetpack/androidx/releases/navigation, V; https://developer.android.com/guide/navigation/design/type-safety, V).
  - The Navigation releases page carries a page-level caution: "This library is in maintenance mode and will only receive critical fixes; new features are not planned. We recommend using Jetpack Compose" (V). It points to Compose, so it may target the View-based artifacts, but the page doesn't scope it (inference, U).
- **Callbacks, not `NavController`.** "Warning: Don't pass your `NavController` to your composables. Expose an event." "You should only call `navigate()` as part of a callback" (https://developer.android.com/guide/navigation/use-graph/navigate, V).
- **Navigation events.** UI-triggered navigation goes through callbacks. When navigation depends on ViewModel logic, "the ViewModel would need to expose that state to the UI. The UI would react to that state change and navigate accordingly" (https://developer.android.com/topic/architecture/ui-layer/events, V). That is close to the toolkit's action-as-state, but scoped to the validation case.
- **Navigation 3.**
  - Stable since 1.0.0 (2025-11-19); 1.2.0 (2026-09-23) is current (https://developer.android.com/jetpack/androidx/releases/navigation3, V).
  - "You own the back stack": a `SnapshotStateList` of `@Serializable` `NavKey`s rendered by `NavDisplay` with an `entryProvider`. `rememberNavBackStack` survives process death (https://developer.android.com/guide/navigation/navigation-3/basics, V; /save-state, V).
  - ViewModels are scoped per entry through `ViewModelStoreNavEntryDecorator` (/naventrydecorators, V).
  - Results: `ResultEventBus` (1.2.0+), which is in-memory and lost on process death (/return-results, V).
  - The architecture recommendations say "Use Navigation 3 to navigate between screens" for single-activity apps (Strongly recommended, V).
  - The stable blog post says Navigation Compose users "should consider migrating to Nav3" (https://android-developers.googleblog.com/2025/11/jetpack-navigation-3-is-stable.html, V).
  - NIA has migrated to Nav3, with assisted Hilt ViewModels keyed by `NavKey` (V).

**Gap / recommendation**
- **CHANGE (must):** screens receive navigation lambdas (`onBack`, `onOpenDetail: (Id) -> Unit`). They never receive `NavController` or the back stack, and the nav graph wires those lambdas.
- **DROP:** `ViewModelNav`, `NavigationAction`, `HandleNavigation` and the route-name popping helpers.
  - When the ViewModel must decide (after a save succeeds), expose that in `UiState` (`savedId`, `isDone`) and let the screen call its lambda. That is Google's documented pattern.
- **KEEP:** `@Serializable` type-safe routes. **CHANGE:** use `data object` for parameterless routes and read arguments with `toRoute<T>()`.
- **ADD (default):** Navigation 3 for new Compose projects, with Navigation Compose 2.x kept in existing projects. Never migrate without asking (open decision).
- **CHANGE:** results.
  - On Nav 2.x, use `previousBackStackEntry.savedStateHandle` with Bundle-able values only.
  - On Nav 3, use `ResultEventBus` for transient results, and the `NavKey` or `SavedStateHandle` for anything that must survive process death.

**Community split?** Yes. Some teams like VM-driven navigation for testability. Nav3 vs Nav2 depends on project age.

**Confidence:** High on docs. Medium on the scope of the "maintenance mode" caution.

---

## 7. Data layer

**Toolkit today**
- **Interfaces.** Repository interfaces live in `domain/repository/`, return `ApiResult<T>` (suspend) or `Flow<ApiResult<T>>`, and impls extend `BaseRepository` (`.claude/rules/repositories.md:14-35,103-107`).
- **Call factories.** Apollo data sources are "call factories": non-suspend methods that return `ApolloCall` and never execute, split by auth level (`.claude/rules/data-sources.md:16-31`). Retrofit projects inject the API service directly (`.claude/rules/repositories.md:74-101`).
- **`ApiResult`.** `ApiSuccess`, `ApiError(code, message)` and `ApiLoading` are top-level non-data classes (`.claude/skills/repository-primitives/SKILL.md:29-50`).
- **Retrofit `safeCallSuspend`** (`.claude/skills/repository-primitives/SKILL.md:185-202`):
  - It catches only `IOException`/`HttpException`.
  - A null body becomes `ApiError("Empty body")`, which breaks 204 responses.
  - The raw `errorBody()?.string()` becomes the user-facing message.
- **Apollo `safeApolloCallSuspend`** catches only `ApolloException` (`:115-124`), so an exception thrown by the mapper escapes.
- **Missing:** Room, DataStore and offline-first appear only in passing. `.claude/rules/data-sources.md:79` covers DAOs.

**Current guidance**
- **Data layer page** (https://developer.android.com/topic/architecture/data-layer, V):
  - Naming: data sources are "type of data + type of source + DataSource" (`NewsRemoteDataSource`). "Each data source class should have the responsibility of working with only one source of data."
  - Entry point: other layers "should never access data sources directly; the entry points to the data layer are always the repository classes".
  - Exposure: "For one-shot operations, expose suspend functions." "To be notified of data changes over time, expose flows." Calls "should be main-safe".
  - Errors: "use `try/catch` blocks when appropriate; and in flows, use the `catch` operator… the UI layer is expected to handle exceptions". Alternatively, model results with a `Result` class "making the UI aware of known errors".
  - Models: "create new models in any case where a data source receives data that doesn't match with what the rest of your app expects".
- **Recommendations.** "Create repositories even if they contain only a single data source" (Strongly recommended). "Create a model per layer in complex apps" (Recommended) (V).
- **Offline-first.** "The local data source is the canonical source of truth for the app." "The domain and UI layers… must never communicate directly with the network layer" (https://developer.android.com/topic/architecture/data-layer/offline-first, V).
- **Paging.**
  - "Because `PagingData`… is not an immutable type—do not represent it in an immutable UI state. Instead, expose it from the ViewModel independently in its own stream" (https://developer.android.com/topic/architecture/ui-layer, V). That confirms the toolkit's Paging rule (`CLAUDE.md:105`).
  - Apply `cachedIn` in the ViewModel (https://developer.android.com/topic/libraries/architecture/paging/v3-transform, V).
  - Paging 3.5.3 (2026-10-12) is stable (V).
- **Retrofit.**
  - 3.0.0 (2025-05-15) is current and binary-compatible with 2.x. The kotlinx-serialization converter is official since 2.10.0 (https://github.com/lysine-dev/retrofit/blob/master/CHANGELOG.md, V).
  - The repo moved from `square/` to `lysine-dev/`, and Maven coordinates are unchanged (V).
- **Other clients.** OkHttp 5.5.0 (2026-08-16, V), Ktor 3.6.0 (2026-09-18, V), Apollo Kotlin 5.2.0 (2026-09-16, V). Apollo 4 uses the `com.apollographql.apollo` package (V); the 5.x package is U.

**Gap / recommendation**
- **KEEP:**
  - Repositories as the only entry point.
  - Domain types at the repository boundary.
  - Suspend for one-shot calls and `Flow` for streams.
  - Mapping at the boundary via `toX()` extensions.
  - PagingData outside UiState, with `cachedIn(viewModelScope)`.
- **CHANGE (must):** a repository or data source never lets an expected failure (network, HTTP, parsing) escape untyped, and never swallows `CancellationException`.
  - Use either exceptions caught at the ViewModel or a sealed result. Pick one per project.
  - Default to Kotlin `Result` or a small sealed `DataResult` with a typed error (`Network`, `Unauthorized`, `NotFound`, `Server(code)`, `Unknown`), not a `message: String?` (open decision).
- **DROP:** `ApiLoading` inside a result type. Loading is UI state, not a data-layer result (inference from Google's `Result` sample, which has no loading case).
- **DROP:** `BaseRepository` inheritance. Use a top-level `suspend inline fun <T> safeApiCall(block)` or a Retrofit call adapter, so repositories compose instead of inheriting (inference). **OPTIONAL** in projects that have one.
- **CHANGE:** data sources. Follow Google's definition: one source each (remote, local), executing calls. **OPTIONAL:** the Apollo "call factory" split by auth level; it is a valid project choice but not the default.
- **ADD (default):** offline-first when the app has a local cache. Room or DataStore is the source of truth, the network syncs into it, and repositories expose `Flow` from the local source.
- **ADD (must):** network DTOs are `@Serializable` data-layer types and are never shown in UI or domain signatures. **KEEP** "no network types in domain".
- **CHANGE:** model placement. UI models (`uimodel/`) belong in the UI layer, not `domain/model/uimodel/` (`.claude/rules/folder-structure.md:18`).

**Community split?** Yes, on `Result` wrappers vs exceptions and on ArrowKt `Either`. Google documents both and prefers neither strongly.

**Confidence:** High on Google text. Medium on the `BaseRepository` replacement (inference).

---

## 8. Domain layer and use cases

**Toolkit today**
- "Only create a use case when the logic is non-trivial or reused across multiple ViewModels", citing Google (`.claude/rules/use-cases.md:10-13`).
- Shape: a single `operator fun invoke`, stateless, no `@Singleton`, named `<Verb><Resource>UseCase` (`:37-64`).
- Use cases return `ApiResult` or `Flow<PagingData>` (`:41`).

**Current guidance** (https://developer.android.com/topic/architecture/domain-layer, V)
- "The domain layer is an optional layer… You should only use it when needed-for example, to handle complexity or favor reusability." The toolkit's claim is accurate.
- Naming: "verb in present tense + noun/what (optional) + UseCase". `invoke()` with `operator` is shown.
- "It's normal to have multiple levels of use cases", so use cases may depend on other use cases.
- "Use cases don't have their own lifecycle… you should create a new instance of a use case class every time you pass it as a dependency." "Use cases from the domain layer must be main-safe."
- NIA keeps use cases in `core:domain` for combining flows (`GetFollowableTopicsUseCase`), and its ViewModels call repositories directly for events (V).

**Gap / recommendation**
- **KEEP:** the "only when needed" rule, `operator fun invoke`, the naming, unscoped and stateless.
- **ADD:** use cases may call other use cases. They must be main-safe, with injected dispatchers for CPU work.
- **CHANGE:** the purity rule. Domain code has no Android framework or UI imports. Decide explicitly whether `PagingData` (paging-common is pure Kotlin) is allowed (inference). Today the rule and its own example disagree.
- **DROP:** the "one file per resource" folder taxonomy under `domain/model/response|request|uimodel` as a must. **OPTIONAL** as a project layout.

**Confidence:** High.

---

## 9. Dependency injection

**Toolkit today**
- Hilt, with `ApiModule`, `DataSourceModule` and `RepositoryModule` in `data/di/`. `@Binds` is preferred over `@Provides` (`.claude/rules/dependency-injection.md:10-53`).
- `@Named` string qualifiers with constants in one file (`:56-75`).
- Repositories and data sources are `@Singleton`, delegates `@ViewModelScoped`, use cases unscoped (`:79-86`).
- `hiltViewModel()` is imported from `androidx.hilt.navigation.compose` (`.claude/skills/screen-anatomy/SKILL.md:200`).
- No assisted injection, and no dispatcher injection.

**Current guidance**
- "Use constructor injection" and "Scope to a component only when needed" are both Strongly recommended. "Use Hilt" is Recommended (https://developer.android.com/topic/architecture/recommendations, V).
- `@Binds` tells Hilt which implementation to use for an interface. `@Provides` is for classes you don't own or that need builders (https://developer.android.com/training/dependency-injection/hilt-android, V).
- **`hiltViewModel()` move.** In androidx.hilt 1.3.0 (2025-09-10), "The `hiltViewModel()` APIs for Compose have been moved to a new artifact (`androidx.hilt:hilt-lifecycle-viewmodel-compose`)… without transitively depending on `androidx.navigation`". 1.4.0 (2026-07-01) is current (https://developer.android.com/jetpack/androidx/releases/hilt, V). Deprecation of the old import is U.
- **Assisted ViewModels.** `@HiltViewModel(assistedFactory = …)` plus `withCreationCallback`. "Assisted values are not saved to disk" (https://dagger.dev/hilt/view-model, V). Added in Dagger 2.49 (V).
- **Versions.** Dagger 2.60.1 (2026-07-06, V).
- **Alternatives.**
  - Koin 4.2.2 (2026-06-15) and the Koin Compiler Plugin 1.2.1 for compile-time checks (V).
  - Metro 1.0.0 (2026-04-27), a compiler-plugin DI framework, latest 1.4.5 (https://github.com/ZacSweers/metro, V; third-party).
- **Kotlin annotation targets.** With Kotlin 2.4 the new default targets are stable: an annotation on a constructor property applies to `param` and `property`, and `@param:` remains valid (https://kotlinlang.org/docs/annotations.html, V).

**Gap / recommendation**
- **KEEP (default):** Hilt, constructor injection, `@Binds` for interface bindings, `@Provides` for builders and third-party types, and scopes only when shared state needs them.
- **OPTIONAL:** the three-module split and the `data/di/` location. Modules belong next to what they bind (per feature or layer module).
- **CHANGE:** custom `@Qualifier` annotations over `@Named("string")` (inference: compile-checked). **CHANGE:** fix the `@param:` explanation. It is a valid way to pin the target, but not because of "field injection".
- **ADD (default):**
  - Inject `CoroutineDispatcher`s with qualifiers (§10).
  - Use assisted injection for ViewModels that need runtime keys, as NIA does with Nav3 keys.
  - Import `hiltViewModel` from `androidx.hilt.lifecycle.viewmodel.compose` on androidx.hilt 1.3+.
- **ADD (audit):** any DI approach is accepted, but flag composables that resolve dependencies directly (mirrors the iOS rule).

**Community split?** Hilt vs Koin vs Metro. Hilt is Google's recommendation; Koin is common in KMP; Metro is new.

**Confidence:** High.

---

## 10. Concurrency

**Toolkit today:** viewModelScope everywhere. `GlobalScope` is a High audit finding (`.claude/skills/audit-branch/SKILL.md:99,132`). Nothing on dispatchers, main-safety, cancellation or `CancellationException`.

**Current guidance** (https://developer.android.com/kotlin/coroutines/coroutines-best-practices, V). The headings in order:
1. Inject Dispatchers ("Don't hardcode `Dispatchers`").
2. Suspend functions should be safe to call from the main thread.
3. The ViewModel should create coroutines.
4. Don't expose mutable types.
5. The data and business layer should expose suspend functions and Flows.
6. Inject TestDispatchers in tests.
7. Avoid GlobalScope (inject a `CoroutineScope`).
8. Make your coroutine cancellable.
9. Watch out for exceptions.

Versions: kotlinx.coroutines 1.11.0 (2026-05-08, V).

**Gap / recommendation**
- **ADD (must):**
  - Suspend functions and repository and use-case APIs are main-safe.
  - Dispatchers are injected, never hard-coded, outside the composition root.
  - No `GlobalScope`. Work that must outlive a screen uses an injected application `CoroutineScope`.
  - Never catch `CancellationException` without rethrowing it, e.g. no bare `catch (e: Exception)` in suspend code. The toolkit's paging helper does this correctly at `.claude/skills/repository-primitives/SKILL.md:243`; the rule should say so everywhere.
- **ADD (default):** combine flows with `combine`/`stateIn`, use `flatMapLatest` for query-driven loads (the toolkit's paging example already does), and avoid `runBlocking` outside `main` and tests.

**Confidence:** High.

---

## 11. Strings, icons, accessibility

**Toolkit today**
- **Strings.**
  - All UI strings come from `stringResource`. Plurals use `pluralStringResource`. Formatted strings use placeholders, not concatenation (`.claude/rules/string-resources.md:10-82`).
  - Enum labels use `@param:StringRes val labelRes: Int` (`:52-64`).
  - Non-UI layers carry `UiText`, never `Context.getString` (`:79`).
- **Icons.** Material Icons (`Icons.*`) are banned. All icons are vector drawables registered in one `<ICONS>` object (`.claude/rules/icons.md:10-29`).
- **Accessibility.** Icon `contentDescription` comes from resources, or is `null` when decorative (`:56-69`). Nothing else on accessibility.
- **Missing:** per-app languages, RTL, and locale-aware number and date formatting.

**Current guidance**
- **Material icons.** "The `androidx.compose.material.icons` library is no longer recommended… We have stopped publishing updates to this library". material3 1.4.0 dropped its `material-icons-core` dependency (compose-material3 release notes, V).
  - Google recommends downloading Material Symbols XML from Google Fonts and using `painterResource` (https://developer.android.com/develop/ui/compose/graphics/images/material, V).
  - The last icons version is 1.7.8 (V).
- **Accessibility.** "make sure to set the minimum size to 48dp" for touch targets. `contentDescription = null` marks a decorative element. Use `clickable(onClickLabel = …)` (https://developer.android.com/develop/ui/compose/accessibility/api-defaults, V).
  - Merge logical groups with `semantics(mergeDescendants = true)`, and use `clearAndSetSemantics` "sparingly" (https://developer.android.com/develop/ui/compose/accessibility/merging-clearing, V).
  - From Compose 1.8.0, `composeTestRule.enableAccessibilityChecks()` (artifact `ui-test-junit4-accessibility`) checks labels, contrast, touch target size and traversal order (https://developer.android.com/develop/ui/compose/accessibility/testing, V).
- **Per-app languages.** Use `generateLocaleConfig = true` plus `res/resources.properties` (AGP 8.1+), and `AppCompatDelegate.setApplicationLocales` for the backport. Compose apps need `AppCompatActivity` for the backport (https://developer.android.com/guide/topics/resources/app-languages, V).
- **Kotlin 2.4 annotation targets.** `@StringRes` on a constructor property now applies to the param and property by default (https://kotlinlang.org/docs/annotations.html, V). `@param:StringRes` still works, but its stated rationale (`.claude/rules/string-resources.md:64`) is now obsolete (inference).

**Gap / recommendation**
- **KEEP (must):** no hard-coded user-visible strings, plurals via `pluralStringResource`, whole localized sentences with placeholders, and content descriptions from resources.
- **KEEP:** `UiText` for text produced outside composables, plus format args (§3).
- **CHANGE:** the icons rule.
  - **KEEP** the ban on `Icons.*` for new code, now for Google's reason (no longer maintained, build-time cost), not only design control.
  - **ADD** Material Symbols vector drawables as the default source.
  - **OPTIONAL** the central `<ICONS>` registry: it adds a lookup with no compile-time benefit over `R.drawable` (inference).
- **ADD (must):**
  - Touch targets of at least 48dp.
  - A content description or `onClickLabel` on every icon-only control.
  - No information carried by color alone.
  - Text sizes in `sp` through the typography scale, with layouts that survive 200% font scale (preview with `@PreviewFontScales`).
- **ADD (default):** one `enableAccessibilityChecks()` UI test per main screen. That is the Android analogue of the iOS `performAccessibilityAudit` rule.
- **ADD (default):** per-app language support via `generateLocaleConfig` in apps with more than one locale.
- **ADD (must):** locale-aware formatting for numbers, currency and dates (`NumberFormat`, `DateTimeFormatter` with the user locale), never string templates. This fixes the `displayPrice` bug.

**Confidence:** High.

---

## 12. Theming and design tokens

**Toolkit today**
- **Bans.** `MaterialTheme.colorScheme` and `MaterialTheme.shapes` are banned. Only `MaterialTheme.typography` is allowed (`CLAUDE.md:23-35`, `.claude/rules/theming-and-tokens.md:10-24`).
- **Tokens.** All colors go through `<THEME>.colors.<semantic>`, from an `ExtendedColors` interface in a `compositionLocalOf`. Primitives are `internal`, and a parallel Material `ColorScheme` is mapped from the same primitives (`.claude/skills/theme-primitives/SKILL.md:18-26,254-358`).
- **Dimensions.** Spacing, padding, radius, icon size and border width live in a `Dimens` object. Any one-off `.dp` must be a file-level `private val` (`.claude/rules/theming-and-tokens.md:26-39`).
- **Missing:** dynamic color, a high-contrast scheme and Material 3 Expressive.

**Current guidance** (https://developer.android.com/develop/ui/compose/designsystems/custom, V)
- "While Material is our recommended design system… you are not forced to use it."
- Three options: extend `MaterialTheme` with extra values, replace one or more Material subsystems, or implement a fully custom system.
- The recommended accessor is an `@Immutable` data class provided through `staticCompositionLocalOf`, plus an `object ExtendedTheme { val colors @Composable get() = Local….current }`. That is the same shape as the toolkit's, but the toolkit uses an interface and `compositionLocalOf`.
- Caveat: when a subsystem is replaced, "using Material components as-is may result in unwanted Material color, type, or shape values". Wrap them in your own components.
- **Dynamic color** is available on Android 12+, with a fallback to a custom scheme (https://developer.android.com/develop/ui/compose/designsystems/material3, V).
- **Material 3 Expressive.**
  - Stable material3 1.4.0 (2025-09-24) ships no Expressive APIs; they were removed in 1.4.0-beta01 and live in 1.5.0 (1.5.0-beta01, 2026-10-07).
  - Components graduate piecemeal (V).
  - BOM 2026.09.00 maps to material3 1.4.0 (https://developer.android.com/develop/ui/compose/bom/bom-mapping, V).

**Gap / recommendation**
- **KEEP (must):** no literal colors (`Color(0x…)`, `Color.Red`) in UI code. All color comes from theme tokens.
- **CHANGE:** the `MaterialTheme.colorScheme` ban.
  - It works only if the project fully maps its tokens into the Material `ColorScheme` too. Material components still read `colorScheme`, so the toolkit maintains two parallel palettes (inference).
  - Default: follow the project's theme accessor. For new projects, choose between extending `MaterialTheme` (Material components stay correct) and a fully custom theme with wrapped components (open decision).
- **CHANGE:** use `staticCompositionLocalOf` with `@Immutable` token classes, as in Google's sample.
- **OPTIONAL:** dimension tokens. **DROP** "every one-off `.dp` must be a file-level constant" as a must. It is noise for genuinely local layout values; keep "repeated values become tokens".
- **ADD (default):** a dynamic-color decision per project (brand apps usually opt out), and dark and light schemes that pass contrast checks.
- **ADD:** no Expressive APIs unless the project is on material3 1.5+ and opts in.

**Community split?** Yes. Brand design systems vs Material. Google endorses both.

**Confidence:** High on docs. The ban's cost is inference.

---

## 13. Logging, crash reporting, privacy

**Toolkit today**
- Timber only. `android.util.Log`, `TAG` constants and `println` are banned. The release tree forwards to a crash reporter (`.claude/rules/logging.md:10-58`).
- "Do not log user-identifiable data" (`:57`). `BaseRepository` logs every error with `Timber.e`, including GraphQL messages (`.claude/skills/repository-primitives/SKILL.md:111-150`).

**Current guidance**
- **Timber.** The last release is 5.0.1 (2021-08-13). The repo is maintained (KMP migration commits in 2025, ktfmt in 2025-11, Renovate in 2026), but nothing has been released in five years (https://github.com/JakeWharton/timber, V). It still works; "Community" adoption is wide.
- Android's own guidance on log privacy and Play's Data safety form were not researched (U).

**Gap / recommendation**
- **KEEP (must):** no `println` or `Log.*` in production code. Log through the project's logging facade (Timber if present).
- **KEEP (must):** no PII, tokens or raw response bodies in logs. **ADD:** never log full request or response bodies outside debug builds, and keep `HttpLoggingInterceptor` at `NONE` in release.
- **OPTIONAL:** Timber as the default facade for new projects (open decision: Timber, Kermit for KMP, or a thin in-house interface).
- **ADD:** crash reporting is follow-only. Use the project's SDK (Crashlytics, Sentry), never introduce one, and keep the Play Data safety declaration in mind when adding an SDK (U on the exact Play wording).

**Confidence:** Medium-high.

---

## 14. Testing

**Toolkit today**
- JUnit4, `MainDispatcherRule` (Unconfined), Turbine on `uiState.test { }`, hand-written fakes, no mocking libraries, camelCase names (`.claude/rules/testing.md:13-136`).
- The template expects `loading = true` then `loading = false` from a fake that returns immediately (`:55-69`).
- Compose UI tests get one line (`:136`). There are no screenshot tests.
- `/add-tests` covers unit tests only (`.claude/commands/add-tests.md:15`).

**Current guidance**
- **Fakes.** "Prefer fakes to mocks" (Strongly recommended) (https://developer.android.com/topic/architecture/recommendations, V). "Fakes don't require a mocking framework and are lightweight. They are preferred" (https://developer.android.com/training/testing/fundamentals/test-doubles, V).
  - NIA "notably does not use any mocking libraries" and uses `Test*Repository` doubles (V).
- **StateFlow conflation.** "If values are set in a `StateFlow` rapidly, collectors of that `StateFlow` are not guaranteed to receive all intermediate values, only the most recent one." "We generally recommend treating `StateFlow` as a data holder and asserting on its `value` property."
  - `stateIn(WhileSubscribed)` needs a collector, e.g. `backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) { collect {} }`.
  - Source: https://developer.android.com/kotlin/flow/test (V).
  - Recommendations: "Assert on the `value` property whenever possible" (Strongly recommended, V).
- **Dispatchers.** `runTest` uses `StandardTestDispatcher`. `UnconfinedTestDispatcher` "can be a good choice for simple tests". The `MainDispatcherRule` sample defaults to Unconfined. All test dispatchers "must all share the same scheduler" (https://developer.android.com/kotlin/coroutines/test, V).
- **Naming.**
  - Kotlin: "In tests (and only in tests), you can use method names with spaces enclosed in backticks… only supported by Android runtime from API level 30. Underscores in method names are also allowed in test code" (https://kotlinlang.org/docs/coding-conventions.html, V).
  - Android style guide: "Underscores are permitted to appear in test function names", e.g. `pop_emptyStack` (https://developer.android.com/kotlin/style-guide, V).
- **Screenshot testing.**
  - Compose Preview Screenshot Testing is still alpha (0.0.1-alpha16), and its standalone plugin setup is now deprecated in favor of AGP test suites (AGP 9.5.0-alpha03+) (https://developer.android.com/studio/preview/compose-screenshot-testing, V).
  - Roborazzi 1.76.0 (2026-09-29) is active; NIA uses it (V).
  - Paparazzi has shipped only 2.0 alphas since 1.3.5 (2024-11-07) (V).
- **Turbine.** 1.2.1 (2025-06-11, V).

**Gap / recommendation**
- **KEEP (must):** fakes over mocks, `runTest` not `runBlocking`, test doubles in `src/test` and never in `main`, `MainDispatcherRule` in ViewModel tests.
- **CHANGE (default):** ViewModel tests assert on `uiState.value` after `advanceUntilIdle()`. Use Turbine only for genuine streams (non-conflated flows, event sequences). Rewrite the template so it doesn't depend on observing a conflated intermediate state.
- **CHANGE:** names. Pick one convention per project; the toolkit currently has two. Default to backtick sentence names in JVM unit tests and `subject_condition_expected` in instrumented tests (backticks need API 30+). **OPTIONAL** camelCase if the project uses it.
- **ADD (default):**
  - Compose UI tests for stateless screen content with fake state, using semantics matchers and `testTag` only where no semantic matcher exists.
  - `enableAccessibilityChecks()` (§11).
  - Fakes named `Fake*` (Google's naming recommendation, Optional) or follow the project's convention (`Test*` in NIA).
- **OPTIONAL:** screenshot tests (Roborazzi as the default if adopted; open decision).
- **CHANGE:** `ErrorEventBus.drain()` disappears with the bus. Without a global object there is no cross-test leakage.

**Community split?** Turbine everywhere vs `value` assertions. JUnit4 vs JUnit5 (Android instrumentation is JUnit4-centric).

**Confidence:** High.

---

## 15. Tooling, build and release

**Toolkit today:** no lint, Detekt, ktlint, ktfmt or Compose lint config; no baseline profiles, R8 keep-rule guidance or CI. The only mentions are `./gradlew ktlintCheck / test / lintDebug` in a checklist (`.claude/skills/data-feature/SKILL.md:252`) and `./gradlew assembleDebug` after scaffolding (`.claude/commands/new-screen.md:159`). Signing, ProGuard and CI are "out of scope" (`.claude/commands/init-project.md:26`).

**Current guidance**
- **Detekt.** The stable release is 1.23.8 (2025-02-20), built against Kotlin 2.0.21. 2.0.0-alpha.6 (2026-08-04) is built against Kotlin 2.4.10 under the new `dev.detekt` id. The README calls 2.0 "recommended… but it is still available only as a pre-release" (https://github.com/detekt/detekt, V).
- **ktlint.** 1.8.0 (2025-11-14). The project left Pinterest, and from 2.0 the coordinates are `io.github.ktlint` (https://github.com/ktlint/ktlint/releases, V).
- **ktfmt.** v0.65 (2026-10-07), monthly cadence (V). **Spotless** is active (2026-10-08, V).
- **Compose lint.**
  - slackhq/compose-lints 1.6.1 (2026-10-07) ships Android Lint rules: `ComposeModifierMissing`, `ComposeModifierWithoutDefault`, `ComposeParameterOrder`, `ComposeViewModelForwarding`, `ComposeUnstableCollections`, `ComposePreviewPublic`, … (https://github.com/slackhq/compose-lints, V).
  - mrmans0n/compose-rules v0.6.8 (2026-10-09) is the maintained successor of twitter/compose-rules for ktlint and Detekt (V).
- **R8.** Full mode has been the default since AGP 8.0. AGP 9 adds strict full-mode keep rules and optimized resource shrinking by default (V).
- **Baseline profiles.** About 30% faster code execution from first launch, via the Baseline Profile Gradle plugin and Macrobenchmark (https://developer.android.com/topic/performance/baselineprofiles/overview, V).

**Gap / recommendation**
- **ADD (must):** before finishing, run the project's lint, formatter and static analysis (whatever exists, e.g. `lint`, `detekt`, `ktlintCheck` or `spotlessCheck`) and the affected unit tests. Fix the code; never disable a rule or add a baseline entry to get a clean run. This mirrors the iOS must.
- **ADD (default for new projects):** one formatter (ktlint or ktfmt, never both), Android Lint with compose-lints, and optionally Detekt. Detekt 2.0 is still pre-release, which is part of the open decision.
- **ADD (must):** when adding a library that uses reflection or serialization, add or verify its R8 keep rules and test a minified release build. **ADD (default):** a baseline profile module for production apps.
- **ADD:** CI and signing are follow-only, as in the iOS spec.

**Confidence:** High on tool facts. The tool choice is opinion.

---

## 16. Platform musts missing from the toolkit

**Toolkit today:** none of the items below appear. The scaffold pads the `NavHost` with `systemBarsPadding()` (`.claude/skills/project-scaffold/SKILL.md:266`), and every screen also applies Scaffold `paddingValues` (`.claude/rules/screens.md:96`). Inference: that double-pads, and it prevents content from drawing behind the bars.

**Current guidance**
- **Edge-to-edge.**
  - Enforced on Android 15 for apps targeting 35 (https://developer.android.com/about/versions/15/behavior-changes-15, V).
  - For apps targeting 36, `windowOptOutEdgeToEdgeEnforcement` "is deprecated and disabled" (https://developer.android.com/about/versions/16/behavior-changes-16, V).
  - Call `enableEdgeToEdge()` in `onCreate` (https://developer.android.com/develop/ui/compose/system/setup-e2e, V). Scaffold insets are your responsibility (§5).
- **Predictive back.** At targetSdk 36, system animations are on by default, and "`onBackPressed` is not called and `KeyEvent.KEYCODE_BACK` is not dispatched anymore" (behavior-changes-16, V). Use `BackHandler`/`PredictiveBackHandler` (https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture, V).
- **Large screens.**
  - At targetSdk 36, orientation, resizability and aspect-ratio restrictions are ignored on sw600dp+ displays.
  - The temporary opt-out `PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY` "won't apply when targeting API level 37" (behavior-changes-16, V).
  - Android 17 removes the opt-out (https://developer.android.com/about/versions/17/behavior-changes-17, V).
- **Android 17 (API 37), stable 2026-06-16** (https://android-developers.googleblog.com/2026/06/Android-17.html, V). For apps targeting 37:
  - Activities are no longer recreated by default for typical configuration changes, with `android:recreateOnConfigChanges` to opt back in. This is from the blog only; I didn't see it on the behavior-changes summary, so recheck.
  - Local network access is blocked by default.
  - Stricter memory limits apply.
- **Adaptive layouts.** `currentWindowAdaptiveInfo().windowSizeClass` breakpoints are compact <600dp, medium 600–840dp, and expanded 840–1200dp. `NavigationSuiteScaffold` and `ListDetailPaneScaffold` cover the canonical layouts, and size classes "are not intended for isTablet-type logic". material3-adaptive 1.3.0 (2026-08-12) is stable (https://developer.android.com/develop/ui/compose/layouts/adaptive/use-window-size-classes, V).
- **16 KB page size.** Apps targeting Android 15+ must support 16 KB pages on 64-bit devices; "Starting February 1, 2027", updates that don't can't be released (https://developer.android.com/guide/practices/page-sizes, V).
  - The original 2025-11-01 date came from the 2025 blog (S).
  - This only affects apps with native `.so` code (V).
- **Notifications.** `POST_NOTIFICATIONS` is a runtime permission on API 33+, and notifications are off by default for new installs (https://developer.android.com/develop/ui/views/notifications/notification-permission, V).

**Gap / recommendation**
- **ADD (must):** edge-to-edge.
  - `enableEdgeToEdge()` in the activity.
  - Apply insets once, through Scaffold `innerPadding` or `WindowInsets.safeDrawing`.
  - Never use a global `systemBarsPadding()` on the nav host.
  - Lists pass insets as `contentPadding`.
- **ADD (must):** back handling only through `BackHandler`/`PredictiveBackHandler` or navigation. Never override `onBackPressed`, and never use back callbacks for logging or business logic.
- **ADD (must):** no `screenOrientation` lock or `resizeableActivity=false` in new code. Layouts adapt via window size classes, never device-type checks. **ADD (default):** `NavigationSuiteScaffold` for top-level navigation in apps that target tablets or foldables.
- **ADD (must):** keep UI state in ViewModel or `rememberSaveable` so it survives both recreation and (at API 37) non-recreation. Don't rely on `onCreate` re-running after configuration changes (inference from the Android 17 change).
- **ADD (must):** runtime permissions requested in context through `rememberLauncherForActivityResult(RequestPermission())`, with a handled denial path. `POST_NOTIFICATIONS` is requested before the first notification on API 33+.
- **ADD (must):** native libraries are 16 KB aligned (check with the APK Analyzer or `zipalign -P 16`) when the app ships `.so` files.
- **ADD (default):** the targetSdk tracks the Play deadline. Raising it is a separate, asked-for change that reviews the behavior changes listed above.

**Confidence:** High, except the Android 17 configuration-change item (blog only).

---

## 17. Agent-file mechanics

**Toolkit today**
- **CLAUDE.md** is 223 lines, mixing project facts, conventions and long code samples.
- **Path-scoped rules.** 25 `.claude/rules/*.md`, each with `name`, `description` and `paths` frontmatter. Three of them glob `**/*.kt` or `**/presentation/**/*.kt` (`folder-structure.md:5`, `logging.md:5`, `theming-and-tokens.md:5`).
- **Skills.** 9 skills, mostly "primitives" holding verbatim code to copy. `audit-branch` uses `context: fork` and says "Enter plan mode immediately… Call `ExitPlanMode`" (`.claude/skills/audit-branch/SKILL.md:9,14,179`).
- **Commands.** 7, several of which are thin wrappers around skills (`.claude/commands/audit-branch.md`).
- **Tool-specific instructions.** `AskUserQuestion` is required in commands (`.claude/commands/new-screen.md:27`). `/update-docs` writes to `~/.claude/projects/<slug>/memory/` (`.claude/commands/update-docs.md:69`).
- **Placeholders** are substituted by `sed` or by `/init-project` (`README.md:86-102`).
- **External dependency.** README says a separate `tech-android` skill is "treated as an external dependency" (`README.md:161`), but it isn't included or linked.
- **No LICENSE file** in the repo.

**Current guidance**
- **Rules.** "`paths` is the only field Claude Code reads from a rule; any other field is ignored." A path-scoped rule loads when Claude reads, writes or edits a matching file (https://code.claude.com/docs/en/memory, V).
- **Commands are skills.** "Custom commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy`" (https://code.claude.com/docs/en/skills, V).
- **Skill frontmatter.** `context: fork` runs the skill in a subagent that "doesn't see your conversation history". The page also recommends "Keep `SKILL.md` under 500 lines" (V).
- **Forked skills and plan mode.** Subagents lose `AskUserQuestion` and `EnterPlanMode`, and `ExitPlanMode` too unless their `permissionMode` is `plan` (https://code.claude.com/docs/en/sub-agents, V). Inference: the forked audit skill cannot enter plan mode as instructed.
- **Agent Skills spec** (https://agentskills.io/specification, V):
  - `name` is 1–64 lowercase characters and must match the directory name; `description` is required and at most 1024 characters.
  - `allowed-tools` is "Experimental".
  - Progressive disclosure: references are one level deep, and `SKILL.md` stays under 500 lines.
  - The client showcase lists Claude Code, Codex, OpenCode, Cursor, Gemini CLI, Copilot and others (https://agentskills.io/clients, V).
- **AGENTS.md** is "stewarded by the Agentic AI Foundation under the Linux Foundation" (https://agents.md, V).

**Gap / recommendation**
- **Carry over:** rule content only, rewritten. Path globs, `context: fork`, commands, `AskUserQuestion`, plan mode, memory paths and placeholders are Claude-specific or project-specific. Per the iOS spec, none belongs in a portable `android/` skill.
- **CHANGE:** the format.
  - One `android/SKILL.md` (frontmatter `name: android-guidelines` and a description naming Kotlin/Android/Compose work) with must rules inline.
  - Reference files per area: architecture-and-di, ui-compose, navigation, data-and-networking, concurrency, testing, tooling-and-build, platform-and-accessibility, logging-and-privacy.
  - Path scoping becomes an index line ("open before writing a ViewModel").
- **DROP:** verbatim "primitives" code skills. Short canonical snippets in references are fine. Full scaffolds belong in a template repo if ever wanted (a non-goal, as in iOS).
- **DROP:** project facts in the skill (commit message format, ticket prefix, "PRs go to `dev`", "never commit automatically"). Those belong in each project's AGENTS.md or CLAUDE.md, or the user's global file.
- **ADD:** the same "must vs default, project conventions win" framing as `ios/SKILL.md`.
- **License (U):** the repo has no LICENSE file, so by default no reuse rights are granted. Write the Android rules from primary sources, as was done for iOS, and copy no text or code from the toolkit.

**Confidence:** High.

---

## 18. Additional findings

- **Constants.** `presentation/utils/Constants.kt` ships `EMPTY_STRING`, `SPACE_STRING`, `DOT_STRING`, `ABSOLUTE_WEIGHT`, `DEFAULT_OPACITY` (`.claude/skills/misc-primitives/SKILL.md:190-209`). "No inline magic literals in the class body" applies even to `300L` (`.claude/rules/constants-in-viewmodels.md:40`). Naming `""` adds indirection without meaning (Community opinion, inference). **DROP** as a must; **KEEP** "name values that carry meaning (timeouts, limits)".
- **Extension and utils file naming.** One receiver type per file, `<Type>Ext.kt`, no `*Utils.kt`, and "never `private`" file-level extensions (`.claude/rules/extensions.md:16-47`). These are team preferences. Kotlin's conventions allow grouping related top-level functions in one file named for its content (V). **OPTIONAL.**
- **Folder naming.** `domain/model/response/` holds domain models while `.claude/rules/models.md:51` bans "Response" naming. That is confusing; use `domain/model/<feature>/`.
- **Single Activity.** `MainActivity` takes `MainViewModel` and passes it into `MainScreen` (`.claude/skills/project-scaffold/SKILL.md:178-188`), which is fine. It is a `ComponentActivity`, so the per-app language backport (which needs `AppCompatActivity`) won't work without changes (V, §11).
- **Dialogs.** Every dialog must use `<DIALOG>` with a 62% width (`.claude/skills/component-primitives/SKILL.md:94`), and `usePlatformDefaultWidth = false` without `decorFitsSystemWindows = false`, which the `DialogProperties` KDoc recommends pairing to avoid IME glitches (androidx source, V). **CHANGE:** prefer M3 `AlertDialog` / `BasicAlertDialog` with the platform width, and use custom sizing only via window size class.
- **Audit scope.** `/audit-branch` audits only `app/src/main/**/*.kt` (`.claude/skills/audit-branch/SKILL.md:33`), skipping tests, resources, manifests and Gradle files, where many platform musts live.

---

## Candidate criteria (accept or reject each)

Each item is a verifiable rule for the Android skill. "Must" items apply to every project. "Default" items apply only where the project has no convention. Verifications assume `rg` at the module root.

**Toolchain and build**
- [ ] **A1.** (Must) The agent never changes Kotlin, AGP, Gradle, compileSdk, targetSdk or minSdk without asking. Verify: the diff touches none of `libs.versions.toml` version keys, `gradle-wrapper.properties` or `*Sdk =` lines unless the task asked for it.
- [ ] **A2.** (Must) Dependencies live in the version catalog. Verify: `rg '"[a-z0-9.-]+:[a-z0-9.-]+:[0-9]' --glob '*.gradle.kts'` returns nothing.
- [ ] **A3.** (Default) New projects use compileSdk and targetSdk 36 and minSdk 23 or higher. Verify: the convention plugin or `build.gradle.kts` values.
- [ ] **A4.** (Default) Shared build logic lives in `build-logic/` convention plugins once there are two or more modules. Verify: `build-logic/convention` exists and modules apply its plugin ids.
- [ ] **A5.** (Default) Explicit backing fields are used only on Kotlin 2.4+. Verify: `rg 'Xexplicit-backing-fields'` returns nothing, and on Kotlin < 2.4 no `field =` property syntax appears.

**Architecture and state**
- [ ] **A6.** (Must) `MutableStateFlow`/`MutableState` are never exposed publicly from a ViewModel. Verify: `rg 'val \w+: MutableStateFlow' --glob '*ViewModel.kt'` shows only `private` declarations.
- [ ] **A7.** (Must) No async work in a ViewModel `init` or constructor. Verify: `rg -U 'init \{[^}]*launch' --glob '*ViewModel.kt'` returns nothing.
- [ ] **A8.** (Default) Stream-derived UI state uses `stateIn(viewModelScope, WhileSubscribed(5_000), …)`. Verify: ViewModels that inject a repository returning `Flow` call `stateIn`.
- [ ] **A9.** (Default) One public ViewModel function per user action. A project's `handleEvent` convention wins, and then its event type is a `sealed interface`. Verify: `rg '^(?!sealed ).*interface \w+Event'` returns nothing.
- [ ] **A10.** (Must) ViewModels don't send one-off events to the UI through `Channel` or `SharedFlow`. Messages and navigation outcomes are UI state that the UI clears. Verify: `rg 'Channel<|MutableSharedFlow<' --glob '*ViewModel.kt'` returns nothing in new code.
- [ ] **A11.** (Must) ViewModels take no `Context`, `Activity` or `Resources`, and are not `AndroidViewModel`. Verify: `rg 'AndroidViewModel|Context' --glob '*ViewModel.kt'` returns nothing.

**Compose UI**
- [ ] **A12.** (Must) UI collects flows with `collectAsStateWithLifecycle()`. Verify: `rg '\.collectAsState\(' --glob '*.kt'` returns nothing.
- [ ] **A13.** (Must) Composables below the screen entry receive state and lambdas, never a ViewModel, `NavController` or back stack. Verify: `rg 'NavController|ViewModel' ` in component files and screen content returns nothing.
- [ ] **A14.** (Must) Every UI-emitting reusable composable takes `modifier: Modifier = Modifier` as its first optional parameter and applies it to the root. Verify: compose-lints `ComposeModifierMissing`/`ComposeModifierWithoutDefault`/`ComposeParameterOrder` report nothing.
- [ ] **A15.** (Must) `LazyColumn`/`LazyRow` items pass a stable `key`. Verify: `rg 'items\((?![^)]*key)'` in changed files returns nothing.
- [ ] **A16.** (Must) Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`, and effects that call lambda params use `rememberUpdatedState`. Verify: no new `LifecycleEventObserver` in composables.
- [ ] **A17.** (Default) Reusable components and screen content have previews covering light/dark and at least one large font scale. Verify: `@PreviewLightDark` or the project wrapper, plus `@PreviewFontScales`, on screen content.
- [ ] **A18.** (Default) Screen content is not `private`, so UI and screenshot tests can call it. Verify: `rg 'private fun \w+ScreenContent'` returns nothing.

**Navigation**
- [ ] **A19.** (Must) Routes are `@Serializable` types (`data object`/`data class`, or `NavKey` on Nav3), and navigation arguments are read with `toRoute<T>()` or from the key. Verify: `rg 'route = "'` returns nothing in new code.
- [ ] **A20.** (Default) New Compose projects use Navigation 3. Existing Nav2 projects stay on Nav2 unless asked. Verify: `androidx.navigation3` in the catalog for new projects.

**Data, domain, DI, concurrency**
- [ ] **A21.** (Must) Network DTOs and client types never appear in domain or UI signatures. Verify: `rg 'retrofit2|com.apollographql|Dto\b'` under domain and UI packages returns nothing.
- [ ] **A22.** (Must) Expected data failures are typed (a sealed error or typed exceptions). Raw exception or server messages are never shown to users. Verify: `rg 'e\.message|errorBody\(\)' ` feeds no UI text.
- [ ] **A23.** (Must) `CancellationException` is never swallowed. Verify: every `catch (e: Exception)`/`Throwable` in suspend code rethrows cancellation (`rg -U 'catch \(e: (Exception|Throwable)\)'` reviewed).
- [ ] **A24.** (Must) Dispatchers are injected, not hard-coded, outside DI modules. Verify: `rg 'Dispatchers\.(IO|Default)' --glob '!**/di/**'` returns nothing in main code.
- [ ] **A25.** (Must) No `GlobalScope`. Verify: `rg GlobalScope` returns nothing.
- [ ] **A26.** (Default) Use cases exist only for shared or complex logic, are named `VerbNounUseCase`, expose one `operator fun invoke`, and are main-safe. Verify: no use case whose body is a single repository call.
- [ ] **A27.** (Default) Hilt with constructor injection, `@Binds` for interfaces, and scopes only for shared state. Verify: `rg '@Provides' ` shows no interface-to-impl bindings.
- [ ] **A28.** (Default) Paging flows are exposed separately from UiState and `cachedIn(viewModelScope)`. Verify: `rg 'PagingData' --glob '*UiState*'` returns nothing.

**Strings, accessibility, theming**
- [ ] **A29.** (Must) No hard-coded user-visible strings in composables. Verify: Android Lint `HardcodedText` plus `rg 'Text\("'` return nothing.
- [ ] **A30.** (Must) Numbers, currency and dates are formatted locale-aware. Verify: `rg '"\$\$\{|"\$\{.*/ 100'` returns nothing.
- [ ] **A31.** (Must) Touch targets are at least 48dp, and every icon-only control has a content description or click label. Verify: `enableAccessibilityChecks()` UI tests pass.
- [ ] **A32.** (Must) No literal colors in UI code. Verify: `rg 'Color\(0x|Color\.(Red|Blue|Black|White|Gray)'` outside the theme package returns nothing.
- [ ] **A33.** (Must) No `androidx.compose.material.icons` in new code. Verify: `rg 'material.icons'` returns nothing in changed files.

**Platform**
- [ ] **A34.** (Must) Edge-to-edge is enabled, and insets are applied once. Verify: `enableEdgeToEdge()` in the activity, and `rg 'systemBarsPadding\(\)'` is not applied to a NavHost or root.
- [ ] **A35.** (Must) No `onBackPressed` overrides; back goes through `BackHandler`/navigation. Verify: `rg 'override fun onBackPressed'` returns nothing.
- [ ] **A36.** (Must) No orientation locks or `resizeableActivity="false"` in new code. Verify: the manifest diff.
- [ ] **A37.** (Must) Runtime permissions, including `POST_NOTIFICATIONS` on API 33+, are requested in context with a handled denial. Verify: each `<uses-permission>` of a dangerous permission has a `RequestPermission` call site.
- [ ] **A38.** (Must) Apps that ship native libraries support 16 KB pages. Verify: APK Analyzer or `zipalign -c -P 16` passes.

**Testing and tooling**
- [ ] **A39.** (Must) Fakes over mocking libraries in new tests. Verify: `rg 'mockk|mockito'` shows no new imports.
- [ ] **A40.** (Default) ViewModel tests assert `uiState.value` after `advanceUntilIdle()`, using Turbine only for non-conflated streams. Verify: reviewed in the test diff.
- [ ] **A41.** (Default) One `enableAccessibilityChecks()` Compose UI test per main screen. Verify: `rg enableAccessibilityChecks androidTest`.
- [ ] **A42.** (Must) Before finishing, the agent runs the project's lint, formatter and static analysis plus the affected unit tests, and fixes findings without disabling rules or adding baselines. Verify: the commands' exit codes in the session log.

**Logging**
- [ ] **A43.** (Must) No `println`/`Log.*` in production code; use the project's logger. No PII, tokens or bodies are logged in release. Verify: `rg 'println\(|android\.util\.Log'` under `src/main` returns nothing.

**Skill format**
- [ ] **A44.** `android/SKILL.md` has valid Agent Skill frontmatter (`name: android-guidelines`, matching the directory) and stays under 150 lines, with the must/default framing and a reference index. Verify: `wc -l` and a frontmatter check.
- [ ] **A45.** No text or code is copied from the toolkit, and no client name appears. Verify: a manual diff review.

---

## Open decisions (for the grilling session)

1. **Event handling style.**
   - *One function per action:* matches Google docs and NIA, with no event types.
   - *`handleEvent(sealed Event)`:* one entry point and easy logging, at the cost of more types and boilerplate.
2. **Navigation library for new projects.**
   - *Navigation 3:* Google's current recommendation, stable 1.2, owns the back stack, adaptive scenes.
   - *Navigation Compose 2.x:* familiar and stable, but in maintenance mode per the releases page (scope unclear).
3. **Error model.**
   - Kotlin `Result` plus typed exceptions.
   - A custom sealed `DataResult<T, E>`.
   - Exceptions caught in the ViewModel.
4. **App-wide messages.** Per-screen `userMessage` state only, or an injected app-level message state holder for cross-screen errors.
5. **Theme strategy.**
   - *Extend MaterialTheme:* M3 components stay correct, with one palette.
   - *Fully custom theme object:* brand control, but every component is wrapped.
   - *Toolkit hybrid:* two palettes; not recommended.
6. **Module default.**
   - Single `app` module.
   - App plus `core:*`.
   - Full NIA-style feature modules with API/impl splits.
7. **DI.** Hilt (Google-recommended, NIA), Koin (KMP, runtime, optional compiler plugin), or Metro (compiler plugin, 1.x).
8. **State exposure on Kotlin 2.3 projects.** `_uiState` plus `asStateFlow()`, or ask to upgrade to 2.4 for explicit backing fields.
9. **Delegates.** Allow `@ViewModelScoped` state delegates as an optional pattern, or ban them in favor of use cases and plain state holders.
10. **Formatter and static analysis.**
    - ktlint (1.8; 2.0 coming under new coordinates) or ktfmt.
    - Detekt 1.23 (old Kotlin) vs 2.0 alpha vs none.
    - compose-lints (Android Lint) vs compose-rules (ktlint/Detekt).
11. **Screenshot testing.** None, Roborazzi (active, used by NIA), or Compose Preview Screenshot Testing (alpha, setup in flux).
12. **Test naming.** Backtick sentences (JVM only), `subject_condition_expected`, or camelCase.
13. **Logging facade.** Timber (no release since 2021, still maintained), Kermit (KMP), or an in-house interface.
14. **minSdk floor.** 23 (Hilt, NIA), 24, 26 (`java.time` without desugaring), or higher. Tradeoff: reach vs desugaring and compat noise.
15. **Icons.** A central registry object vs plain `R.drawable` references.
16. **Previews.** Required for every composable vs only reusable components and screen content; private vs internal.

---

## Sources

Primary (Google, JetBrains, Gradle, AndroidX, official samples and library repos):
- https://kotlinlang.org/docs/releases.html
- https://kotlinlang.org/docs/whatsnew22.html
- https://kotlinlang.org/docs/whatsnew23.html
- https://kotlinlang.org/docs/whatsnew24.html
- https://kotlinlang.org/docs/properties.html
- https://kotlinlang.org/docs/annotations.html
- https://kotlinlang.org/docs/sealed-classes.html
- https://kotlinlang.org/docs/coding-conventions.html
- https://github.com/Kotlin/KEEP/blob/main/proposals/KEEP-0430-explicit-backing-fields.md
- https://youtrack.jetbrains.com/issue/KT-14663
- https://developer.android.com/build/releases/gradle-plugin
- https://developer.android.com/build/releases/past-releases/agp-9-0-0-release-notes
- https://developer.android.com/build/releases/past-releases/agp-8-0-0-release-notes
- https://gradle.org/releases/
- https://docs.gradle.org/9.0.0/release-notes.html
- https://docs.gradle.org/current/userguide/configuration_cache_enabling.html
- https://support.google.com/googleplay/android-developer/answer/11926878
- https://developer.android.com/google/play/requirements/target-sdk
- https://developer.android.com/about/versions/15/behavior-changes-15
- https://developer.android.com/about/versions/16/behavior-changes-16
- https://developer.android.com/about/versions/17/behavior-changes-17
- https://android-developers.googleblog.com/2026/06/Android-17.html
- https://developer.android.com/guide/practices/page-sizes
- https://developer.android.com/guide/topics/resources/app-languages
- https://developer.android.com/develop/ui/views/notifications/notification-permission
- https://developer.android.com/topic/modularization
- https://developer.android.com/topic/modularization/patterns
- https://developer.android.com/topic/architecture/recommendations
- https://developer.android.com/topic/architecture/ui-layer
- https://developer.android.com/topic/architecture/ui-layer/events
- https://developer.android.com/topic/architecture/ui-layer/stateholders
- https://developer.android.com/topic/architecture/ui-layer/state-production
- https://developer.android.com/topic/architecture/domain-layer
- https://developer.android.com/topic/architecture/data-layer
- https://developer.android.com/topic/architecture/data-layer/offline-first
- https://developer.android.com/topic/libraries/architecture/paging/v3-paged-data
- https://developer.android.com/topic/libraries/architecture/paging/v3-transform
- https://developer.android.com/jetpack/androidx/releases/paging
- https://developer.android.com/jetpack/androidx/releases/navigation
- https://developer.android.com/guide/navigation/design/type-safety
- https://developer.android.com/guide/navigation/use-graph/navigate
- https://developer.android.com/jetpack/androidx/releases/navigation3
- https://developer.android.com/guide/navigation/navigation-3
- https://developer.android.com/guide/navigation/navigation-3/basics
- https://developer.android.com/guide/navigation/navigation-3/save-state
- https://developer.android.com/guide/navigation/navigation-3/naventrydecorators
- https://developer.android.com/guide/navigation/navigation-3/return-results
- https://android-developers.googleblog.com/2025/11/jetpack-navigation-3-is-stable.html
- https://developer.android.com/training/dependency-injection/hilt-android
- https://developer.android.com/jetpack/androidx/releases/hilt
- https://dagger.dev/hilt/view-model
- https://github.com/google/dagger/releases
- https://developer.android.com/kotlin/coroutines/coroutines-best-practices
- https://developer.android.com/kotlin/coroutines/test
- https://developer.android.com/kotlin/flow/test
- https://developer.android.com/training/testing/fundamentals/test-doubles
- https://developer.android.com/kotlin/style-guide
- https://developer.android.com/develop/ui/compose/api-guidelines
- https://github.com/androidx/androidx/blob/androidx-main/compose/docs/compose-api-guidelines.md
- https://github.com/androidx/androidx/blob/androidx-main/compose/docs/compose-component-api-guidelines.md
- https://developer.android.com/develop/ui/compose/performance/stability
- https://developer.android.com/develop/ui/compose/performance/stability/fix
- https://developer.android.com/develop/ui/compose/performance/stability/diagnose
- https://developer.android.com/develop/ui/compose/performance/stability/strongskipping
- https://developer.android.com/topic/libraries/architecture/compose
- https://developer.android.com/jetpack/androidx/releases/lifecycle
- https://github.com/androidx/androidx/blob/androidx-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/platform/AndroidCompositionLocals.android.kt
- https://github.com/androidx/androidx/blob/androidx-main/compose/ui/ui/src/androidMain/kotlin/androidx/compose/ui/window/AndroidDialog.android.kt
- https://developer.android.com/develop/ui/compose/designsystems/custom
- https://developer.android.com/develop/ui/compose/designsystems/material3
- https://developer.android.com/jetpack/androidx/releases/compose-material3
- https://developer.android.com/develop/ui/compose/bom/bom-mapping
- https://developer.android.com/develop/ui/compose/graphics/images/material
- https://developer.android.com/develop/ui/compose/tooling/previews
- https://developer.android.com/studio/preview/compose-screenshot-testing
- https://developer.android.com/develop/ui/compose/accessibility/api-defaults
- https://developer.android.com/develop/ui/compose/accessibility/merging-clearing
- https://developer.android.com/develop/ui/compose/accessibility/testing
- https://developer.android.com/develop/ui/compose/system/setup-e2e
- https://developer.android.com/develop/ui/compose/system/material-insets
- https://developer.android.com/develop/ui/compose/system/insets
- https://developer.android.com/develop/ui/compose/layouts/adaptive
- https://developer.android.com/develop/ui/compose/layouts/adaptive/use-window-size-classes
- https://developer.android.com/jetpack/androidx/releases/compose-material3-adaptive
- https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture
- https://developer.android.com/develop/ui/compose/system/predictive-back-setup
- https://developer.android.com/topic/performance/baselineprofiles/overview
- https://github.com/android/nowinandroid
- https://github.com/lysine-dev/retrofit/blob/master/CHANGELOG.md
- https://github.com/lysine-dev/okhttp/blob/master/CHANGELOG.md
- https://github.com/ktorio/ktor/releases
- https://github.com/apollographql/apollo-kotlin/releases
- https://github.com/apollographql/apollo-kotlin/blob/main/docs/source/migration/4.0.mdx
- https://github.com/Kotlin/kotlinx.serialization/releases
- https://github.com/Kotlin/kotlinx.coroutines/releases
- https://github.com/cashapp/turbine/releases
- https://github.com/JakeWharton/timber
- https://github.com/detekt/detekt
- https://github.com/ktlint/ktlint/releases
- https://github.com/facebook/ktfmt/releases
- https://github.com/diffplug/spotless/releases
- https://github.com/InsertKoinIO/koin/releases
- https://github.com/InsertKoinIO/koin-compiler-plugin
- https://code.claude.com/docs/en/memory
- https://code.claude.com/docs/en/skills
- https://code.claude.com/docs/en/sub-agents
- https://code.claude.com/docs/en/tools-reference
- https://agentskills.io/specification
- https://agentskills.io/clients
- https://agents.md/

Community / secondary (opinion signals or third-party maintainers):
- https://manuelvivo.dev/viewmodel-events-antipatterns (author mirror of the Android Developers Medium post, which returned 403)
- https://github.com/slackhq/compose-lints
- https://github.com/mrmans0n/compose-rules
- https://github.com/takahirom/roborazzi/releases
- https://github.com/cashapp/paparazzi/releases
- https://github.com/ZacSweers/metro
- https://android-developers.googleblog.com/2025/05/prepare-play-apps-for-devices-with-16kb-page-size.html (search snippet only)
