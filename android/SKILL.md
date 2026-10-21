---
name: android-guidelines
description: "Kotlin/Android coding rules. Load before writing, changing or reviewing Kotlin or Android code: Jetpack Compose screens, ViewModels, navigation, repositories, networking, Hilt, Gradle or manifest changes, tests."
---

# Android guidelines

## How to apply
- **Must** rules apply in every project.
- **Default** rules apply only where the project has no established convention for that area. When a project convention (or its AGENTS.md/CLAUDE.md) conflicts with a default, the project wins.
- Infra (CI, signing, Play publishing, crash reporting) is follow-only: use what exists, never introduce or replace it.

## Must rules

### Toolchain and build
- Never change Kotlin, KSP, AGP, Gradle, minSdk, compileSdk or targetSdk without asking, including a Kotlin version pinned through the `buildscript` classpath.
  - Verify: unless the task asked for it, the diff touches no Kotlin, KSP, AGP or Gradle version keys, no `gradle-wrapper.properties` and no `*Sdk =` lines.
- When you raise targetSdk, handle that level's behavior changes in the same change.
- On AGP 9+, use AGP's built-in Kotlin: new modules don't apply `org.jetbrains.kotlin.android`. Use KSP; `com.android.legacy-kapt` only for a processor without KSP support. Compiler options go in `kotlin { compilerOptions }`, not `android.kotlinOptions`. Never add `android.builtInKotlin=false`, `android.newDsl=false` or `android.disallowKotlinSourceSets=false`.
  - Verify: grep build files and `gradle.properties` for `org.jetbrains.kotlin.android`, `kotlinOptions` and those three flags.
- Declare dependencies through the project's existing mechanism; don't hard-code versions in module build files when the project centralizes them elsewhere.
  - Verify: where the project centralizes versions, review new `rg '"[a-z0-9.-]+:[a-z0-9.-]+:[0-9]' --glob '*.gradle.kts'` hits for versions that bypass that mechanism.

### Modules
- Put new code in the module the structure implies. Never create a module without asking.

### Architecture and state
- `MutableStateFlow`/`MutableState` are never public. ViewModels take no `Context`, `Activity` or `Resources` and are not `AndroidViewModel`.
  - Verify: `rg 'val \w+: MutableStateFlow' --glob '*ViewModel.kt'` shows only private; `rg 'AndroidViewModel|: Context' --glob '*ViewModel.kt'` returns nothing.
- ViewModels send no one-off events through `Channel` or `SharedFlow`; outcomes are UI state that the UI clears.
  - Verify: `rg 'Channel<|MutableSharedFlow<' --glob '*ViewModel.kt'` returns nothing in new code.

### Navigation
- Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x). No string routes in new code.
  - Verify: `rg 'route = "'` returns nothing in new code.
- Screens expose navigation callbacks (`onBack`, `onOpenDetail: (Id) -> Unit`); the nav graph wires them. Screen composables never receive `NavController` or the back stack.
- On Navigation 3, pass `entryDecorators = listOf(rememberSaveableStateHolderNavEntryDecorator(), rememberViewModelStoreNavEntryDecorator())` to `NavDisplay`, so ViewModels and saved state are scoped per entry. The list replaces the defaults, so name both.

### Data and networking
- Never swallow `CancellationException`. Catch serialization failures and map them to a typed error (`DataError.Serialization`). Treat a 204 as success. Raw `e.message` or `errorBody()` text never reaches users: the UI maps `DataError` to string resources.
  - Verify: review every `catch (e: Exception|Throwable)` (`rg -U 'catch \(e: (Exception|Throwable)\)'`); `rg 'e\.message|errorBody\(\)'` feeds no UI text.
- DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer; don't introduce a second serializer or migrate without asking.
  - Verify: `rg 'retrofit2|Dto\b'` under domain and UI packages.
- No hard-coded base URLs or keys in source. No secrets in source or committed `gradle.properties`.

### DI
- Never introduce a second DI library or migrate away from Hilt without asking. A project already on another DI library keeps it.

