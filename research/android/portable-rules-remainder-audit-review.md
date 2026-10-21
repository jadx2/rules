# Independent review of `portable-rules-remainder-audit.md`

**Date:** 2026-10-21
**Status:** review of the audit report; no rule or spec edits applied.
**Review target:** `research/android/portable-rules-remainder-audit.md` (untracked at review time).
**Checkout reviewed:** `0bb5642` (merge of `07d1ff5`). The audit's recorded baseline is `0e8673a`.
**Relationship to earlier work:** separate from `portable-rules-evaluation-review.md`, `…-verdict.md` and the seven-proposal implementation. Those proposals were not reopened.

## Summary

The audit is directionally sound but needs corrections before it serves as an implementation handoff. Thirteen of fifteen IDs hold in substance, two are rejected (R04, R12), two citations are wrong (NavDisplay commit URL, OkHttp repository), and several proposed corrections over-scope (R06, R07, R11, O02). Severities for R02 and R03 are lowered.

**Conclusion: requires corrections** (see the last section for the exact list and what remains unresolved).

## Scope and method

- Read `android/SKILL.md` (130 lines), all nine `android/references/*.md` files (203 lines) and `specs/android-guidelines.md` (201 lines). Every quotation and line reference in the audit matches the local text.
- Re-fetched the cited primary sources and, where the cited page was silent, the underlying source (androidx main, AOSP AGP and AAPT2 sources, GitHub API). Released behavior is distinguished from main-branch code below.
- No Android project was compiled or run. The R07 counterexample is derived from documented semantics and generator source, not reproduced.

## Repository changes since the audit's baseline

`git diff 0e8673a..HEAD` touches `specs/android-guidelines.md` lines 19 and 185 (client-name wording) and `specs/ios-guidelines.md`. No Android rule text or line number changed, so the audit's locations are correct for `0bb5642`. The audit file itself is untracked.

## Verdicts

| ID | Verdict | Severity | Classification | Basis |
|---|---|---|---|---|
| R01 | Accept | High | Technical error | Released reference KDoc: "This requires API 34+ (Android U), and currently does not work on Robolectric." Declared `@RequiresApi(34)`; supported only on `AndroidComposeTestRule`. Under the rule's default host the check never runs, so the verify on `SKILL.md:80` passes vacuously. |
| R02 | Accept | Medium | Technical scope error | Broadcasts overview: system broadcasts from "highly privileged apps, such as Bluetooth and telephony" need `RECEIVER_EXPORTED`; a not-exported receiver gets "some system broadcasts and broadcasts from your app, but not broadcasts from the highly privileged apps". Silent failure, but only apps listening to those senders are affected: medium, not high. |
| R03 | Accept with changes | Low | Overbroad rule | Play lists permitted core purposes for both permissions, subject to declaration and review. The must lacks the "without asking" escape hatch that the orientation must on `SKILL.md:89` already uses. Smaller wording than the audit's. |
| R04 | Reject | None | House convention, accurate in context | The local sentence is a consequence of its stated premise (JUnit 4-only tools). The android-junit5 README says to add Vintage "if you also have JUnit 4 tests (e.g. for Robolectric)". JUnit 6 docs mark Vintage deprecated, which supports the JUnit 4 default. The audit argues against a universal claim the text does not make. |
| R05 | Accept | Medium | Internal contradiction | Navigation 3 1.2.0 (stable 2026-09-23) ships `ResultEventBus`. The guide's state-based model is `conflateAsState` on that bus; nav3-recipes `results/state` reads `LocalResultEventBus` and calls `conflateAsState`; `results/serializable` keeps the bus and adds `rememberSerializable`. The rule points at Google's state recipe and bans the API it is built on. The in-memory caution is correct. |
| R06 | Accept with changes | Low | Incomplete verification hint | The page-size guide checks ELF segments (`check_elf_alignment.sh` / `llvm-objdump`), APK packaging (`zipalign -c -P 16 -v 4`) and then a 16 KB device. Change "or" to both; no need to describe what each tool establishes. |
| R07 | Accept with changes | Low | Overbroad house convention | Evidence adequate once combined with default-rule semantics (see next section). A qualifier suffices; the audit's "permit narrow verified rules" text duplicates what `SKILL.md:105` already allows. |
| R08 | Accept | Medium | Internal contradiction | Type-safe routes are "available as of Navigation 2.8.0" for Navigation Compose and the Kotlin DSL; XML graphs use Safe Args. `NavigationBackHandler` is a `navigationevent-compose` function (1.0.0 notes); the predictive-back guide's View path is `OnBackPressedCallback`/`OnBackPressedDispatcher`. A Fragment/XML project has no permitted back mechanism under `SKILL.md:87`. |
| R09 | Accept with changes | Low | Overbroad | Custom-modifiers guide: the simplest custom modifier "combines other already defined modifier factories"; `Modifier.Node` is for "more custom behavior"; `composed {}` is "no longer recommended". |
| R10 | Accept with changes | Low | Internal inconsistency | Entry-point `hiltViewModel()` is the documented pattern; the assisted overload shipped in Hilt 1.2.0 (2024-02-21), the package moved in 1.3.0 (2025-09-10). Scope the audit line below the entry. |
| R11 | Accept with changes | Low | Overbroad must in letter | State-hoisting guide: internal UI element state "is acceptable", "usually the case of animation state"; plain state holders are remembered; `rememberSaveable` plus a `Saver` only when preservation is wanted. One qualifier fixes it; the audit's three-way split is more than a must line needs. |
| R12 | Reject | None | House convention | Google: "we generally recommend treating StateFlow as a data holder and asserting on its value property." The rule does not block loading-then-content tests: step the dispatcher and assert `.value` at each point. The audit's counter-evidence shows a permitted alternative, not an error. |
| O01 | Accept with changes | Low | Incomplete documentation | AGP 8.1+, off by default, `res/resources.properties` with `unqualifiedResLocale`, compileSdk 33+. `AppCompatActivity` is only for the in-app picker / back-compat path, as the audit says. |
| O02 | Accept with changes | Low | Internal ambiguity | square/okhttp README: HEADERS/BODY "have the potential to leak sensitive information such as 'Authorization' or 'Cookie' headers and the contents of request and response bodies"; `redactHeader()` exists. The smallest consistent fix scopes the prohibition to release logs; the audit's "define a test-data scope" is scope expansion. |
| O03 | Accept with changes | Low | Partly stale cross-reference | 1.4.0-beta01 removed every `ExperimentalMaterial3ExpressiveApi` API from the 1.4 line ("please switch to 1.5.0-alpha"); 1.5.0-alpha18 promoted the expressive theme and added a `Material3ExpressiveApi` annotation that needs no opt-in; no stable 1.5.0 exists (1.5.0-beta02 released 2026-10-21). The "1.5+" gate is exact; "opts in" should mean adoption, with the must applying only to APIs still marked experimental. |

