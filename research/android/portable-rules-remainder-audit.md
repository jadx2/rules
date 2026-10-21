# Remaining Android rules: independent audit findings

**Date:** 2026-10-21  
**Status:** findings for independent review; no rule or spec edits applied.  
**Audit baseline:** `0e8673a79b982f02cbc35bf7b00f237f50ae3016`  
**Final checkout observed during the audit:** `07d1ff52f15272058cc43cf55ffb177b22b521c4`  
**Relationship to earlier work:** separate from `portable-rules-evaluation-verdict.md` and the seven-proposal implementation.

This document records the preceding audit. Creating this file did not rerun the external checks. During that audit, the checkout advanced to `07d1ff5`; the intervening diff changed client names in the specs, not the Android rules or their line numbering. Locations below therefore apply to both observed snapshots.

## Instructions for the reviewing model

- Review this report critically. Do not treat a prior agent's verification label, an upstream recommendation, or this report's proposed correction as authoritative by itself.
- Recheck findings against the actual local text and cited primary sources. Distinguish a technical error from an intentional house convention, an incomplete explanation, or an unverified claim.
- For each R01–R12 and O01–O03, return **accept**, **reject**, or **accept with changes**, with evidence and exact local/spec references. Challenge severity and proposed scope where appropriate.
- Do not reopen the seven-proposal fix list unless a concrete interaction requires it. Some remaining findings concern previously untouched clauses on the same physical lines; those clauses are identified below.
- Do not modify files, add dependencies, redesign the architecture, or introduce unrelated topics. For accepted findings, propose the smallest wording change. Mark anything you cannot independently verify as unverified, not as a defect.

## Scope and method

Read all ten Android rule files: `android/SKILL.md` (130 lines) and all nine reference files (203 lines), plus `specs/android-guidelines.md` (201 lines). Checked local consistency and researched technical assertions through Android, AndroidX, Kotlin, library and policy sources.

This was a documentation/source audit. There is no Android application in this repository to compile or run. Code examples were read against API/source evidence; they were not compiled. Reading all files does not establish every cross-version combination as verified.

The earlier audit covered catalog/serializer choice, collection stability, callback freshness, insets, Photo Picker, consumer-rule bundling and the generic experimental-API wording. This pass covers the rest, including untouched clauses on lines 62, 98, 100 and 105 and interactions with the references.

## Summary

The references contain substantive issues, not just style differences. The strongest confirmed reference-file defect is the instruction to combine `enableAccessibilityChecks()` with Robolectric despite the API's explicit platform restriction.

| ID | Classification | Local location | Matching spec | Finding |
|---|---|---|---|---|
| R01 | Technical error; high priority | `android/references/testing.md:9`; `android/SKILL.md:80` | :157; :124 supplies the accessibility requirement | Accessibility checks require API 34+ and currently do not work on Robolectric. |
| R02 | Technical scope error; high priority | `android/SKILL.md:100` | :140 | Universal `RECEIVER_NOT_EXPORTED` can miss legitimate external or privileged-system broadcasts. |
| R03 | Policy scope error | `android/SKILL.md:98` | :139 | Blanket package-visibility/all-files bans exclude permitted core use cases. |
| R04 | Incorrect rationale | `android/references/testing.md:5` | :153 | JUnit 5 does not universally require Vintage. |
| R05 | Incorrect distinction/source framing | `android/references/navigation.md:7` | :74 | State-based result observation and persistence are separate; Google's current state-based recipe also uses `ResultEventBus`. |
| R06 | Insufficient verification instruction | `android/SKILL.md:92–93` | :135 | APK ZIP alignment alone does not establish complete 16 KB compatibility. |
| R07 | Overbroad technical prohibition; qualified evidence | `android/SKILL.md:105` | :145 | A class being an Activity/View subclass does not establish that all its reflective entry points are already kept. |
| R08 | Portability/scope correction | `android/SKILL.md:34,87`; `android/references/navigation.md:5,30` | :69–70, :131–132 | Universal route/back instructions assume Compose/newer APIs despite preserving legacy and Fragment/XML projects. |
| R09 | Overbroad implementation requirement | `android/SKILL.md:65` | :102 | Not every custom modifier needs a new `Modifier.Node` implementation. |
| R10 | Internal audit-rule inconsistency | `android/references/di-and-concurrency.md:11`; `android/references/navigation.md:21–26` | :88; compare :49 | The blanket composable-DI audit flags the documented entry-point `hiltViewModel()` pattern. |
| R11 | Overbroad state-ownership rule | `android/SKILL.md:91` | :134 | ViewModel and `rememberSaveable` are not exhaustive choices for all UI state. |
| R12 | Over-restrictive default/rationale | `android/references/testing.md:7` | :155 | Conflation does not make controlled intermediate-emission tests invalid. |