### Concurrency
- Dispatchers are injected, not hard-coded outside DI modules. No `GlobalScope`.
  - Verify: `rg 'Dispatchers\.(IO|Default)' --glob '!**/di/**'` and `rg GlobalScope` in main code.

### Compose UI
- Collect with `collectAsStateWithLifecycle()`.
  - Verify: `rg '\.collectAsState\('`.
- Reusable UI composables take `modifier: Modifier = Modifier` as the first optional parameter and apply it to the root.
  - Verify: compose-lints `ComposeModifierMissing`, `ComposeModifierWithoutDefault`, `ComposeParameterOrder`.
- Composables below the screen entry receive state and lambdas, never a ViewModel, `NavController` or back stack.
  - Verify: compose-lints `ComposeViewModelForwarding`.
- Lazy list items pass a stable `key` and, for mixed item types, a `contentType`.
  - Verify: `rg -U -P 'items\((?![^)]*\bkey\s*=)'` in changed files returns nothing.
- Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. For keyed effects, read a lambda parameter through `rememberUpdatedState` when changing it must not restart the effect; use the lambda as a key when changing it should restart the effect.
  - Verify: no new `LifecycleEventObserver` in composables.
- With strong skipping (default since Kotlin 2.0.20), don't add `@Immutable`, `@Stable` or `ImmutableList` to collection-bearing models; preserve collection instances instead. Making an otherwise unstable parameter stable switches instance comparison to `equals`, which can walk the collection whenever a different instance arrives. Make an exception only when the stability contract is valid, the collection reference cannot be preserved and benchmarks show structural comparison beats recomposition. Don't change existing annotations or collection types merely to comply with this rule. Collection-free classes (e.g. theme tokens) may keep valid `@Immutable`.
- Custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.
- No experimental Compose APIs requiring an opt-in, alpha Compose APIs, new `@OptIn` annotations or module-wide opt-in flags without asking. Check the project's Compose version before using an API that needs a newer one.

### Theme and icons
- No literal colors (`Color(0x...)`, `Color.Red`) or hard-coded `.dp` design values in composables; values come from theme tokens. Repeated values become tokens.
  - Verify: `rg 'Color\(0x|Color\.(Red|Blue|Black|White|Gray)'` outside the theme package returns nothing; review `rg '[0-9]+\.dp'` hits there.
- No `androidx.compose.material.icons` (`Icons.*`) in new code.
  - Verify: `rg 'material.icons'` in changed files.

### Strings, localization, accessibility
- No hard-coded user-visible strings. Plurals use `pluralStringResource`. Sentences are whole strings with placeholders, never concatenated.
  - Verify: Android Lint `HardcodedText` plus `rg 'Text\("'`.
- Format numbers, currency and dates locale-aware.
  - Verify: review `rg 'String\.format\(|SimpleDateFormat\('` hits that build user-visible text.
- Touch targets are at least 48dp. Every icon-only control has a `contentDescription` from resources (`null` when decorative) or a click label.
  - Verify: UI tests with `enableAccessibilityChecks()` pass.

### Platform
- Enable edge-to-edge (`enableEdgeToEdge()`) and apply each inset once. When screens should draw behind system bars, don't apply system-bar padding to the whole NavHost or root; apply the required insets per screen or component. Root padding is valid when keeping all content within safe bounds is intentional.
  - Verify: the activity calls `enableEdgeToEdge()`; review `rg 'systemBarsPadding\(\)'` hits.