## R07: does the general reflection evidence support the Activity/View finding?

Yes, once combined with what the default and generated rules actually do.

- **AGP default rules** (`proguard-common.txt`) keep View setters/getters and Activity `onClick` handlers with `-keepclassmembers`, which the keep-rule docs define as preserving members "only if R8 doesn't remove the class".
- **AAPT2's generator** (`frameworks/base/tools/aapt2/java/ProguardRules.cpp`) emits `-keep class X { <init>(); }` for manifest `application`/`activity`/`service`/`receiver`/`provider` entries, `-keep class X { <init>(android.content.Context, android.util.AttributeSet); }` for Views referenced from layouts (conditional on the layout resource staying live via `-if class **.R$layout`), and `-keepclassmembers` for `android:onClick` targets. The current Android documentation pages no longer describe this; the audit does not cite it.
- **Counterexample (not compiled).** A View subclass instantiated through `Class.forName(name)` where `name` comes from remote config, with a `(Context)` constructor: AAPT2 never sees the name, the member-only default rules do not retain the class, and R8 removes it, producing `ClassNotFoundException`. This is the dynamic-class-name pattern the keep-rule examples page documents. A second case is a View method invoked from native code (JNI upcall).
- **Conclusion.** The clause "No rules for Activity and View classes" is overbroad in letter but nearly always right in practice. Low severity; qualify rather than rewrite.

## Errors and weak points in the audit

**Incorrect references**
- The NavDisplay link pinned at commit `4d74181` returns 404 (github.com and raw). The claim itself holds on `androidx-main`: `entryDecorators` defaults to `listOf(rememberSaveableStateHolderNavEntryDecorator())` and is passed unchanged to `rememberDecoratedNavEntries`. Verified on main only, not a release tag.
- O02 cites `github.com/lysine-dev/okhttp`. The GitHub API reports it is not a fork of `square/okhttp`. The canonical README carries the same warnings and should be the citation.