## Technical corrections

### R01 — Accessibility checks cannot use the stated Robolectric default

**Local text:** `android/references/testing.md:9`:

> Write one Compose UI test per main screen against `ScreenContent` with `enableAccessibilityChecks()`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device.

`android/SKILL.md:80` also makes passing tests with `enableAccessibilityChecks()` the accessibility verification hint. Spec :157 mirrors the combined testing recommendation.

**Primary evidence:** the API reference states:

> This requires API 34+ (Android U), and currently does not work on Robolectric.

It also says enabling the checks is currently supported for `AndroidComposeTestRule`; the API declaration carries `@RequiresApi(34)`.

Sources: [API reference](https://developer.android.com/reference/kotlin/androidx/compose/ui/test/junit4/accessibility/package-summary), [Compose accessibility testing](https://developer.android.com/develop/ui/compose/accessibility/testing).

**Impact:** the recommended default combines incompatible requirements. “Unless it needs a device” does not clearly resolve a paragraph that recommends this combination as the normal setup.

**Minimum correction:** separate ordinary Compose tests that can run on Robolectric from these automated accessibility checks, which need supported API 34+ instrumentation. Qualify the verification hint accordingly. Do not invent Robolectric support or imply that every ordinary Compose UI test requires a device.

### R02 — Dynamic receiver export flags depend on the sender

**Local text:** `android/SKILL.md:100`:

> Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.

**Primary evidence:** Android documents that receiving all system broadcasts, including broadcasts from highly privileged apps, requires `RECEIVER_EXPORTED`. It specifically identifies Bluetooth and telephony senders. The app-local case uses `RECEIVER_NOT_EXPORTED`.

Source: [Broadcasts overview — context-registered receivers](https://developer.android.com/develop/background-work/background-tasks/broadcasts).

**Impact:** applying the local rule universally can silently prevent expected broadcast delivery.

**Minimum correction:** choose the flag based on the expected sender. Default app-local receivers to not exported; use an appropriately secured exported receiver when required for external senders. Do not replace one blanket rule with “export every receiver,” and do not assume every system broadcast has the same sender identity.

The component-export and PendingIntent clauses on this line are separate; this finding does not reject them wholesale.

### R03 — Restricted permissions have legitimate exceptions

**Previously untouched local clauses:** `android/SKILL.md:98`:

> No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE` …

**Primary evidence:** Play's package-visibility policy permits qualifying core-purpose device search, antivirus, file-manager and browser use cases, subject to its requirements. The all-files policy also lists eligible core functionality, including file management, backup/restore and antivirus, with declaration/review requirements.

Sources: [`QUERY_ALL_PACKAGES` policy](https://support.google.com/googleplay/android-developer/answer/10158779), [all-files access policy](https://support.google.com/googleplay/android-developer/answer/10467955), [Android all-files access](https://developer.android.com/training/data-storage/manage-all-files).

**Impact:** an agent following the must literally would refuse otherwise legitimate features in qualifying apps.

**Minimum correction:** prefer narrower APIs and targeted package visibility; allow broad permissions only for justified core functionality when narrower alternatives cannot satisfy it and applicable policy/declaration requirements are met. Distinguish Android capability from Google Play distribution policy.

**Not included:** the already-reviewed Photo Picker clause. Do not broaden `USE_EXACT_ALARM` using generic exact-alarm guidance: its distinct restricted policy supports alarm/timer/calendar use cases and directs other use cases to evaluate `SCHEDULE_EXACT_ALARM`. [Exact alarm policy](https://support.google.com/googleplay/android-developer/answer/13161072)

### R04 — JUnit 5 does not inherently require Vintage

**Local rationale:** `android/references/testing.md:5`:

> … JUnit 5 forces the vintage engine and two test styles.

**Primary evidence:** JUnit Vintage runs JUnit 3/4 tests on the JUnit Platform. Jupiter has its own engine. Pure Jupiter tests do not require Vintage. Android's runner documentation separately describes instrumented JUnit 4 tests.

Sources: [JUnit architecture/engines](https://docs.junit.org/6.1.3/overview.html), [AndroidJUnitRunner](https://developer.android.com/training/testing/instrumented-tests/androidx-test-libraries/runner).

**Minimum correction:** retain JUnit 4 as a house default if desired, but explain the existing Android runner/rule ecosystem accurately. Vintage is relevant when executing legacy JUnit tests through the JUnit Platform, not a universal dependency of selecting JUnit 5 for JVM tests. Separate JVM and instrumentation configurations can also use different engines.

This finding does not claim all AndroidX, Hilt or Robolectric integrations have native Jupiter equivalents; that separate universal claim was not established.

### R05 — State-based observation is not persistence

**Local text:** `android/references/navigation.md:7`:

> Return results to a previous screen as state (Google's results-as-state recipe), unless the project already has a result mechanism. Avoid the in-memory `ResultEventBus`: it loses results on process death.

**Primary evidence:** the current Navigation 3 guide describes event-based observation with `ResultEffect` and state-based observation with `conflateAsState`, both using `ResultEventBus`. It explicitly warns:

> ResultEventBus is an in-memory communication bus. Results aren't saved across process death or activity recreation automatically.

The guide identifies saved-state mechanisms for persistence rather than claiming state-based observation alone supplies it.

Source: [Return results in Navigation 3](https://developer.android.com/guide/navigation/navigation-3/return-results).

**What remains correct:** the warning about an in-memory bus losing unsaved results.

**What needs correction:** the implied contrast between Google's state-based recipe and the bus, and any inference that merely representing or observing results as state solves restoration.

**Minimum correction:** separate observation style from persistence requirements. When results must survive recreation/process death, use an appropriate saved-state mechanism. This is not a recommendation to introduce or migrate to a new result bus.

### R06 — ZIP alignment is only part of 16 KB verification

**Local text:** `android/SKILL.md:92–93`:

> Apps that ship native libraries support 16 KB pages.
>
> Verify: APK Analyzer, or `zipalign -c -P 16 -v 4 app-release.apk`.

**Primary evidence:** Android's page-size guide distinguishes ELF segment alignment, APK packaging/ZIP alignment, and execution testing. Native dependencies supplied by SDKs also count; apps need not contain their own C++ source to be affected.

Source: [Support 16 KB page sizes](https://developer.android.com/guide/practices/page-sizes).

**Impact:** a successful `zipalign` command does not prove native ELF alignment or eliminate runtime assumptions about page size. The command is valid; treating it as a complete alternative verification is the problem.

**Minimum correction:** make the check cover ELF and packaging alignment and testing in a 16 KB environment. State what APK Analyzer and `zipalign` each establish rather than presenting them as interchangeable complete checks.

### R07 — Activity/View identity is not a universal keep-rule exemption

**Previously untouched local clause:** `android/SKILL.md:105`:

> No rules for Activity and View classes.

**Primary evidence:** R8 documentation explains that indirect uses can require explicit narrow rules and gives reflective class/constructor examples. Standard rules and manifest processing cover standard entry points; they do not establish coverage for every possible reflective member on every application subclass.

Sources: [R8 keep-rule overview](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview), [keep-rule examples](https://developer.android.com/topic/performance/app-optimization/keep-rule-examples).

**Evidence qualification:** the fetched example establishes the general reflective-access limitation; it was not an independently compiled Activity/View-specific reproduction. Applying that limitation to arbitrary application subclasses is the reasoning behind this finding. The reviewing model should challenge or strengthen this example, not treat a specific failing custom View as already reproduced.

**Minimum correction:** prohibit redundant blanket keeps for entries already covered by standard rules, while permitting narrow verified rules for additional reflective access that is not covered. Do not claim that every custom View requires manual rules.

This finding concerns the Activity/View clause, not the earlier consumer-rule bundling proposal or its library list.

## Scope and consistency corrections

### R08 — Navigation musts need a project/API boundary

**Local requirements:** `android/SKILL.md:34` requires serializable routes for Navigation 3 and “2.x,” with no string routes in new code. Line 87 allows navigation or `NavigationBackHandler`, with `BackHandler`/`PredictiveBackHandler` only in projects already using them. In contrast, `android/references/navigation.md:5` explicitly preserves existing Navigation Compose 2.x and Fragment/XML stacks.

**Primary evidence:** type-safe Navigation Compose/Kotlin DSL routes are available from Navigation **2.8.0**. Fragment/XML navigation has established mechanisms such as Safe Args. Predictive-back guidance also documents supported non-Compose handling through AndroidX callbacks/dispatchers. `NavigationBackHandler` is a Compose API in `androidx.navigationevent.compose`.

Sources: [type-safe routes](https://developer.android.com/guide/navigation/design/type-safety), [passing arguments](https://developer.android.com/guide/navigation/use-graph/pass-data), [predictive back](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture), [Navigation Event Compose API](https://developer.android.com/reference/kotlin/androidx/navigationevent/compose/package-summary).

**Impact:** the musts can force unsupported route APIs or a Compose-oriented mechanism into projects that the defaults promise not to migrate.

**Minimum correction:** scope serializable route instructions to the applicable Navigation Compose/Kotlin DSL version and Navigation 3 persistence setup. Preserve existing legacy/XML routing and supported non-Compose back handling. Keep migration/version changes permission-gated. There is no need to relax the preference against overriding deprecated `onBackPressed` merely to fix this scope problem.

### R09 — Not every custom modifier needs a Node

**Local text:** `android/SKILL.md:65`:

> Custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.

**Primary evidence:** Android's custom-modifier guide first recommends combining existing modifier factories when sufficient, then using `Modifier.Node` for additional low-level behavior. It says `Modifier.composed` is no longer recommended due to performance concerns.

Source: [Custom modifiers](https://developer.android.com/develop/ui/compose/custom-modifiers).

**Minimum correction:** reserve the Node requirement for implementing new low-level behavior. Permit simple factories that compose existing modifiers. Avoiding `Modifier.composed` in new code can remain; the problem is the universal Node mandate, not a need to restore `composed` as the default.

### R10 — The DI audit catches an allowed entry point

**Local text:** `android/references/di-and-concurrency.md:11`:

> DI audit: flag composables that resolve dependencies directly.

But `android/references/navigation.md:21–26` explicitly obtains a ViewModel using `hiltViewModel()` in the navigation entry. `android/references/compose-ui.md:6` and `android/SKILL.md:58` distinguish the entry boundary from reusable/content composables.

**Primary support:** Hilt's Compose integration documents ViewModel acquisition at this boundary. The current Hilt source has the assisted `creationCallback` overload used by the example.

Sources: [Hilt with Jetpack](https://developer.android.com/training/dependency-injection/hilt-jetpack), [current Hilt Compose source](https://github.com/androidx/androidx/blob/androidx-main/hilt/hilt-lifecycle-viewmodel-compose/src/main/kotlin/androidx/hilt/lifecycle/viewmodel/compose/HiltViewModel.kt).

**Minimum correction:** exempt designated screen/navigation entry points; flag dependency resolution below that boundary. This is primarily an internal consistency fix, not a proposal for another DI pattern.

### R11 — UI state has more than two valid owners

**Local text:** `android/SKILL.md:91`:

> UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.

**Primary evidence:** Android distinguishes UI-element state from screen/business state and documents local state and plain state holders. State is hoisted to a screen holder when its consumers or business logic require that scope; not every transient value needs restoration.

Sources: [Where to hoist state](https://developer.android.com/develop/ui/compose/state-hoisting), [state holders](https://developer.android.com/topic/architecture/ui-layer/stateholders), [Compose state](https://developer.android.com/develop/ui/compose/state).

**Minimum correction:** distinguish screen/business state, restoration-required state and disposable UI-element state. Keep the warning against relying on `onCreate` for every configuration change. Do not force `remember`-appropriate transient state into a ViewModel or saved-state mechanism.

### R12 — Conflation is not a prohibition on controlled emission tests

**Local text:** `android/references/testing.md:7`:

> Assert `StateFlow` on the final `uiState.value` or latest item, never an intermediate emission (conflation).

**Primary evidence:** Android says collectors are not guaranteed every intermediate value, recommends `.value` assertions generally, but also explicitly recognizes scenarios where collecting intermediate values is desirable. Kotlin's test dispatcher documentation demonstrates a controlled collector observing all updates with `UnconfinedTestDispatcher`.

Sources: [Testing flows on Android](https://developer.android.com/kotlin/flow/test), [UnconfinedTestDispatcher](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-unconfined-test-dispatcher.html).

**Minimum correction:** retain final/current-value assertions as the default, but allow deliberately controlled collection tests. Such a test must not be misrepresented as a production guarantee that every StateFlow emission will be delivered.

This is an over-restrictive default and rationale, not evidence that the repository's chosen ordinary assertion style is wrong.

## Smaller clarifications

### O01 — LocaleConfig setup prerequisites

**Location:** `android/references/compose-ui.md:21` → spec :125.

Automatic `generateLocaleConfig` is supported from AGP 8.1 and requires the default-locale setup using `resources.properties` and `unqualifiedResLocale`. The existing rule is a legitimate default but not a complete setup recipe.

**Suggested clarification:** name the version/default-locale prerequisites and preserve existing manual configuration. Do not unnecessarily require `AppCompatActivity` just to generate LocaleConfig; that belongs to the separate compatibility path for an in-app language picker.

Source: [Per-app language preferences](https://developer.android.com/guide/topics/resources/app-languages).

### O02 — Debug BODY logging is not a privacy exception

**Location:** `android/SKILL.md:118` → spec :167; related defaults in `android/references/tooling-and-logging.md:10–12`.

The line prohibits personal data, tokens and full bodies in logs, then says `HttpLoggingInterceptor` at `BODY` is debug-only. This creates ambiguity about whether debug is intended as an exception. The privacy prohibition is not explicitly restricted to production in the local text.

OkHttp warns that `HEADERS`/`BODY` can expose sensitive headers and request/response contents. Debug builds can still handle real credentials or personal data. Header redaction alone does not establish body safety.

**Suggested clarification:** make the all-build-types privacy boundary explicit. If retaining any BODY exception, define its allowed non-sensitive/test-data scope without silently weakening the existing no-full-body rule. Do not equate “debug-only” with “safe.”

Source: [OkHttp logging interceptor README](https://github.com/lysine-dev/okhttp/blob/main/okhttp-logging-interceptor/README.md).

### O03 — Expressive adoption versus experimental API opt-in

**Location:** `android/references/compose-ui.md:17` → spec :115.

“Material 3 Expressive … and opts in (see the experimental-API must)” should distinguish a project's adoption decision from compiler opt-in for a particular API. Previously verified Material 3 releases graduate specific Expressive APIs; not every API in the family remains experimental.

**Suggested clarification:** preserve the project decision, but check the installed version and the actual selected API's annotation rather than implying a blanket compiler opt-in. This is a cross-file wording interaction, not a request to reopen the earlier line-66 correction or research unrelated Compose features.

Source: [Material 3 release notes](https://developer.android.com/jetpack/androidx/releases/compose-material3).

## Coverage of the nine reference files

| Reference | Outcome of this pass |
|---|---|
| `build-and-modules.md` | No confirmed mandatory correction. Module structure is a house default. The basic API-26 `java.time` rationale has support. “Latest stable” still needs a compatible toolchain, not independently chosen incompatible versions. |
| `environments.md` | No confirmed defect in the flavor/build-type snippet or release optimization defaults. Existing-project exceptions are present. No build was performed. |
| `architecture-and-state.md` | Backing-field language support and the general narrowing mechanism are documented. Delegates, event dispatch and message-holder choices are house conventions. R11 concerns the related universal state-owner must. |
| `navigation.md` | R05 and R08 need attention; its entry-point example also exposes R10. Decorator replacement behavior and the Hilt API move have source support. |
| `data-and-networking.md` | Ktor-only-for-KMP and the cache/offline-first trigger are house choices, not technical limitations. Ktor supports Android/JVM too. Do not label these choices Android requirements or override established project conventions. |
| `di-and-concurrency.md` | R10 is a local consistency issue. Dispatcher injection, avoiding GlobalScope and the binding conventions otherwise have supporting guidance or are explicit preferences. |
| `compose-ui.md` | O01 and O03 are clarification candidates. Screen/content separation, preview APIs and theme-extension patterns have support; file placement and design-token preferences remain house choices. |
| `testing.md` | R01 is a concrete incompatibility; R04 and R12 correct rationale/scope. Screenshot tooling constraints have partial version-specific evidence, with limits below. |
| `tooling-and-logging.md` | The chosen stable tool versions are not presently obsolete. Timber is a house choice. O02 concerns the related logging must rather than replacing the logging framework. |

## Checks that should not become false-positive findings

- **Kotlin backing fields:** Kotlin 2.4 documentation and KEEP-0430 support stable explicit backing fields and narrowing to the backing-field type within the declaration's scope. The example should not be rejected simply because its public type is `StateFlow`. It was not compiled here. [Properties](https://kotlinlang.org/docs/properties.html#explicit-backing-fields), [KEEP-0430](https://raw.githubusercontent.com/Kotlin/KEEP/main/proposals/KEEP-0430-explicit-backing-fields.md)
- **Navigation decorators:** `NavDisplay` defaults to a saveable-state-holder decorator and passes the caller-supplied list directly to `rememberDecoratedNavEntries`; it does not append the default. [Source at `4d74181`](https://github.com/androidx/androidx/blob/4d74181ec5f92b49275c03d9715a965080bb319b/navigation3/navigation3-ui/src/commonMain/kotlin/androidx/navigation3/ui/NavDisplay.kt)
- **Hilt APIs:** the 1.3.0 package/artifact move, deprecated old-package overloads and assisted creation callback have source support. [Release notes](https://developer.android.com/jetpack/androidx/releases/hilt), [old-package source](https://github.com/androidx/androidx/blob/androidx-main/hilt/hilt-navigation-compose/src/main/java/androidx/hilt/navigation/compose/HiltViewModel.kt)
- **Tool versions:** release metadata checked during the audit showed ktlint `1.8.0` stable / `2.0.0-ALPHA-4` prerelease, and detekt `v1.23.8` stable / `v2.0.0-alpha.6` prerelease. No basis was found for declaring the existing stable-version defaults obsolete. [ktlint releases](https://github.com/ktlint/ktlint/releases), [detekt releases](https://github.com/detekt/detekt/releases)
- **AGP option:** `android.disallowKotlinSourceSets` is real. Primary source defines it as true by default when built-in Kotlin is enabled; false silently ignores those source sets. A ban on setting it false is a house safeguard, not an invalid-flag mistake. [AGP BooleanOption source](https://android.googlesource.com/platform/tools/base/+/mirror-goog-studio-main/build-system/gradle-core/src/main/java/com/android/build/gradle/options/BooleanOption.kt?format=TEXT)

## Verification limits

- No Android project was compiled or executed. Source-level agreement is not a reproduced runtime result, and code examples omit surrounding project configuration/imports.
- Roborazzi 1.56.0 and Paparazzi 2.0.0-alpha05.1 have release evidence for AGP 9 support; this pass did not establish exhaustively that they are the **earliest** compatible releases. Compose Preview Screenshot Testing's experimental status and different Gradle-task/IDE requirements were checked. [Roborazzi](https://github.com/takahirom/roborazzi/releases/tag/1.56.0), [Paparazzi](https://github.com/cashapp/paparazzi/releases/tag/2.0.0-alpha05.1), [Compose screenshot testing](https://developer.android.com/studio/preview/compose-screenshot-testing)
- The exact latest Kotlin/AGP compatibility matrix and every older-project API combination were not exhaustively verified. Do not turn compatibility caution into an assertion that the current latest stable pair is incompatible.
- The specific Navigation 3 key-to-`SavedStateHandle` injection behavior was not independently established from a direct implementation/contract in this pass. The Hilt callback signature was verified; this is not equivalent to proving all argument-restoration behavior.
- Current documentation and `main` sources are mutable. Not every dated provenance statement in the spec was fully recertified. Unverified points are not asserted as bugs. Some initial partial-fetch results were inconclusive; findings above use the subsequently retrieved evidence, notably the explicit accessibility API restriction.

## Conclusion for the next reviewer

The remaining rules should not receive blanket approval unchanged. Review the technical corrections first, then the scope/consistency issues, and keep optional clarifications distinct. Preserve useful existing conventions instead of treating every non-universal preference as a defect. Any implementation should be a separately approved change after this independent review.
