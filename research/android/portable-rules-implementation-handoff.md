# Handoff: implement the Android rule corrections

**Date:** 2026-10-21 · **Baseline:** `02e2f97` on `main`
**Decision record:** `research/android/portable-rules-evaluation-handoff.md` (proposals), `…-review.md` (first review), `…-verdict.md` (second review), and the final pass summarized here. The evidence is settled; do not re-verify sources.

## Task

Apply the edits below to four files, mirror them in the spec, run the checks, commit on a branch and open a PR. Nothing else changes: no other rules, no new files, no reformatting of untouched lines.

Working rules for this repo:
- `fd` is not installed; use `rg` or `find`. Call `/usr/bin/grep` directly.
- Spec edits: match by text, not by line number. Edit S0 inserts a line and shifts everything below it.
- Never amend or force-push. No `Co-Authored-By` or "Generated with" lines.
- Branch from `main`; this repo merges through PRs (#1, #2).

## Edits to `android/SKILL.md`

Line numbers are baseline. Each edit replaces the whole line unless stated.

**E1, lines 21-22** (proposal 6, reclassification)

Current:
```markdown
- Dependencies live in the version catalog, never hard-coded in module build files.
  - Verify: `rg '"[a-z0-9.-]+:[a-z0-9.-]+:[0-9]' --glob '*.gradle.kts'` returns nothing.
```
Replacement:
```markdown
- Declare dependencies through the project's existing mechanism; don't hard-code versions in module build files when the project centralizes them elsewhere.
  - Verify: where the project centralizes versions, review new `rg '"[a-z0-9.-]+:[a-z0-9.-]+:[0-9]' --glob '*.gradle.kts'` hits for versions that bypass that mechanism.
```

**E2, line 42** (proposal 7, reclassification). Line 43 stays.

Current:
```markdown
- DTOs are `@Serializable` data-layer types mapped to domain models, never in domain or UI signatures. No Gson in new code.
```
Replacement:
```markdown
- DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer; don't introduce a second serializer or migrate without asking.
```

**E3, line 62** (proposal 2, technical). Line 63 stays.

Current:
```markdown
- Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. Effects that call lambda params use `rememberUpdatedState`.
```
Replacement:
```markdown
- Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. For keyed effects, read a lambda parameter through `rememberUpdatedState` when changing it must not restart the effect; use the lambda as a key when changing it should restart the effect.
```

**E4, line 64** (proposal 1, technical; final wording from the third pass)

Current:
```markdown
- Never add `@Immutable`, `@Stable` or `ImmutableList` to models that hold collections: strong skipping (default since Kotlin 2.0.20) already skips on instance equality, and the annotations force an O(n) `equals` on every recomposition. Classes without collection fields (e.g. theme tokens) may keep `@Immutable`.
```
Replacement:
```markdown
- With strong skipping (default since Kotlin 2.0.20), don't add `@Immutable`, `@Stable` or `ImmutableList` to collection-bearing models; preserve collection instances instead. Making an otherwise unstable parameter stable switches instance comparison to `equals`, which can walk the collection whenever a different instance arrives. Make an exception only when the stability contract is valid, the collection reference cannot be preserved and benchmarks show structural comparison beats recomposition. Don't change existing annotations or collection types merely to comply with this rule. Collection-free classes (e.g. theme tokens) may keep valid `@Immutable`.
```

**E5, line 66** (flagged omission, optional but approved)

Current:
```markdown
- No experimental or alpha Compose APIs (Material 3 Expressive, Styles, Grid/FlexBox, `mediaQuery`) and no new `@OptIn` or module-wide opt-in flag without asking. Check the project's Compose version before using an API that needs a newer one.
```
Replacement:
```markdown
- No experimental Compose APIs requiring an opt-in, alpha Compose APIs, new `@OptIn` annotations or module-wide opt-in flags without asking. Check the project's Compose version before using an API that needs a newer one.
```

**E6, line 83** (proposal 3, technical). Lines 84-86 stay.

Current:
```markdown
- Enable edge-to-edge (`enableEdgeToEdge()`) and apply insets once; never `systemBarsPadding()` on a NavHost or root.
```
Replacement:
```markdown
- Enable edge-to-edge (`enableEdgeToEdge()`) and apply each inset once. When screens should draw behind system bars, don't apply system-bar padding to the whole NavHost or root; apply the required insets per screen or component. Root padding is valid when keeping all content within safe bounds is intentional.
```

**E7, line 98** (proposal 4, technical). Line 99 stays as a review prompt.

Current:
```markdown
- Media selection uses the Photo Picker (`PickVisualMedia`), not `READ_MEDIA_*`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```
Replacement:
```markdown
- Use the Photo Picker (`PickVisualMedia`) for user-selected photos and videos. Request `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only when broad library access is essential to core functionality and system pickers are insufficient; satisfy applicable Play declaration and review requirements. For a permission-based gallery on Android 14+ targeting API 34+, handle partial access and reselection with `READ_MEDIA_VISUAL_USER_SELECTED`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```

**E8, line 105** (proposal 5, technical). Line 104 stays.

Current:
```markdown
- Keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. No rules for libraries that ship their own (AndroidX, kotlinx, coroutines, Retrofit 2.9+) or for Activity and View classes. Under full mode `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.
```
Replacement:
```markdown
- Keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. Don't duplicate consumer rules the exact artifact/version already bundles: check its JAR or AAR (`META-INF/proguard/`, `META-INF/com.android.tools/`, `proguard.txt`) and add only missing, verified narrow rules. AndroidX, kotlinx.coroutines, kotlinx.serialization (except named companions), Retrofit 2.10+ and Gson 2.11+ bundle theirs. No rules for Activity and View classes. Under full mode `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.
```
One judgment call, decided for the examples: the second review preferred dropping the "AndroidX … bundle theirs" sentence as a version-dated list. Keep it unless Jaim says otherwise; the versions were verified on 2026-10-21.

## Edits to reference files

**E9, `android/references/build-and-modules.md:3`**

Current:
```markdown
The SKILL.md musts (ask before changing toolchain or SDK levels, version catalog, new code in the implied module, no module without asking) apply; this file adds the how.
```
Replacement:
```markdown
The SKILL.md musts (ask before changing toolchain or SDK levels, follow the project's dependency mechanism, new code in the implied module, no module without asking) apply; this file adds the how.
```

**E10, `android/references/build-and-modules.md`, insert after line 8** ("New projects use the latest stable Kotlin and AGP…"):
```markdown
- New projects use a version catalog, unless the project or template already establishes another dependency-management mechanism.
```

**E11, `android/references/data-and-networking.md:5`**

Current:
```markdown
- Use Retrofit 3 + OkHttp + the kotlinx.serialization converter, unless the project already uses another networking library.
```
Replacement:
```markdown
- Use Retrofit 3 + OkHttp unless the project already uses another networking library. Use kotlinx.serialization for DTOs and its converter when using Retrofit, unless the project already uses another serializer.
```

## Edits to `specs/android-guidelines.md`

Match by text. Baseline line numbers are given for orientation only.

**S0, after line 5** (the "Revised 2026-10-19" paragraph), insert:
```markdown
**Revised 2026-10-21** after an independent review of seven rules against current Android, Compose, Play and library sources (`research/android/portable-rules-evaluation-review.md`, `…-verdict.md`): stability annotations, `rememberUpdatedState`, root insets, Photo Picker, consumer keep rules, version catalog and serializer rules narrowed or reclassified.
```

**S1, line 21.** Replace the single must with two lines:
```markdown
- **Must:** declare dependencies through the project's existing mechanism; never hard-code versions in module build files when the project centralizes them elsewhere.
- **Default:** new projects use a version catalog.
```

**S2, line 76.** Replace the final sentence "Existing projects keep their library." with "Existing projects keep their networking library and serializer." The rest of the line stays.

**S3, line 80.** Replace with:
```markdown
- **Must:** DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer; never introduce a second serializer or migrate without asking.
```

**S4, line 98.** Replace with:
```markdown
- **Must:** lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. For keyed effects, a lambda parameter is read through `rememberUpdatedState` when changing it must not restart the effect, and used as a key when changing it should.
```

**S5, line 99.** Replace with:
```markdown
- **Must:** with strong skipping (default since Kotlin 2.0.20), don't add `@Immutable`, `@Stable` or `ImmutableList` to collection-bearing models; preserve collection instances instead. Why: making an unstable parameter stable switches instance comparison to `equals`, which can walk the collection whenever a different instance arrives. Exception only when the stability contract is valid, the reference cannot be preserved and benchmarks show structural comparison beats recomposition. Existing annotations are not changed merely to comply. Collection-free classes may keep valid `@Immutable`.
```

**S6, line 101.** Replace with:
```markdown
- **Must:** no experimental Compose APIs requiring an opt-in, alpha Compose APIs, new `@OptIn` annotations or module-wide opt-in flags without asking. Check the project's Compose version before using an API that needs a newer one.
```

**S7, line 126.** Replace with:
```markdown
- **Must:** edge-to-edge enabled (`enableEdgeToEdge()`), each inset applied once. When screens should draw behind system bars, no system-bar padding on the whole NavHost or root; insets are applied per screen or component. Root padding is valid when keeping all content within safe bounds is intentional.
```

**S8, line 137.** Replace with:
```markdown
- **Must:** the Photo Picker (`PickVisualMedia`) for user-selected photos and videos. `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only when broad library access is essential to core functionality and system pickers are insufficient, with Play's declaration and review requirements met. A permission-based gallery on Android 14+ targeting API 34+ handles partial access and reselection with `READ_MEDIA_VISUAL_USER_SELECTED`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`. Verify: review manifest grep hits.
```

**S9, line 143.** Replace with:
```markdown
- **Must:** keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. Don't duplicate consumer rules the exact artifact/version already bundles (check `META-INF/proguard/`, `META-INF/com.android.tools/`, `proguard.txt`); add only missing, verified narrow rules. AndroidX, kotlinx.coroutines, kotlinx.serialization (except named companions), Retrofit 2.10+ and Gson 2.11+ bundle theirs. No rules for Activity and View classes. Under full mode, `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.
```

**S10, end of the "Research claims, rechecked" list** (last bullets of the file). Append:
```markdown
  - Retrofit's bundled R8 full-mode rules for generic signatures of `Call`, `Response` and service return types first shipped in 2.10.0 (2024-03-18); 2.9.0 (2020-05-20) bundled rules but lacked them (square/retrofit CHANGELOG and `retrofit2.pro` history).
  - Google's stability pages now caution against `@Immutable`/`@Stable`/`ImmutableList` on collection-holding models and allow them only when the instance reference cannot be preserved and benchmarks confirm a gain (developer.android.com/develop/ui/compose/performance/stability/fix, 2026-10-21).
  - Play's Photo and Video Permissions policy covers `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only, allows them when system pickers cannot serve core functionality, and exempts private and enterprise device-management apps (support.google.com/googleplay/android-developer/answer/16935362).
```

## Checks before committing

Run from the repo root and record the outputs for the PR body.

```sh
# 1. SKILL.md stays under the 150-line limit (spec criterion 2)
wc -l android/SKILL.md

# 2. Old wording is gone everywhere
rg -n 'Retrofit 2\.9|READ_MEDIA_\*|No Gson in new code|never `systemBarsPadding|force an O\(n\)|as an automatic performance fix' android specs
# expect: no output

# 3. Verify hints unchanged in number (spec criterion 6): compare with baseline
git show 02e2f97:android/SKILL.md | rg -c 'Verify:'
rg -c 'Verify:' android/SKILL.md

# 4. New default carries the "unless the project already" clause (spec criterion 3)
rg -n 'version catalog' android/references/build-and-modules.md

# 5. Traceability (spec criterion 4): each edited SKILL rule has a spec twin
for p in "project's existing mechanism" "project's serializer" "For keyed effects" "collection-bearing models" "requiring an opt-in" "Root padding is valid" "READ_MEDIA_VISUAL_USER_SELECTED" "Retrofit 2.10+"; do
  printf '%-40s SKILL=%s spec=%s\n' "$p" "$(rg -c -F "$p" android/SKILL.md)" "$(rg -c -F "$p" specs/android-guidelines.md)"
done
# expect: both counts >= 1 on every row

# 6. No client names (spec criterion 5)
rg -n -i 'mywalkup|nelnet|crmb' android specs
# expect: no output

# 7. Diff review: only the intended lines changed
git diff --stat
git diff -U0 -- android specs | rg '^[-+]' | rg -v '^(\+\+\+|---)' | wc -l
```

## Git steps

```sh
git switch -c feature/android-rules-review-fixes main
# commit 1: rules
git add android/SKILL.md android/references/build-and-modules.md android/references/data-and-networking.md
git commit -m "fix(android): narrow seven musts after primary-source review"
# commit 2: spec mirror
git add specs/android-guidelines.md
git commit -m "docs(specs): mirror the 2026-10-21 Android rule corrections"
# commit 3: decision record (the four research docs are untracked; the gap analysis was committed the same way)
git add research/android/portable-rules-evaluation-handoff.md research/android/portable-rules-evaluation-review.md research/android/portable-rules-evaluation-verdict.md research/android/portable-rules-implementation-handoff.md
git commit -m "docs(research): add the portable Android rules review and its implementation handoff"
git push -u origin feature/android-rules-review-fixes
```

PR body: list the eight SKILL edits by line, the three reference/spec consequences, and the check outputs. Every number in the body comes from a command above, not an estimate. Open it with `gh pr create` against `main`.

## Out of scope

- Any rule not named above, including the `PendingIntent`, Activity/View keep-rule and Navigation 3 lines.
- Re-verifying external sources; the evidence and quotes are in the review and verdict files.
- Styles, Grid/FlexBox and `mediaQuery` status: unverified, which is why E5 stops naming components.
- The optional Gson nullability note: rejected in the second review, do not add.