**Incorrect or weak assumptions and conclusions**
- R04 attacks "JUnit 5 universally requires Vintage", which the local text never claims; the local sentence is conditional on the JUnit 4-only tool set it names.
- R12 quotes the kotlinx `UnconfinedTestDispatcher` example accurately ("no values will be skipped"), but omits the same KDoc's caveat that the dispatcher "can only be used reliably for testing functionality, not the specific order of actions".
- R03's exact-alarm note is internally inconsistent: it quotes the policy's "alarm or timer app" case and then tells implementers not to broaden. The local "alarm or calendar apps" omits "timer", which is the policy's own wording. Optional one-word alignment.
- R07 relies on "standard rules and manifest processing" without a source; see the section above.
- O03 misses the decisive facts: Expressive was removed from the 1.4 line, and 1.5 has no stable release yet.
- The Hilt "deprecated old-package overloads" guard is source-supported on `androidx-main` (`@Deprecated` with `ReplaceWith`), but the release notes carry no deprecation entry, so the version in which it shipped is unverified. Spec :197 states it as fact.

**Severity challenges**
- R01 high stands (silent false pass of the accessibility verification).
- R02 medium (silent, but narrow population of apps).
- R03 low (policy scope, with an "ask" pattern already present on the neighbouring must).

## "Checks that should not become false-positive findings", re-verified

- **Kotlin explicit backing fields:** "What's new in Kotlin 2.4.0" (2026-07-14) lists them as Stable, no opt-in. The properties page confirms the in-class smart cast to the backing-field type that `architecture-and-state.md:25–26` relies on (`uiState.update { }` inside the class). Not compiled.
- **Navigation decorators:** confirmed on `androidx-main`; the cited commit URL is wrong (above).
- **Hilt APIs:** 1.3.0 package move and 1.2.0 assisted `creationCallback` overload confirmed in release notes and source.
- **Tool versions:** ktlint latest release 1.8.0 (2025-11-14), repository now `ktlint/ktlint`; detekt v1.23.8 stable, v2.0.0-alpha.6 latest prerelease. No basis to call the existing defaults obsolete.
- **AGP option:** `android.disallowKotlinSourceSets` exists in `BooleanOption.kt`, default `true`, stage Experimental, enforced from AGP 10.0.

## Verification limits after this pass

- `navigation.md:10` ("the key does not reach `SavedStateHandle`") still has no primary contract source. The official Hilt recipe uses assisted injection without mentioning `SavedStateHandle`; only third-party articles report the `toRoute` crash. Not a defect; remains unverified.
- The Roborazzi 1.56.0 and Paparazzi 2.0.0-alpha05.1 earliest-version claims were not re-verified here.
- The page-size guide does not state that APK Analyzer checks ELF segment alignment specifically; the R06 wording below avoids asserting it.
- Nothing was compiled or run.

## Accepted replacements (current text → smallest exact replacement → spec line)

### R01
- `android/references/testing.md:9`
  - Current: `Write one Compose UI test per main screen against `ScreenContent` with `enableAccessibilityChecks()`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device.`
  - Replace with: `Write one Compose UI test per main screen against `ScreenContent`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device. Add `enableAccessibilityChecks()` only in instrumented tests on API 34+: it does not run on Robolectric.`
- `android/SKILL.md:80`
  - Current: `Verify: UI tests with `enableAccessibilityChecks()` pass.`
  - Replace with: `Verify: instrumented UI tests on API 34+ with `enableAccessibilityChecks()` pass.`
- Spec :157: mirror the first sentence. Spec :124: no change.

### R02
- `android/SKILL.md:100`, sentence: `Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.`
  - Replace with: `Dynamic receivers register with `RECEIVER_NOT_EXPORTED` when they only receive the app's own broadcasts; receivers for other apps' broadcasts or for system broadcasts sent by privileged apps (Bluetooth, telephony) use `RECEIVER_EXPORTED` in a separate receiver.`
- Spec :140: same sentence, same replacement.

### R03
- `android/SKILL.md:98`, fragment: `No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.`
  - Replace with: ``QUERY_ALL_PACKAGES` (declare `<queries>` instead) and `MANAGE_EXTERNAL_STORAGE` only when Play's policy lists the app's core function, and only after asking; `USE_EXACT_ALARM` only for alarm or calendar apps; every foreground service declares a `foregroundServiceType`.`
- Spec :139: same fragment. Optional in both: `alarm, timer or calendar apps`.