- Lists take insets through `contentPadding`, never `Modifier.padding`. `NavigationSuiteScaffold` and `ListDetailPaneScaffold` don't pass insets down: each screen applies its own, never `safeDrawingPadding` on the scaffold.
- Apply IME insets exactly once: `adjustResize` in the manifest, `imePadding()` before `verticalScroll`, no `imePadding()` under a Scaffold whose `contentWindowInsets` already include the IME.
- Handle back only through navigation (Navigation 3's built-in back) or `NavigationBackHandler` (`androidx.navigationevent`). Use `BackHandler`/`PredictiveBackHandler` only where the project already does. Never override `onBackPressed` or set `enableOnBackInvokedCallback="false"`.
  - Verify: `rg 'override fun onBackPressed'`.
- Never add `screenOrientation` locks or `resizeableActivity="false"` without asking. If a client asks for a lock, apply it on phones only. Layouts adapt through window size classes, never device-type checks.
  - Verify: `rg 'screenOrientation|resizeableActivity'` in manifests shows no new hits.
- UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
- Apps that ship native libraries support 16 KB pages.
  - Verify: APK Analyzer, or `zipalign -c -P 16 -v 4 app-release.apk`.

### Security and permissions
- Request runtime permissions (including `POST_NOTIFICATIONS` on API 33+) in context with handled denial: rationale when `shouldShowRequestPermissionRationale` says so, app settings after a permanent denial, never loop requests or cache grant state. Background location is a separate request after foreground is granted.
  - Verify: each dangerous `<uses-permission>` has a request call site with denial handling.
- Use the Photo Picker (`PickVisualMedia`) for user-selected photos and videos. Request `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only when broad library access is essential to core functionality and system pickers are insufficient; satisfy applicable Play declaration and review requirements. For a permission-based gallery on Android 14+ targeting API 34+, handle partial access and reselection with `READ_MEDIA_VISUAL_USER_SELECTED`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
  - Verify: `rg 'READ_MEDIA_|QUERY_ALL_PACKAGES|MANAGE_EXTERNAL_STORAGE|USE_EXACT_ALARM'` in manifests; every foreground `<service>` has a `foregroundServiceType`.
- Components are `exported="false"` unless another app needs them. `PendingIntent`s use `FLAG_IMMUTABLE`; a mutable one gets an explicit component. Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.
- Never launch an Intent taken from extras without validating it (`IntentSanitizer` or package/exported checks). `onNewIntent` calls `setIntent` and validates like `onCreate`.

### R8
- When adding a library that uses reflection or serialization, confirm it ships consumer rules or add verified narrow ones, and test a minified release build.
- Keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. Don't duplicate consumer rules the exact artifact/version already bundles: check its JAR or AAR (`META-INF/proguard/`, `META-INF/com.android.tools/`, `proguard.txt`) and add only missing, verified narrow rules. AndroidX, kotlinx.coroutines, kotlinx.serialization (except named companions), Retrofit 2.10+ and Gson 2.11+ bundle theirs. No rules for Activity and View classes. Under full mode `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.

### Configuration
- Read required config once at startup and fail fast with a clear message.

### Testing
- Test doubles live in test source sets, never in `main`. Use `runTest`, not `runBlocking`.

### Tooling
- Before finishing, run the project's formatter, linter and static analysis plus the affected unit tests, and fix findings. Never disable, suppress or baseline a finding to get a clean run. Never run `clean` to verify; `./gradlew help` or `--dry-run` checks configuration cheaply.
  - Verify: each command ran and exited 0.

### Logging
- No `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.
  - Verify: `rg 'println\(|\bLog\.(v|d|i|w|e|wtf)\('` under `src/main` returns nothing (`Log` priority constants are fine).

## Defaults (reference files)
- [references/build-and-modules.md](references/build-and-modules.md): open before setting SDK levels, creating a project, adding a module or a convention plugin.
- [references/environments.md](references/environments.md): open before touching flavors, build types, `buildConfigField` or release minify settings.
- [references/architecture-and-state.md](references/architecture-and-state.md): open before adding a screen, ViewModel, use case, delegate or UI message.
- [references/navigation.md](references/navigation.md): open before adding routes, navigation, results, adaptive layouts or back handling.
- [references/data-and-networking.md](references/data-and-networking.md): open before adding or changing repositories, API calls or error types.
- [references/di-and-concurrency.md](references/di-and-concurrency.md): open before adding Hilt modules, injecting dependencies or writing coroutine code.
- [references/compose-ui.md](references/compose-ui.md): open before building UI: screens, previews, theme tokens, icons, app language.
- [references/testing.md](references/testing.md): open before writing or changing tests, fakes or screenshot tests.
- [references/tooling-and-logging.md](references/tooling-and-logging.md): open before configuring lint or formatting, or adding logging.
