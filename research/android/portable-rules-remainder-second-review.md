# Independent second review of the remaining Android rules

**Date:** 2026-10-21  
**Status:** review findings only; no Android rule or spec edits applied.  
**Conclusion:** **requires corrections**.  
**Checkout reviewed:** `0bb5642977aadf8584abc7de23c0a80b856cf614`.  
**Review targets:** `research/android/portable-rules-remainder-audit.md` and `research/android/portable-rules-remainder-audit-review.md`. Neither was treated as an authority.  
**Relationship to earlier work:** the original seven-proposal review remains closed. Only concrete interactions with the remaining findings are discussed.

## Method and evidence limits

- Read `android/SKILL.md`, all nine `android/references/*.md` files, the matching spec and both review targets. Checked quotations and repository history/status locally.
- Primary-source retrieval was delegated to research agents. Follow-up retrievals addressed truncated pages, exact URLs, complete source files and published source JARs. The conclusions below use the corrected, complete-source results, not superseded partial-fetch conclusions.
- Compared released source where available with mutable `main` sources. A documentation recommendation is not automatically a platform requirement or a reason to discard a house convention.
- No Android project was compiled or run. R07 is a conditional source-derived counterexample, not a reproduced failure. No exact Robolectric runtime outcome is claimed.
- This document records that review; creating this Markdown file did not rerun the external checks. No raw retrieval archive is bundled here. A subsequent reviewer should independently retrieve the cited evidence.

## Verdict table

“Text override” means agreement with the finding or its severity, but not approval of the first review's exact replacement. Severities are impact judgments, not execution results.

| ID | First review verdict | Second review verdict | Severity: first → second | Agreement / override | One-line reason |
|---|---|---|---|---|---|
| R01 | Accept | Accept with changes | High → **Medium** | Override | The API restriction is documented; a vacuous pass is not established, and the local text already has a device exception. |
| R02 | Accept | Accept with changes | Medium → Medium | Text override | Export depends on the sender; separate receivers are needed when both flags are needed, not unconditionally. |
| R03 | Accept with changes | Accept with changes | Low → Low | Text override | Preserve narrower alternatives and applicable policy requirements; a listed app category alone is not authorization. |
| R04 | Reject | **Accept with changes** | None → **Low** | Override | JUnit 4 is a valid default; JUnit 5 does not necessarily force Vintage even in a mixed Android project. |
| R05 | Accept | Accept with changes | Medium → Medium | Text override | Google's recipe uses the bus, but adopting it reverses the local default and needs a provider/decorator absent from the mandated list. |
| R06 | Accept with changes | Accept with changes | Low → Low | Text override | Do not use an unverified APK Analyzer capability as an alternative to explicit ELF checking. |
| R07 | Accept with changes | Accept with changes | Low → Low | Text override | Manifest/XML references do not preserve every reflective constructor or member of a referenced class. |
| R08 | Accept | Accept with changes | Medium → Medium | Text override | Also scope the back-handling default and the route-verification grep. |
| R09 | Accept with changes | Accept with changes | Low → Low | Agree | Correctly separates chaining existing modifiers from implementing new behavior. |
| R10 | Accept with changes | Accept with changes | Low → Low | Minimal-text adjustment | Adding “below the screen entry” suffices; the extra `hiltViewModel()` sentence is redundant. |
| R11 | Accept with changes | Accept with changes | Low → Low | Agree | The configuration-survival qualifier removes the overreach without reorganizing state. |
| R12 | Reject | Reject | None → None | Agree, with qualification | A valid house convention, not technical impossibility; the audit's controlled example is not refuted either. |
| O01 | Accept with changes | Accept with changes | Low → Low | Text override | If documenting prerequisites, include compileSdk and the `unqualifiedResLocale` key. |
| O02 | Accept with changes | Accept with changes | Low → Low | **Reject proposed replacement** | Restricting the prohibition to release logs explicitly weakens the privacy must in debug. |
| O03 | Accept with changes | Accept with changes | Low → Low | Text override | Separate adoption from opt-in without adding the incorrect historical claim “the first line that ships it.” |

## Literal checks and repository changes