### R05
- `android/references/navigation.md:7`
  - Current: `Return results to a previous screen as state (Google's results-as-state recipe), unless the project already has a result mechanism. Avoid the in-memory `ResultEventBus`: it loses results on process death.`
  - Replace with: `Return results to a previous screen as state (Google's state-based result recipe on `ResultEventBus`), unless the project already has a result mechanism. The bus is in-memory: persist a result that must survive process death (`rememberSaveable`, `SavedStateHandle`, or the `NavKey`).`
- Spec :74
  - Current: `results returned to a previous screen travel as state (Google's results-as-state recipe), not through the in-memory `ResultEventBus`, which loses results on process death.`
  - Replace with: `results returned to a previous screen travel as state (Google's state-based result recipe on `ResultEventBus`); the bus is in-memory, so a result that must survive process death is persisted (`rememberSaveable`, `SavedStateHandle`, or the `NavKey`).`

### R06
- `android/SKILL.md:93`
  - Current: `Verify: APK Analyzer, or `zipalign -c -P 16 -v 4 app-release.apk`.`
  - Replace with: `Verify: ELF segment alignment with `check_elf_alignment.sh` or APK Analyzer, plus `zipalign -c -P 16 -v 4 app-release.apk` for the APK, then a run on a 16 KB emulator image.`
- Spec :135: no verify text; no change.

### R07
- `android/SKILL.md:105`, sentence: `No rules for Activity and View classes.`
  - Replace with: `No rules for Activity and View classes that the manifest or layout XML already reference.`
- Spec :145: same sentence.

### R08
- `android/SKILL.md:34`
  - Current: `Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x). No string routes in new code.`
  - Replace with: `Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on Navigation Compose 2.8+). No string routes in new code on those versions; older 2.x and Fragment/XML graphs keep their existing route or Safe Args style.`
- Spec :70: mirror.
- `android/SKILL.md:87`, first sentence
  - Current: `Handle back only through navigation (Navigation 3's built-in back) or `NavigationBackHandler` (`androidx.navigationevent`).`
  - Replace with: `Handle back only through navigation (Navigation 3's built-in back), `NavigationBackHandler` (`androidx.navigationevent`) in Compose, or `OnBackPressedCallback` in View/Fragment code.` (rest of the line unchanged)
- Spec :131: insert `, `OnBackPressedCallback` in View/Fragment code` after the `NavigationBackHandler` clause.
- Spec :69, :132 and `navigation.md:5`, `:30`: no change.

### R09
- `android/SKILL.md:65`
  - Current: `Custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.`
  - Replace with: `Custom modifier behavior uses `Modifier.Node`/`ModifierNodeElement`; a factory that only chains existing modifiers needs no Node. Never `Modifier.composed`.`
- Spec :102: mirror.

### R10
- `android/references/di-and-concurrency.md:11`
  - Current: `DI audit: flag composables that resolve dependencies directly.`
  - Replace with: `DI audit: flag composables below the screen entry that resolve dependencies directly; `hiltViewModel()` belongs only in the entry composable.`
- Spec :88: mirror.

### R11
- `android/SKILL.md:91`
  - Current: `UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.`
  - Replace with: `UI state that must survive configuration changes lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.`
- Spec :134: mirror.

### O01
- `android/references/compose-ui.md:21`
  - Current: `Support per-app language through `generateLocaleConfig` in apps with more than one locale, unless the project already configures locales another way.`
  - Replace with: `Support per-app language through `generateLocaleConfig` (AGP 8.1+, default locale in `res/resources.properties`) in apps with more than one locale, unless the project already configures locales another way.`
- Spec :125: mirror.

### O02
- `android/SKILL.md:118`, second sentence
  - Current: `No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.`
  - Replace with: `No personal data, tokens or full request/response bodies in release logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.`
- Spec :167: mirror.
- Note: this makes the existing debug carve-out explicit for all three items. Requiring `redactHeader("Authorization")` would tighten debug logs but is new scope; owner's call.

### O03
- `android/references/compose-ui.md:17`, last sentence
  - Current: `Use Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).`
  - Replace with: `Use Material 3 Expressive only when the project is on material3 1.5+ (the first line that ships it) and has chosen it; Expressive APIs still marked experimental fall under the experimental-API must.`
- Spec :115: mirror.

## Conclusion

**Requires corrections.** Before the audit becomes the implementation handoff:

