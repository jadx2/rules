# Handoff: implement the remaining Android rule corrections

**Date:** 2026-10-21 · **Baseline:** `0bb5642977aadf8584abc7de23c0a80b856cf614` on `main`
**Decision record:** `research/android/portable-rules-remainder-audit.md` (audit), `…-audit-review.md` (first review), `…-second-review.md` (second review), and a third independent review whose outcome is recorded here. Where those three documents disagree with this file, this file wins: it carries the owner's two decisions and the replacement texts that survived all three passes. The evidence is settled; do not re-verify sources.

## Task

Apply the edits below to five rule files, mirror them in the spec, run the checks, commit on a branch and open a PR. Nothing else changes: no other rules, no new files, no reformatting of untouched lines, no edits to the three review documents.

Working rules for this repo:
- `fd` is not installed; use `rg` or `find`. Call `/usr/bin/grep` directly.
- Match by text, not by line number. Line numbers below are baseline; spec edit S0 inserts a line and shifts everything below it.
- Never amend or force-push. No `Co-Authored-By` or "Generated with" lines.
- Branch from `main`; this repo merges through PRs (#1 to #4).

## Settled decisions

- **Navigation 3 results (R05):** keep the no-bus default. Remove the false attribution to Google's recipe and replace the rationale with the guide's own persistence rule. Do not adopt `ResultEventBus` and do not touch the decorator list on `SKILL.md:37`.
- **Debug BODY logging (O02):** keep the exception and make it explicit. Do not scope the privacy must to release logs, and do not add a `redactHeader()` requirement.
- **Accessibility test host (R01):** the per-screen Compose UI test stays on Robolectric; only `enableAccessibilityChecks()` moves to instrumented tests on API 34+. Say "unsupported", not "does not run": the published 1.12.1 code logs a warning on Robolectric and still installs the validator.
- **R12 (StateFlow assertions):** no change. The `.value` default is a house convention.
- **Rejected texts, do not apply:** the first review's `ResultEventBus` default (its R05), "release logs" (its O02), "the first line that ships it" (its O03), the extra `hiltViewModel()` clause (its R10), the "manifest or layout XML already reference" qualifier (its R07), and the second review's "Run this accessibility test as an instrumented test" (its R01) and "use JUnit 4" (its R04).

## Edits to `android/SKILL.md`

Each edit replaces the whole line unless it says "fragment". Fragment edits leave the rest of the line unchanged.

**E1, line 34** (R08)

Current:
```markdown
- Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x). No string routes in new code.
```
Replacement:
```markdown
- Routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on Navigation Compose 2.8+). No string routes in new code on those versions; older 2.x and Fragment/XML graphs keep their existing route or Safe Args style.
```

**E2, line 35** (R08)

Current:
```markdown
  - Verify: `rg 'route = "'` returns nothing in new code.
```
Replacement:
```markdown
  - Verify: on those stacks and versions, `rg 'route = "'` returns nothing in new code.
```

**E3, line 65** (R09)

Current:
```markdown
- Custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.
```
Replacement:
```markdown
- Custom modifier behavior uses `Modifier.Node`/`ModifierNodeElement`; a factory that only chains existing modifiers needs no Node. Never `Modifier.composed`.
```

**E4, line 80** (R01)

Current:
```markdown
  - Verify: UI tests with `enableAccessibilityChecks()` pass.
```
Replacement:
```markdown
  - Verify: instrumented UI tests on API 34+ with `enableAccessibilityChecks()` pass.
```

**E5, line 87** (R08)

Current:
```markdown
- Handle back only through navigation (Navigation 3's built-in back) or `NavigationBackHandler` (`androidx.navigationevent`). Use `BackHandler`/`PredictiveBackHandler` only where the project already does. Never override `onBackPressed` or set `enableOnBackInvokedCallback="false"`.
```
Replacement:
```markdown
- Handle back only through navigation (Navigation 3's built-in back), `NavigationBackHandler` (`androidx.navigationevent`) in Compose, or `OnBackPressedCallback` in View/Fragment code. Use `BackHandler`/`PredictiveBackHandler` only where the project already does. Never override `onBackPressed` or set `enableOnBackInvokedCallback="false"`.
```

**E6, line 91** (R11)

Current:
```markdown
- UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
```
Replacement:
```markdown
- UI state that must survive configuration changes lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
```

**E7, line 93** (R06)

Current:
```markdown
  - Verify: APK Analyzer, or `zipalign -c -P 16 -v 4 app-release.apk`.
```
Replacement:
```markdown
  - Verify: ELF segment alignment with `check_elf_alignment.sh`, APK alignment with `zipalign -c -P 16 -v 4 app-release.apk`, and a run on a 16 KB device or emulator.
```

**E8, line 98, fragment** (R03). The Photo Picker sentences before it stay.

Current fragment:
```markdown
No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```
Replacement fragment:
```markdown
`QUERY_ALL_PACKAGES` (prefer `<queries>`) and `MANAGE_EXTERNAL_STORAGE` only when narrower alternatives cannot satisfy core functionality, after asking and meeting applicable policy, declaration and review requirements; `USE_EXACT_ALARM` only for alarm, timer or calendar apps; every foreground service declares a `foregroundServiceType`.
```

**E9, line 100, fragment** (R02). The component and `PendingIntent` sentences stay.

Current fragment:
```markdown
Dynamic receivers register with `RECEIVER_NOT_EXPORTED`.
```
Replacement fragment:
```markdown
Dynamic receivers use `RECEIVER_NOT_EXPORTED` unless their expected senders (other apps, or privileged system senders such as Bluetooth and telephony) require `RECEIVER_EXPORTED`; use separate receivers when both flags are needed, and protect exported receivers with a permission or a sender check.
```

**E10, line 105, fragment** (R07). Everything else on the line stays.

Current fragment:
```markdown
No rules for Activity and View classes.
```
Replacement fragment:
```markdown
No redundant rules for Activity and View entry points already covered by generated or default rules.
```

**E11, line 118** (O02)

Current:
```markdown
- No `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.
```
Replacement:
```markdown
- No `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; the one exception is `HttpLoggingInterceptor` at `BODY`, in debug builds only.
```

## Edits to reference files

**F1, `android/references/testing.md:5`** (R04)

Current:
```markdown
- Use JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 forces the vintage engine and two test styles. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
```
Replacement:
```markdown
- Use JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 means two test styles, plus the vintage engine wherever those JUnit 4 tests share a JVM test task. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
```

**F2, `android/references/testing.md:9`** (R01). Line 10, the indented sub-bullet, stays.

Current:
```markdown
- Write one Compose UI test per main screen against `ScreenContent` with `enableAccessibilityChecks()`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device.
```
Replacement:
```markdown
- Write one Compose UI test per main screen against `ScreenContent`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device. Run `enableAccessibilityChecks()` only in instrumented tests on API 34+; it is unsupported on Robolectric.
```

**F3, `android/references/navigation.md:7`** (R05)

Current:
```markdown
- Return results to a previous screen as state (Google's results-as-state recipe), unless the project already has a result mechanism. Avoid the in-memory `ResultEventBus`: it loses results on process death.
```
Replacement:
```markdown
- Return results to a previous screen as state, unless the project already has a result mechanism. Avoid the in-memory `ResultEventBus`. A result that must survive process death is persisted (`rememberSaveable`, a `SavedStateHandle` or the `NavKey`), whatever carries it.
```

**F4, `android/references/navigation.md:30`** (R08)

Current:
```markdown
- Use `NavigationBackHandler` for new back handling outside navigation, unless the project already uses `BackHandler` or `PredictiveBackHandler`.
```
Replacement:
```markdown
- In Compose, use `NavigationBackHandler` for new back handling outside navigation, unless the project already uses `BackHandler` or `PredictiveBackHandler`.
```

**F5, `android/references/di-and-concurrency.md:11`** (R10)

Current:
```markdown
- DI audit: flag composables that resolve dependencies directly.
```
Replacement:
```markdown
- DI audit: flag composables below the screen entry that resolve dependencies directly.
```

**F6, `android/references/compose-ui.md:17`** (O03)

Current:
```markdown
- Dynamic color is a per-project decision: follow the project's choice. Use Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).
```
Replacement:
```markdown
- Dynamic color is a per-project decision: follow the project's choice. Use Material 3 Expressive only when the project is on material3 1.5+ and has chosen it (see the experimental-API must).
```

**F7, `android/references/compose-ui.md:21`** (O01)

Current:
```markdown
- Support per-app language through `generateLocaleConfig` in apps with more than one locale, unless the project already configures locales another way.
```
Replacement:
```markdown
- Support per-app language through `generateLocaleConfig` (AGP 8.1+, compileSdk 33+, and `unqualifiedResLocale` in `res/resources.properties`) in apps with more than one locale, unless the project already configures locales another way.
```

## Edits to `specs/android-guidelines.md`

**S0, insert after line 6** (the line that starts `**Revised 2026-10-21** after an independent review of seven rules`). New line:
```markdown
**Revised 2026-10-21 (remainder)** after an independent audit of the remaining rules and two reviews of it (`research/android/portable-rules-remainder-audit.md`, `…-audit-review.md`, `…-second-review.md`; settled in `…-remainder-handoff.md`): accessibility-test host, receiver export flags, restricted permissions, JUnit rationale, Navigation 3 results, 16 KB verification, Activity/View keep rules, navigation version boundaries, custom modifiers, DI audit scope, UI-state scope, LocaleConfig prerequisites, debug BODY logging and Material 3 Expressive wording corrected.
```

Baseline line numbers below are before S0.

**S1, line 70** (E1)

Current:
```markdown
- **Must:** routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on 2.x); no string routes in new code.
```
Replacement:
```markdown
- **Must:** routes are `@Serializable` types (`NavKey` on Navigation 3, `data object`/`data class` on Navigation Compose 2.8+); no string routes in new code on those versions; older 2.x and Fragment/XML graphs keep their existing route or Safe Args style.
```

**S2, line 74** (F3)

Current:
```markdown
- **Default:** results returned to a previous screen travel as state (Google's results-as-state recipe), not through the in-memory `ResultEventBus`, which loses results on process death.
```
Replacement:
```markdown
- **Default:** results returned to a previous screen travel as state, not through the in-memory `ResultEventBus`; a result that must survive process death is persisted (`rememberSaveable`, a `SavedStateHandle` or the `NavKey`), whatever carries it.
```

**S3, line 88** (F5)

Current:
```markdown
- **Audit:** flag composables that resolve dependencies directly.
```
Replacement:
```markdown
- **Audit:** flag composables below the screen entry that resolve dependencies directly.
```

**S4, line 102** (E3)

Current:
```markdown
- **Must:** custom modifiers use `Modifier.Node`/`ModifierNodeElement`, never `Modifier.composed`.
```
Replacement:
```markdown
- **Must:** custom modifier behavior uses `Modifier.Node`/`ModifierNodeElement`; a factory that only chains existing modifiers needs no Node. Never `Modifier.composed`.
```

**S5, line 115** (F6)

Current:
```markdown
- **Default:** dynamic color is a per-project decision. Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).
```
Replacement:
```markdown
- **Default:** dynamic color is a per-project decision. Material 3 Expressive only when the project is on material3 1.5+ and has chosen it (see the experimental-API must).
```

**S6, line 125** (F7)

Current:
```markdown
- **Default:** per-app language support via `generateLocaleConfig` in apps with more than one locale.
```
Replacement:
```markdown
- **Default:** per-app language support via `generateLocaleConfig` (AGP 8.1+, compileSdk 33+, and `unqualifiedResLocale` in `res/resources.properties`) in apps with more than one locale.
```

**S7, line 131** (E5)

Current:
```markdown
- **Must:** back handling only through navigation (Navigation 3's built-in back on Nav3), `NavigationBackHandler` (`androidx.navigationevent`), or `BackHandler`/`PredictiveBackHandler` where the project already uses them. No `onBackPressed` overrides; never set `enableOnBackInvokedCallback="false"`.
```
Replacement:
```markdown
- **Must:** back handling only through navigation (Navigation 3's built-in back on Nav3), `NavigationBackHandler` (`androidx.navigationevent`) in Compose, `OnBackPressedCallback` in View/Fragment code, or `BackHandler`/`PredictiveBackHandler` where the project already uses them. No `onBackPressed` overrides; never set `enableOnBackInvokedCallback="false"`.
```

**S8, line 132** (F4)

Current:
```markdown
- **Default:** `NavigationBackHandler` for new back handling outside navigation.
```
Replacement:
```markdown
- **Default:** in Compose, `NavigationBackHandler` for new back handling outside navigation.
```

**S9, line 134** (E6)

Current:
```markdown
- **Must:** UI state lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
```
Replacement:
```markdown
- **Must:** UI state that must survive configuration changes lives in the ViewModel or `rememberSaveable`; don't rely on `onCreate` re-running after configuration changes.
```

**S10, line 139, fragment** (E8). The Photo Picker sentences before it and the trailing `Verify: review manifest grep hits.` stay. Same current and replacement fragments as E8.

**S11, line 140, fragment** (E9). Same current and replacement fragments as E9.

**S12, line 145, fragment** (E10). Same current and replacement fragments as E10.

**S13, line 153** (F1)

Current:
```markdown
- **Default:** JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 forces the vintage engine and two test styles. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
```
Replacement:
```markdown
- **Default:** JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 means two test styles, plus the vintage engine wherever those JUnit 4 tests share a JVM test task. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
```

**S14, line 157, fragment** (F2). The rest of the line, from `Find nodes by semantics`, stays.

Current fragment:
```markdown
- **Default:** one Compose UI test per main screen against `ScreenContent`, with `enableAccessibilityChecks()`, in `test` on Robolectric unless it needs a device.
```
Replacement fragment:
```markdown
- **Default:** one Compose UI test per main screen against `ScreenContent`, in `test` on Robolectric unless it needs a device; `enableAccessibilityChecks()` only in instrumented tests on API 34+, not on Robolectric.
```

**S15, line 167** (E11)

Current:
```markdown
- **Must:** no `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; `HttpLoggingInterceptor` at `BODY` in debug builds only.
```
Replacement:
```markdown
- **Must:** no `android.util.Log` or `println` in production code. No personal data, tokens or full request/response bodies in logs; the one exception is `HttpLoggingInterceptor` at `BODY`, in debug builds only.
```

Unchanged on purpose: spec :73 (decorator list), :124 (accessibility must), :135 (16 KB must has no verify text), :155 (StateFlow assertions).

## Checks before committing

Run from the repo root and record the outputs for the PR body.

```sh
# 1. SKILL.md stays under the 150-line limit (spec criterion 2)
wc -l android/SKILL.md

# 2. Old wording is gone everywhere
rg -n -F -e 'with `enableAccessibilityChecks()`, unless' -e 'with `enableAccessibilityChecks()`, in' -e 'Verify: UI tests with' \
  -e 'Dynamic receivers register with' -e 'No `QUERY_ALL_PACKAGES`' -e 'forces the vintage engine' -e 'results-as-state recipe' \
  -e 'loses results on process death' -e 'Verify: APK Analyzer, or' -e 'No rules for Activity and View classes' -e '`data class` on 2.x' \
  -e "Verify: \`rg 'route" -e '(`androidx.navigationevent`). Use' -e '(`androidx.navigationevent`), or' \
  -e 'Use `NavigationBackHandler` for new' -e '**Default:** `NavigationBackHandler` for new' \
  -e 'ustom modifiers use `Modifier.Node`' -e 'flag composables that resolve' -e 'UI state lives in the ViewModel' \
  -e '`generateLocaleConfig` in apps' -e 'at `BODY` in debug builds only' -e 'and opts in (see' android specs
# expect: no output (on the baseline the same command matches 33 lines: 11 in SKILL.md, 15 in the spec, 7 in references)

# 3. Verify hints unchanged in number (spec criterion 6): compare with baseline
git show 0bb5642:android/SKILL.md | rg -c 'Verify:'
rg -c 'Verify:' android/SKILL.md
# expect: equal

# 4. Edited defaults keep their "unless the project already" clause (spec criterion 3)
rg -n -F 'unless the project already' android/references/testing.md android/references/navigation.md android/references/compose-ui.md | rg -F -e 'enableAccessibilityChecks' -e 'ResultEventBus' -e 'NavigationBackHandler' -e 'generateLocaleConfig'
# expect: four lines

# 5. Traceability (spec criterion 4): each new wording has a spec twin
for p in 'only in instrumented tests on API 34+' 'use separate receivers when both flags are needed' 'alarm, timer or calendar apps' \
  'plus the vintage engine wherever those JUnit 4 tests share a JVM test task' 'whatever carries it' \
  'entry points already covered by generated or default rules' 'Navigation Compose 2.8+' '`OnBackPressedCallback` in View/Fragment code' \
  'a factory that only chains existing modifiers needs no Node' 'below the screen entry that resolve dependencies directly' \
  'UI state that must survive configuration changes' 'compileSdk 33+' 'the one exception is `HttpLoggingInterceptor` at `BODY`, in debug builds only' \
  'has chosen it (see the experimental-API must)'; do
  printf '%-75s android=%s spec=%s\n' "$p" "$(rg -c -F "$p" android | awk -F: '{s+=$2} END{print s+0}')" "$(rg -c -F "$p" specs/android-guidelines.md || echo 0)"
done
# expect: both counts >= 1 on every row

# 6. SKILL-only and spec-only wordings present
rg -c -F -e 'check_elf_alignment.sh' -e 'on those stacks and versions' android/SKILL.md     # expect: 2
rg -c -F 'it is unsupported on Robolectric' android/references/testing.md                   # expect: 1
rg -c -F 'In Compose, use `NavigationBackHandler`' android/references/navigation.md         # expect: 1
rg -c -F -e 'not on Robolectric' -e 'in Compose, `NavigationBackHandler`' specs/android-guidelines.md   # expect: 2

# 7. No client names (spec criterion 5)
rg -n -i 'mywalkup|nelnet|crmb' android specs
# expect: no output

# 8. Diff review: only the intended lines changed
git diff --numstat
# expect: android/SKILL.md 11 11; compose-ui.md 2 2; di-and-concurrency.md 1 1; navigation.md 2 2; testing.md 2 2; specs/android-guidelines.md 16 15
git diff -U0 -- android specs | rg '^[-+]' | rg -v '^(\+\+\+|---)'
```

## Git steps

```sh
git switch -c fix/android-rules-remainder main
# commit 1: rules
git add android/SKILL.md android/references/testing.md android/references/navigation.md android/references/di-and-concurrency.md android/references/compose-ui.md
git commit -m "fix(android): correct fourteen remaining rules after the remainder audit"
# commit 2: spec mirror
git add specs/android-guidelines.md
git commit -m "docs(specs): mirror the 2026-10-21 remainder corrections"
# commit 3: decision record (the four research docs are untracked, as with the earlier handoff)
git add research/android/portable-rules-remainder-audit.md research/android/portable-rules-remainder-audit-review.md research/android/portable-rules-remainder-second-review.md research/android/portable-rules-remainder-handoff.md
git commit -m "docs(research): add the remainder audit, its reviews and the implementation handoff"
git push -u origin fix/android-rules-remainder
```

PR body: list the eleven SKILL edits by line, the seven reference edits, the sixteen spec edits, and the check outputs. Every number in the body comes from a command above, not an estimate. Open it with `gh pr create` against `main`.

## Evidence summary (for the PR body, not for re-verification)

- R01: `ui-test-junit4-accessibility-android` 1.12.1 `AndroidComposeTestRuleExt.android.kt` carries `@RequiresApi(34)`, logs "Accessibility checks are currently not supported by Robolectric" (b/332778271) and installs the validator; `ComposeTestRuleExt.android.kt` throws `NotImplementedError` for non-`AndroidComposeTestRule` rules.
- R02: broadcasts guide, context-registered receivers step 4, including "partition the broadcasts among different broadcast receivers".
- R03: Play policies 10158779 and 10467955 list permitted core purposes; the exact-alarm policy lists "an alarm or timer app" and "a calendar app that shows event notifications".
- R04: android-junit-framework README adds Vintage "if you also have JUnit 4 tests (e.g. for Robolectric)" in the JVM configuration; instrumentation is configured separately.
- R05: Navigation 3 1.2.0 (2026-09-23) ships the Result API; the return-results guide uses the bus for both `ResultEffect` and `conflateAsState` and names `rememberSaveable`, `SavedStateHandle` or the `NavKey` for persistence; `navigation3-runtime` 1.2.0 `ResultEventBus.kt:47` errors with "No ResultEventBus has been provided" without the decorator; `navigation3-ui` 1.2.0 `NavDisplay.kt` defaults `entryDecorators` to the saveable-state decorator only and passes the list unchanged.
- R06: page-sizes guide sections "Check the alignment of ELF segments", `zipalign -c -P 16 -v 4`, and "Test your app in a 16 KB environment"; the APK Analyzer section identifies native code and shows alignment warnings without naming ELF.
- R07: AGP `proguard-common.txt` keeps View setters/getters and Activity `onClick` handlers with `-keepclassmembers`; AAPT2 `ProguardRules.cpp` emits `-keep class X { <init>(...); }` for manifest components and the two-argument constructor for layout Views under `android.useMinimalKeepRules` (default true); the `-if class **.R$layout` form needs `android.useConditionalKeepRules` (default false).
- R08: type-safety guide ("available as of Navigation 2.8.0", Navigation Compose and Kotlin DSL, Safe Args for XML); NavigationEvent 1.0.0 (2025-11-19) `navigationevent-compose` provides `NavigationBackHandler`; predictive-back guide documents `OnBackPressedCallback` for Views and Fragments.
- R09, R11: custom-modifiers and state-hoisting guides, quoted in the first review.
- R10: Hilt 1.2.0 assisted `hiltViewModel()`; 1.3.0-alpha02 and 1.3.0 published sources deprecate the `androidx.hilt.navigation.compose` overloads with `ReplaceWith` to `androidx.hilt.lifecycle.viewmodel.compose.hiltViewModel`.
- O01: app-languages guide (AGP 8.1, compileSdk 33 or higher, `resources.properties`, `unqualifiedResLocale`; `AppCompatActivity` only for the in-app picker path).
- O02: OkHttp logging-interceptor README (repository `lysine-dev/okhttp`, ID 5152285, where `square/okhttp` redirects) warns that HEADERS/BODY can leak credentials and bodies.
- O03: material3 1.4.0-beta01 (2025-07-30) removed all `ExperimentalMaterial3ExpressiveApi` APIs; 1.5.0-alpha18 (2026-04-22) added a `Material3ExpressiveApi` annotation that needs no opt-in; stable 1.4.0, beta 1.5.0-beta02 (2026-10-21).

## Out of scope

- Any rule not named above, including `SKILL.md:37` (decorator list), the StateFlow assertion default on `testing.md:7`, and the seven proposals implemented in PR #3.
- Adopting `ResultEventBus` as a default, adding `redactHeader()`, or scoping the privacy must to release builds.
- Editing the three review documents; their errors are recorded above and in the session that produced this handoff.
- Re-verifying external sources. Open limits that stay open: whether APK Analyzer checks ELF segments specifically, what the Accessibility Test Framework reports on Robolectric, and R07 remaining an uncompiled derivation.