- All **13 quotations labelled `Current:`** in the first review match the referenced local text verbatim. The additional fragments used for R02, R03 and R07 also match. No incorrect local quotation was found.
- The reviewed files contain 130 lines in `android/SKILL.md`, 203 lines across the nine reference files and 201 lines in `specs/android-guidelines.md`. The cited local/spec locations match.
- `HEAD` remained `0bb5642977aadf8584abc7de23c0a80b856cf614`; there were no later commits or tracked working-tree differences from that checkout.
- Both review targets were untracked. Git cannot reconstruct earlier content changes to those untracked documents. This newly created second-review document is a subsequent documentation addition, not a rule/spec change.

## Per-ID evidence and exact replacements

These are proposed substitutions, not applied patches. Where a fragment is quoted, leave the rest of the line unchanged. Spec references are to `specs/android-guidelines.md` at the reviewed checkout.

### R01 — Correct restriction, overstated consequence and severity

The [accessibility API reference](https://developer.android.com/reference/kotlin/androidx/compose/ui/test/junit4/accessibility/package-summary) documents API 34+, lack of Robolectric support and support limited to `AndroidComposeTestRule`.

That does not establish the first review's claim at lines 29 and 69 that the default causes the check never to run and the verification to pass vacuously. The inspected [`main` implementation](https://github.com/androidx/androidx/blob/androidx-main/compose/ui/ui-test-junit4-accessibility/src/androidMain/kotlin/androidx/compose/ui/test/junit4/accessibility/ComposeTestRuleExt.android.kt) throws `NotImplementedError` for incompatible rule types, but that does not establish the outcome of every Robolectric setup either.

The existing “unless it needs a device” exception also matters. Clarify the host; do not describe a reproduced silent failure. Medium is more defensible than the review's high rating on the evidence available.

**`android/references/testing.md:9`, final sentence**

Current:

```text
Run it in `test` on Robolectric unless it needs a device.
```

Replace with:

```text
Run this accessibility test as an instrumented test on API 34+; `enableAccessibilityChecks()` does not support Robolectric.
```

This keeps the existing per-main-screen accessibility test. It neither mandates a device for every ordinary Compose test nor introduces a second suite.

**`android/SKILL.md:80`**

Current:

```text
Verify: UI tests with `enableAccessibilityChecks()` pass.
```

Replace with:

```text
Verify: instrumented UI tests on API 34+ with `enableAccessibilityChecks()` pass.
```

**Spec:** :157, replace the host fragment with `in instrumented tests on API 34+, not Robolectric`; :124 unchanged. The first review's “mirror the first sentence” instruction is not precise enough: remove the Robolectric host from the accessibility-test instruction explicitly.

### R02 — Keep medium, avoid unconditional separation

The [broadcasts guide](https://developer.android.com/develop/background-work/background-tasks/broadcasts) confirms the privileged-sender exception and warns about exported receivers. Splitting receivers is relevant when both exposure modes are needed.

The first replacement requires “a separate receiver” even when only one external receiver exists, and drops the sender-protection qualification present in the audit.

**`android/SKILL.md:100`; identical fragment in spec:140**

Current:

```text
Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.
```

Replace with:

```text
Dynamic receivers use `RECEIVER_NOT_EXPORTED` unless their expected senders require `RECEIVER_EXPORTED`; use separate receivers when both flags are needed, with appropriate sender restrictions for exported receivers.
```

The component and PendingIntent clauses are unchanged.

### R03 — Portable exception, not automatic authorization by category

The [package-visibility policy](https://support.google.com/googleplay/android-developer/answer/10158779) and [all-files policy](https://support.google.com/googleplay/android-developer/answer/10467955) permit specific cases subject to conditions. These are distribution conditions, not an obligation for every house convention to allow every eligible case.

Accept a low-severity portability correction. The first review's replacement loses the audit's narrower-alternatives condition and effective compliance with applicable policy/declaration/review requirements.

**`android/SKILL.md:98`; identical fragment in spec:139**

Current:

```text
No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```

Replace with:

```text
`QUERY_ALL_PACKAGES` (prefer `<queries>`) and `MANAGE_EXTERNAL_STORAGE` only when narrower alternatives cannot satisfy core functionality, after asking and meeting applicable policy, declaration and review requirements; `USE_EXACT_ALARM` only for alarm or calendar apps; every foreground service declares a `foregroundServiceType`.
```

**Incorrect criticism in first review:63:** the audit does not contradict itself about timers. It warns against broadening this permission using generic exact-alarm guidance, while expressly recognizing alarm/timer/calendar cases. The [current policy](https://support.google.com/googleplay/android-developer/answer/16909972?hl=en), reached from the [cited older URL](https://support.google.com/googleplay/android-developer/answer/13161072?hl=en), includes “an alarm or timer app.” Adding `timer` is optional alignment, not repair of an audit contradiction.

The Photo Picker decision remains closed.

### R04 — The rejection dismisses too much

The [Android JUnit framework README](https://github.com/mannodermaus/android-junit-framework/blob/main/README.md) conditions Vintage on also having JUnit 4 tests in the relevant JUnit Platform configuration. It separates JVM configuration from instrumentation.

- The audit overstates its summary by describing a wholly universal claim.
- It nevertheless identifies a real problem with “forces”: JUnit 4 instrumentation does not by itself require running those tests through Vintage alongside JVM Jupiter tests.
- [Vintage's deprecation in JUnit 6](https://docs.junit.org/6.1.3/overview.html) is migration guidance, not proof of that requirement.
- The R01 correction also removes any inference that the accessibility test necessarily requires local Robolectric tests.

**`android/references/testing.md:5`; identical rationale in spec:153**

Current:

```text
Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 forces the vintage engine and two test styles.
```

Replace with:

```text
Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules use JUnit 4.
```

Keep the JUnit 4 default and the existing-project exception unchanged.

### R05 — Recipes confirm the mismatch, not automatic permission to change the default

Verified evidence:

- [Navigation 3 1.2.0](https://developer.android.com/jetpack/androidx/releases/navigation3#1.2.0), stable on 2026-09-23, includes the results API.
- The [return-results guide](https://developer.android.com/guide/navigation/navigation-3/return-results) uses the bus for both event and state observation. State observation does not itself provide persistence; the bus does not automatically preserve results across process death or activity recreation.
- At commit `05e57eda7297b8f5db7f5176d0fa1a4ee1442881`, the [state](https://github.com/android/nav3-recipes/blob/05e57eda7297b8f5db7f5176d0fa1a4ee1442881/app/src/main/java/com/example/nav3recipes/results/state/ResultStateActivity.kt), [event](https://github.com/android/nav3-recipes/blob/05e57eda7297b8f5db7f5176d0fa1a4ee1442881/app/src/main/java/com/example/nav3recipes/results/event/ResultEventActivity.kt) and [serializable](https://github.com/android/nav3-recipes/blob/05e57eda7297b8f5db7f5176d0fa1a4ee1442881/app/src/main/java/com/example/nav3recipes/results/serializable/ResultSerializableActivity.kt) recipes use the bus. The [serializable helper](https://github.com/android/nav3-recipes/blob/05e57eda7297b8f5db7f5176d0fa1a4ee1442881/app/src/main/java/com/example/nav3recipes/results/serializable/Extensions.kt) does call `rememberSerializable`. The first review's summary is correct even though the call is indirect.
- That recipe checkout [pins Navigation 3 1.3.0-alpha02](https://github.com/android/nav3-recipes/blob/05e57eda7297b8f5db7f5176d0fa1a4ee1442881/gradle/libs.versions.toml). Do not present the entire current recipe tree as verified against stable 1.2.0.

**Omitted interaction:** the [published runtime 1.2.0 sources](https://dl.google.com/dl/android/maven2/androidx/navigation3/navigation3-runtime/1.2.0/navigation3-runtime-1.2.0-sources.jar) contain an error for obtaining the bus without its provider: `No ResultEventBus has been provided`. The results decorator installs the provider. The exact list mandated by `android/SKILL.md:37` contains only saved-state and ViewModel-store decorators.

The first review's replacement therefore reverses, rather than preserves, the existing no-bus default; fails to state its stack/version boundary explicitly; and omits the required provider/decorator interaction. This is source evidence, not a runtime reproduction.

**Smallest correction preserving the existing convention — `android/references/navigation.md:7`, first sentence**

Current:

```text
Return results to a previous screen as state (Google's results-as-state recipe), unless the project already has a result mechanism.
```

Replace with:

```text
Return results to a previous screen as state, unless the project already has a result mechanism.
```

Keep the following sentence avoiding the bus. **Spec:74:** likewise remove `(Google's results-as-state recipe)`.

This removes the false attribution without introducing a mechanism or claiming all state is persistent. If the owner instead wants the bus as a new default, that needs an explicit decision and a coordinated correction to `SKILL.md:37`; the first review's proposed replacement is not sufficient.

### R06 — Do not promote a verification limit into an asserted alternative

The [16 KB guide](https://developer.android.com/guide/practices/page-sizes) distinguishes ELF alignment, packaging alignment and execution testing.

The first review acknowledges at :85 and :188 that APK Analyzer's specific ELF capability was not established, but its :120 replacement offers it as an alternative for precisely that check. That is an inconsistent handoff, not evidence that APK Analyzer cannot perform it.

**`android/SKILL.md:93`**

Current:

```text
Verify: APK Analyzer, or `zipalign -c -P 16 -v 4 app-release.apk`.
```

Replace with:

```text
Verify: ELF segment alignment with `check_elf_alignment.sh`, APK alignment with `zipalign -c -P 16 -v 4 app-release.apk`, and execution on a 16 KB device or emulator.
```

**Spec:135:** unchanged; it contains no verification instruction.

### R07 — Sound conditional derivation, wrong proposed boundary

The [AGP defaults](https://android.googlesource.com/platform/tools/base/+/mirror-goog-studio-main/build-system/gradle-core/src/main/resources/com/android/build/gradle/proguard-common.txt), [AAPT2 generator](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/tools/aapt2/java/ProguardRules.cpp), [R8 keep-rule semantics](https://developer.android.com/topic/performance/app-optimization/keep-rules-overview) and [reflective-access examples](https://developer.android.com/topic/performance/app-optimization/keep-rule-examples) support the reasoning:

- `-keepclassmembers` alone does not rescue a removable class.
- Generated rules retain specific entry points, not every possible member.
- Layout-generated retention can depend on the resource remaining live.
- A remote class name is not a reference AAPT2 can process from manifest/layout input.

The counterexample is sound **if** no other roots/keep rules retain the class and the dynamic access is not statically resolvable. First review:51 should say R8 **can remove or rename** the class under those conditions, rather than describe elimination and `ClassNotFoundException` as established outcomes.

The inspected AGP/AAPT2 files are on `main` branches, not pinned releases. Nothing was compiled or reproduced.

The proposed qualifier still fails: a View already referenced by XML can have another constructor or a reflective/JNI member not covered by those generated rules. The correct boundary is coverage of a specific entry point/member, not presence of a class name in XML.

**`android/SKILL.md:105`; identical fragment in spec:145**

Current:

```text
No rules for Activity and View classes.
```

Replace with:

```text
No redundant rules for Activity and View entry points already covered by generated or default rules.
```

The existing narrow-rule requirements remain. The earlier consumer-rule decision is not reopened.

### R08 — Scope the defaults and verification too

The [type-safety guide](https://developer.android.com/guide/navigation/design/type-safety) confirms Navigation 2.8.0, Navigation Compose/Kotlin DSL, and the separate XML/Safe Args path. The [predictive-back guide](https://developer.android.com/guide/navigation/custom-back/predictive-back-gesture) documents the callback/dispatcher path for Views. [NavigationEvent 1.0.0](https://developer.android.com/jetpack/androidx/releases/navigationevent), released 2025-11-19, documents `NavigationBackHandler` in its Compose module.

The first review's main replacements are appropriate, but its instruction to leave `navigation.md:30` and spec:132 unchanged leaves an unscoped Compose default. The route grep also needs the same boundary as its must.

**`android/SKILL.md:34`**

Current:

```text
Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x). No string routes in new code.
```

Replace with:

```text
Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on Navigation Compose 2.8+). No string routes in new code on those versions; older 2.x and Fragment/XML graphs keep their existing route or Safe Args style.
```

**Spec:70:** mirror the boundary.

**`android/SKILL.md:35`**

Current:

```text
Verify: `rg 'route = "'` returns nothing in new code.
```

Replace with:

```text
Verify: on those stacks and versions, `rg 'route = "'` returns nothing in new code.
```

**`android/SKILL.md:87`, first sentence**

Current:

```text
Handle back only through navigation (Navigation 3's built-in back) or `NavigationBackHandler` (`androidx.navigationevent`).
```

Replace with:

```text
Handle back only through navigation (Navigation 3's built-in back), `NavigationBackHandler` in Compose, or `OnBackPressedCallback` in View/Fragment code.
```

**Spec:131:** mirror; preserve the remaining prohibitions.

**`android/references/navigation.md:30`**

Current:

```text
Use `NavigationBackHandler` for new back handling outside navigation, unless the project already uses `BackHandler` or `PredictiveBackHandler`.
```

Replace with:

```text
In Compose, use `NavigationBackHandler` for new back handling outside navigation, unless the project already uses `BackHandler` or `PredictiveBackHandler`.
```

**Spec:132:** add the same `In Compose` boundary. No migration or version change is requested.

### R09 — Agree with the proposed replacement

**Local/spec:** `android/SKILL.md:65` ↔ spec:102.

The [custom-modifiers guide](https://developer.android.com/develop/ui/compose/custom-modifiers) distinguishes combining factories from implementing more custom behavior with Node. The first review's replacement is sufficient and appropriately scoped. Keeping “never composed” is a house convention; the upstream source says it is no longer recommended, not that the framework prohibits it.

### R10 — Valid correction, redundant additional sentence

The conflict with `navigation.md:21–26` and `SKILL.md:58` is real. The [Hilt release notes](https://developer.android.com/jetpack/androidx/releases/hilt) support assisted creation in 1.2.0 and the package move in the 1.3 line.

**`android/references/di-and-concurrency.md:11`**

Current:

```text
DI audit: flag composables that resolve dependencies directly.
```

Replace with:

```text
DI audit: flag composables below the screen entry that resolve dependencies directly.
```

**Spec:88:** add the same qualifier. The first review's additional `hiltViewModel()` clause does not add a necessary restriction.

### R11 — Agree with the proposed replacement

**Local/spec:** `android/SKILL.md:91` ↔ spec:134.

The [state-hoisting guide](https://developer.android.com/develop/ui/compose/state-hoisting) permits local/transient UI state. Adding `that must survive configuration changes` is a sufficient minimal correction. It does not require a new state architecture or weaken the existing screen-level state convention.

### R12 — Reject as a mandatory correction, not because controlled tests are impossible

**Local/spec:** `android/references/testing.md:7` ↔ spec:155.

[Google recommends `.value` assertions](https://developer.android.com/kotlin/flow/test) while also recognizing controlled collection scenarios. The local rule can remain a house default. Checking loading and then content through current-value assertions at coordinated checkpoints does not require asserting every emission.

However, the [`UnconfinedTestDispatcher` order caveat](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-unconfined-test-dispatcher.html) does not automatically invalidate the same documentation's controlled no-skipped-values example. The audit is right about that possibility; it has not demonstrated that the house convention must change.

No replacement is required. Reclassify the audit's concern as optional convention wording rather than a mandatory technical correction. Do not describe intermediate-emission tests as universally invalid or imply production guarantees every update.

### O01 — Verified prerequisites without a blanket AppCompat requirement

The [app-languages guide](https://developer.android.com/guide/topics/resources/app-languages) confirms AGP 8.1+, compileSdk 33+, `resources.properties` and `unqualifiedResLocale`. Its `AppCompatActivity` condition concerns the AppCompat language-selection/back-compat API path, not automatic LocaleConfig generation itself.

**`android/references/compose-ui.md:21`**

Current:

```text
Support per-app language through `generateLocaleConfig` in apps with more than one locale, unless the project already configures locales another way.
```

Replace with:

```text
Support per-app language through `generateLocaleConfig` (AGP 8.1+, compileSdk 33+, and `unqualifiedResLocale` in `res/resources.properties`) in apps with more than one locale, unless the project already configures locales another way.
```

**Spec:125:** mirror the prerequisites. This does not authorize an AGP/SDK upgrade without asking.

### O02 — “Release logs” weakens the must

`SKILL.md:118`, spec:167 and `tooling-and-logging.md:3` prohibit sensitive data without a build-type restriction. Planting `DebugTree` in debug is not an exception.

The [OkHttp README](https://github.com/lysine-dev/okhttp/blob/main/okhttp-logging-interceptor/README.md) warns that HEADERS/BODY can expose credentials and bodies. `redactHeader()` does not sanitize bodies or establish that every other logged field is safe.

- The first review's replacement permits in debug exactly the information the must prohibits.
- Preserving the existing privacy boundary is not new scope. Mandating a particular `redactHeader()` call would be an implementation choice, not the only way to repair the rule.
- Do not invent a synthetic-data exception without an explicit decision. “Debug-only” does not establish one.

**Smallest conservative correction — `android/SKILL.md:118`; identical sentence in spec:167**

Current:

```text
No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.
```

Replace with:

```text
No personal data, tokens or full request/response bodies in logs, including debug logs.
```

This removes an apparent incompatible allowance; it does not replace Timber or introduce logging infrastructure. If the owner wants a BODY exception, its scope must be decided explicitly rather than inferred from build type.

### O03 — Keep the house version gate, correct the explanation

The [Material 3 release notes](https://developer.android.com/jetpack/androidx/releases/compose-material3) confirm:

- 1.4.0-beta01 removed the expressive experimental APIs from that line and pointed users to 1.5.0-alpha.
- 1.5.0-alpha18 promoted APIs and added `Material3ExpressiveApi` without requiring opt-in.
- At retrieval time the release table showed stable 1.4.0 and 1.5.0-beta02, not stable 1.5.0.

`1.5+` can remain the chosen release-line convention. “The first line that ships it” is not a correct historical description: expressive APIs existed in 1.4 prereleases before removal.

**`android/references/compose-ui.md:17`, last sentence**

Current:

```text
Use Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).
```

Replace with:

```text
Use Material 3 Expressive only when the project is on material3 1.5+ and has chosen it (see the experimental-API must).
```

**Spec:115:** likewise change `opts in` to `has chosen it`. The whole must at `SKILL.md:66`, including its alpha-API restriction, remains applicable.

## Citation corrections and newly resolved verification limits

### NavDisplay: broken citation, released behavior now checked

For the exact full SHA `4d74181ec5f92b49275c03d9715a965080bb319b` in audit:283:

| Endpoint | Observed result |
|---|---|
| [GitHub blob](https://github.com/androidx/androidx/blob/4d74181ec5f92b49275c03d9715a965080bb319b/navigation3/navigation3-ui/src/commonMain/kotlin/androidx/navigation3/ui/NavDisplay.kt) | HTTP 404 |
| [Raw file](https://raw.githubusercontent.com/androidx/androidx/4d74181ec5f92b49275c03d9715a965080bb319b/navigation3/navigation3-ui/src/commonMain/kotlin/androidx/navigation3/ui/NavDisplay.kt) | HTTP 404 |
| [Full-SHA commit API](https://api.github.com/repos/androidx/androidx/commits/4d74181ec5f92b49275c03d9715a965080bb319b) | HTTP 422; no commit found for that SHA |

The first review is correct about this broken link. Separately, the [published Navigation 3 UI 1.2.0 source JAR](https://dl.google.com/dl/android/maven2/androidx/navigation3/navigation3-ui/1.2.0/navigation3-ui-1.2.0-sources.jar), `commonMain/androidx/navigation3/ui/NavDisplay.kt`, confirms the saved-state default and passage of the supplied list to `rememberDecoratedNavEntries`, without appending that default. This evidence is no longer limited to `androidx-main`.

### OkHttp: the “wrong repository” accusation is not supported

Exact retrievals produced:

| Check | Observed result |
|---|---|
| [API `square/okhttp`](https://api.github.com/repos/square/okhttp) | Resolved to repository ID `5152285`, `full_name: lysine-dev/okhttp` |
| [API `lysine-dev/okhttp`](https://api.github.com/repos/lysine-dev/okhttp) | Same repository ID `5152285` |
| [Exact audit URL, `main`](https://github.com/lysine-dev/okhttp/blob/main/okhttp-logging-interceptor/README.md) | HTTP 200; contains the cited README |
| [Review URL, `square/.../master/...`](https://github.com/square/okhttp/blob/master/okhttp-logging-interceptor/README.md) | HTTP 200; redirected to the audit URL |
| [Audit raw file](https://raw.githubusercontent.com/lysine-dev/okhttp/main/okhttp-logging-interceptor/README.md) and [review raw file](https://raw.githubusercontent.com/square/okhttp/master/okhttp-logging-interceptor/README.md) | Complete response bodies were exactly equal in memory |

`fork: false` does not prove different repository identities or establish ownership history. Correct first review:11, :58 and :181: there is **one demonstrated broken citation, not two**. The audit's O02 URL resolves to the relevant content. No historical ownership conclusion is made here.

### Hilt: deprecation verified in published sources

The complete source JARs for [1.3.0](https://dl.google.com/dl/android/maven2/androidx/hilt/hilt-navigation-compose/1.3.0/hilt-navigation-compose-1.3.0-sources.jar) and [1.3.0-alpha02](https://dl.google.com/dl/android/maven2/androidx/hilt/hilt-navigation-compose/1.3.0-alpha02/hilt-navigation-compose-1.3.0-alpha02-sources.jar) contain both old-package overloads with `@Deprecated` and `ReplaceWith` targeting `androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel`.

Update first review:66 and :188: the annotation is established in those published versions, not only on main. An earlier partial-source retrieval missed the annotations; the complete-file/JAR follow-up corrected that result. Release-note silence was never evidence of absence.

### Other checks that do not create new findings

- [Kotlin 2.4](https://kotlinlang.org/docs/whatsnew24.html), released 2026-07-14, lists explicit backing fields as stable. The [properties guide](https://kotlinlang.org/docs/properties.html) demonstrates the smart cast to the backing-field type inside the class. The local example was not compiled.
- [ktlint release metadata](https://api.github.com/repos/ktlint/ktlint/releases?per_page=100) confirms 1.8.0 stable and 2.0.0-ALPHA-4 prerelease; [detekt metadata](https://api.github.com/repos/detekt/detekt/releases?per_page=100) confirms v1.23.8 stable and v2.0.0-alpha.6 prerelease.
- [`BooleanOption.kt`](https://android.googlesource.com/platform/tools/base/+/mirror-goog-studio-main/build-system/gradle-core/src/main/java/com/android/build/gradle/options/BooleanOption.kt?format=TEXT) confirms `android.disallowKotlinSourceSets`, default `true`, Experimental stage and the declared Enforced transition in AGP 10. This is branch-source evidence, not certification of all releases or the full behavior of setting it to `false`.

## Verification limits after this second pass

| Point | Status |
|---|---|
| Navigation key → `SavedStateHandle` contract | Still unverified precisely. An assisted-injection recipe alone does not establish impossibility of other integration. Not classified as a defect. |
| Earliest Roborazzi/Paparazzi AGP 9 versions | [Roborazzi 1.56.0](https://github.com/takahirom/roborazzi/releases/tag/1.56.0) and [Paparazzi 2.0.0-alpha05.1](https://github.com/cashapp/paparazzi/releases/tag/2.0.0-alpha05.1) establish support in those releases, not exhaustive earliest-version claims. |
| APK Analyzer specifically checking ELF segments | Still not established specifically. The replacement avoids relying on it; no incapability is asserted. |
| Published Hilt deprecation | Resolved: present in 1.3.0-alpha02 and 1.3.0. |
| Released NavDisplay decorator-list behavior | Resolved for published 1.2.0, in addition to main-source evidence. |
| Compilation/runtime | None performed. R07 remains derived; exact accessibility behavior in a particular Robolectric configuration is not reproduced. |

## Required document corrections and implementation readiness

**Conclusion: requires corrections.**

- **`portable-rules-remainder-audit-review.md`:** correct R01's execution claim/severity, the R04 rejection, the replacements identified above, the timer-note criticism, the OkHttp accusation and the now-resolved verification limits. Withdraw the conclusion that its thirteen current replacements are ready to implement together.
- **`portable-rules-remainder-audit.md`:** repair the NavDisplay citation; align R01/R02 severity and R04 scope; present R12 as optional convention wording, not a mandatory technical correction; incorporate R07's precise conditions and the published-source evidence.
- **Owner decisions:** whether to adopt `ResultEventBus` as a new default with the necessary setup, and whether to define a BODY-logging exception compatible with privacy. Neither decision should be made silently. The conservative replacements above retain the existing constraints.
- **Remaining uncertainties:** precise Navigation key/`SavedStateHandle` contract, exhaustive earliest screenshot-tool versions and APK Analyzer's specific ELF capability. These are not demonstrated defects and do not justify reopening the original seven proposals.

## Copy-ready prompt for an independent review

```text
Read @research/android/portable-rules-remainder-second-review.md and independently review it.

It reviews @research/android/portable-rules-remainder-audit-review.md, which in turn reviews @research/android/portable-rules-remainder-audit.md. All three documents are review targets, not authorities. Do not assume any verdict, severity, quotation, source-retrieval claim, HTTP status, version claim or verification result is correct.

Do not modify or create any file. Do not implement rules or update the spec. Keep the original seven-proposal review closed unless you identify a concrete cross-file interaction that requires examining it.

For every ID R01–R12 and O01–O03:
1. State accept / reject / accept with changes for the second review's verdict and proposed text, with severity and evidence.
2. Check the current local android/SKILL.md, all relevant android/references/*.md text and the matching specs/android-guidelines.md line. Verify quotations verbatim and assess whether each replacement is the smallest change that actually resolves the finding.
3. Re-verify technical claims against cited primary sources. Distinguish published releases from unreleased/main code, recommendations from requirements, and demonstrated errors from unverified claims.
4. Check replacements for new contradictions with musts/defaults, lost house conventions, unnecessary scope expansion or missing cross-file changes.

Challenge these points especially:
- R01: Is medium justified instead of high? Does the existing device exception matter? Is changing the host of the per-screen accessibility test preferable to separating ordinary Robolectric tests from accessibility tests? Do not equate unsupported with a silent no-op or a vacuous pass without evidence.
- R04: Does removing the Vintage rationale preserve the JUnit 4 convention while correcting a genuine overstatement, or did the second review underweight the local Robolectric premise? Distinguish JVM JUnit Platform tests from separate Android instrumentation.
- R12: Is rejection as a mandatory correction consistent with accepting R04? Does the UnconfinedTestDispatcher order warning actually invalidate the controlled example, or merely limit what it proves?
- R05: The second review recommends removing the Google recipe attribution while retaining the current no-bus default. Is that the smallest consistent repair, or does it leave misleading state/persistence reasoning? Independently inspect the return-results guide, results/state, results/event and results/serializable recipes, and the serializable helper. Verify the recipe dependency pin. Check published Navigation 3 1.2.0 sources for LocalResultEventBus's provider requirements and NavDisplay's decorator defaults. Decide whether the first review's bus default conflicts with the exact two-decorator list in SKILL.md:37 and what would be needed if the owner chooses that default.
- R07: Validate the remote-class-name counterexample as a conditional source derivation, not a compiled reproduction. Does “entry points already covered by generated or default rules” preserve permission for uncovered reflective/JNI members, and is it a better boundary than merely being referenced from manifest/XML?
- O02: Does removing the BODY clause preserve the stronger privacy must, or discard an intentional convention that needs an explicit owner decision? Do not silently permit real credentials or personal data in debug. Header redaction alone does not establish body safety.
- R03/R08/O01: Are the proposed qualifications necessary and sufficient without introducing new rules? Check the timer-policy criticism, remaining navigation defaults/verification grep, and exact LocaleConfig prerequisites.
- O03: Verify 1.4 prerelease removal, 1.5 annotation changes and stable/prerelease status. Is replacing “opts in” with “has chosen it” enough while preserving the full experimental/alpha API must?

Independently check the disputed citations and newly resolved limits:
- Fetch the exact full-SHA NavDisplay GitHub URL, raw URL and commit API endpoint. Do not substitute a short-SHA lookup for the exact citation.
- Check both square/okhttp and lysine-dev/okhttp API repository identities, the exact README URLs/redirects and full README contents. The second review reports the same repository ID 5152285 and identical README bodies. Verify this rather than assuming historical ownership or treating fork:false as lineage evidence.
- Inspect the complete published Hilt 1.3.0 and 1.3.0-alpha02 source JAR files for @Deprecated/ReplaceWith. Do not infer absence from release-note silence or a truncated source excerpt.
- Check whether the remaining Navigation key/SavedStateHandle, earliest screenshot-tool versions and APK Analyzer/ELF limits are still unresolved. An unverified claim is not automatically a defect.

Retrieve complete source sections. Do not mistake the first 2,000 lines of a fetched page for the whole page; use retained full content, offsets, targeted retrieval or in-memory extraction. A helper function can contain the behavior even when the calling activity does not. A source-tree filename search is not a content search.

Do not claim compilation or runtime reproduction unless actually performed. Be precise about what was fetched, what was inferred and what remains unverified. Check git status and changes since checkout 0bb5642977aadf8584abc7de23c0a80b856cf614; distinguish tracked changes from untracked review documents.

Return:
- A table covering all 15 IDs: second-review verdict, your verdict, severity, agreement/override and one-line reason.
- Every incorrect quotation, citation, assumption or conclusion you find in any of the three documents, with exact document/local references and primary evidence. Separate factual errors from optional wording preferences and unresolved checks.
- For each override, current local text, your smallest exact replacement and the matching spec line. Preserve untouched parts of the rules.
- A final conclusion: “ready to implement” or “requires corrections”, naming exactly which document needs which change, which owner decisions remain and which verification limits are still open.

Do not edit files, add dependencies, change architecture, refactor or turn this review into implementation.
```