1. Drop R04 and R12, or downgrade them to optional wording.
2. Fix the NavDisplay citation (404 at the pinned commit) and the OkHttp citation (non-canonical repository).
3. Set R02 to medium and R03 to low.
4. Replace the R06, R07, R11 and O02 corrections with the smaller wordings above.
5. Add to O03 that Expressive was removed from the 1.4 line and that 1.5 has no stable release.

With those edits, the thirteen replacements above are implementable as one separately approved change, with the spec lines updated in the same change.

**Unresolved after this pass:** the Navigation 3 key-to-`SavedStateHandle` claim on `navigation.md:10`; the Roborazzi/Paparazzi earliest-version claims; the release version of the old-package `hiltViewModel` deprecation; whether APK Analyzer checks ELF alignment specifically.

## Sources consulted

- Compose accessibility test API: https://developer.android.com/reference/kotlin/androidx/compose/ui/test/junit4/accessibility/package-summary
- Compose accessibility testing: https://developer.android.com/develop/ui/compose/accessibility/testing
- androidx source (main), `ComposeTestRuleExt.android.kt` in `compose/ui/ui-test-junit4-accessibility`
- Broadcasts overview: https://developer.android.com/develop/background-work/background-tasks/broadcasts
- Play policies: https://support.google.com/googleplay/android-developer/answer/10158779 , https://support.google.com/googleplay/android-developer/answer/10467955 , https://support.google.com/googleplay/android-developer/answer/13161072
- JUnit 6.1.3 overview: https://docs.junit.org/6.1.3/overview.html ; android-junit5: https://github.com/mannodermaus/android-junit5
- Navigation 3 return results: https://developer.android.com/guide/navigation/navigation-3/return-results ; releases: https://developer.android.com/jetpack/androidx/releases/navigation3 ; recipes: https://github.com/android/nav3-recipes (`results/state`, `results/serializable`, `results/event`)
- 16 KB page sizes: https://developer.android.com/guide/practices/page-sizes
- Keep rules: https://developer.android.com/topic/performance/app-optimization/keep-rules-overview , https://developer.android.com/topic/performance/app-optimization/keep-rule-examples , https://developer.android.com/topic/performance/app-optimization/add-keep-rules
- AGP default rules: https://android.googlesource.com/platform/tools/base/+/mirror-goog-studio-main/build-system/gradle-core/src/main/resources/com/android/build/gradle/proguard-common.txt
- AAPT2 keep-rule generator: https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/tools/aapt2/java/ProguardRules.cpp ; background: https://jakewharton.com/increased-accuracy-of-aapt2-keep-rules/
- Type-safe navigation: https://developer.android.com/guide/navigation/design/type-safety ; predictive back: https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture ; navigationevent releases: https://developer.android.com/jetpack/androidx/releases/navigationevent
- Custom modifiers: https://developer.android.com/develop/ui/compose/custom-modifiers
- Hilt releases: https://developer.android.com/jetpack/androidx/releases/hilt ; androidx source (main) `hilt-lifecycle-viewmodel-compose` and `hilt-navigation-compose` `HiltViewModel.kt`
- State hoisting: https://developer.android.com/develop/ui/compose/state-hoisting
- Testing flows: https://developer.android.com/kotlin/flow/test ; UnconfinedTestDispatcher: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-unconfined-test-dispatcher.html
- Per-app languages: https://developer.android.com/guide/topics/resources/app-languages
- OkHttp logging interceptor (canonical): https://github.com/square/okhttp/blob/master/okhttp-logging-interceptor/README.md
- Material 3 releases: https://developer.android.com/jetpack/androidx/releases/compose-material3 ; androidx source (main) `MaterialTheme.kt`
- Kotlin 2.4.0: https://kotlinlang.org/docs/whatsnew24.html ; properties: https://kotlinlang.org/docs/properties.html
- Nav3 Hilt arguments recipe: https://developer.android.com/guide/navigation/navigation-3/recipes/passingarguments
- AGP `BooleanOption.kt`: https://android.googlesource.com/platform/tools/base/+/mirror-goog-studio-main/build-system/gradle-core/src/main/java/com/android/build/gradle/options/BooleanOption.kt
- NavDisplay (main): https://github.com/androidx/androidx/blob/androidx-main/navigation3/navigation3-ui/src/commonMain/kotlin/androidx/navigation3/ui/NavDisplay.kt
- GitHub API: `ktlint/ktlint` and `detekt/detekt` releases; `lysine-dev/okhttp` fork status
