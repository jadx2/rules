# Review of the portable Android ruleset handoff

**Date:** 2026-10-21 · **Reviews:** `research/android/portable-rules-evaluation-handoff.md`
**Status:** findings only, no files changed. Written so the handoff's author can re-check each claim.

## Scope and method

- Read all ten rule files, `specs/android-guidelines.md`, the gap analysis and the git history of the rules.
- Verified each of the seven proposals against primary sources fetched on 2026-10-21. Quotes below are verbatim from those fetches.
- android/skills was checked at today's `main`, not the `ed0a272` snapshot named in `specs/android-guidelines.md:5`.
- Verification commands are listed at the end so each claim can be reproduced.

## Summary

| # | Local rule | Handoff proposal | Verdict | Kind |
|---|---|---|---|---|
| 1 | `android/SKILL.md:64` `@Immutable`/`@Stable`/`ImmutableList` ban | Reconsider the blanket prohibition | Needs qualification, opposite direction: keep the ban, fix the rationale | Technical correction |
| 2 | `android/SKILL.md:62` `rememberUpdatedState` | Narrow to effects that must not restart | Agree | Technical correction |
| 3 | `android/SKILL.md:83-86` inset prohibitions | Replace with ownership/consumption guidance | Disagree with the rewrite; narrow line 83 only | Technical correction (line 83) |
| 4 | `android/SKILL.md:98` Photo Picker | Allow justified broad access | Agree | Technical correction |
| 5 | `android/SKILL.md:104-105` consumer rules | Verify per artifact/version | Agree; "Retrofit 2.9+" is wrong | Technical correction |
| 6 | `android/SKILL.md:21` version catalog must | Make it a default | Agree | Reclassification |
| 7 | `android/SKILL.md:42` `@Serializable`, no Gson | Keep the project's serializer | Agree, keep kotlinx as the default | Reclassification |

## Approach

Agree with the curation approach. Two notes:

- No third rule category is needed. The must/default split at `android/SKILL.md:9-10` already separates universal rules from convention-dependent ones. The defect is that two musts (lines 21 and 42) are convention-dependent and escape the override clause at line 10, which only covers defaults.
- Provenance already lives in `specs/android-guidelines.md:189-197`. Keep review dates there, not in the rules.

The toolkit does not drive any of the seven changes. It uses a version catalog (`CLAUDE.md`, "Dependency versions are centralized in `gradle/libs.versions.toml`"), kotlinx.serialization with the Retrofit converter (`.claude/skills/project-scaffold/SKILL.md:317-321`, `:412`), and contains no mention of `@Immutable`, `@Stable`, `ImmutableList` or `rememberUpdatedState` anywhere. Its only inset code is `Modifier.systemBarsPadding()` on the `NavHost` (`project-scaffold/SKILL.md:266`), which is where rule 83's wording comes from.

## Proposal 1: `@Immutable`, `@Stable`, `ImmutableList` on collection models

**Verdict:** needs qualification. The handoff assumes the rule overgeneralizes; current Google guidance says the same thing as the rule. Keep the prohibition, fix its explanation, add the documented escape hatch and a scope guard.

**Local evidence**
- `android/SKILL.md:64` and `specs/android-guidelines.md:99`: "Never add… strong skipping (default since Kotlin 2.0.20) already skips on instance equality, and the annotations force an O(n) `equals` on every recomposition."
- The gap analysis recommended the softer form: "Use `ImmutableList` or a stability config file only when profiling shows a problem" (`research/android/2026-10-android-toolkit-gap-analysis.md:271`, as a default). The hard "never" entered as a must in commit `088975e`.
- `android/references/compose-ui.md:16` keeps `@Immutable` for flat token classes, consistent with the rule's last sentence.

**External evidence**
- Strong skipping page (https://developer.android.com/develop/ui/compose/performance/stability/strongskipping): "Strong skipping is enabled by default in Kotlin 2.0.20." "Unstable parameters are compared using instance equality (`===`)". "Be cautious when annotating models with collections: Adding `@Stable` or `@Immutable` to models containing collections (or replacing `List` with `ImmutableList`) causes Compose to compare every element in the collection with `.equals()` on each recomposition pass. For large collections or models used in lazy layouts… comparing every element can be slower than using instance equality (`===`) or recomposing directly." "Only annotate classes with cheap `.equals()` checks".
- Stability fix page (https://developer.android.com/develop/ui/compose/performance/stability/fix): "Consider `ImmutableList` or stability annotations on collection-holding models only when you can't preserve the collection's instance reference and benchmarks confirm that structural equality checks are faster than recomposing." Warning: "Incorrectly annotating a class could cause recomposition to break or degrade performance." "you shouldn't attempt to make every composable skippable or force structural equality on every parameter."
- android/skills `jetpack-compose/jetpack-compose-performance/SKILL.md`: "never add `@Immutable` or `@Stable` to models containing collections (`List`, `Set`, `Map`)"; "**never** recommend replacing `kotlin.collections.List<T>` with `ImmutableList<T>`"; scope guard: "leave the existing annotations untouched and do not flag them", "Never blindly grep and flag all `@Stable`/`@Immutable` classes across a codebase".
- Why the rationale wording is wrong: a data class `equals` short-circuits on identity before comparing fields. Verified by disassembling `kotlin.Pair` from `kotlin-stdlib-2.2.21.jar` (bundled with the Gradle 9.3.1 wrapper): the first instructions are `aload_0; aload_1; if_acmpne`. Kotlin's `AbstractList.equals` starts with `if (other === this) return true` (JetBrains/kotlin master, `libraries/stdlib/src/kotlin/collections/AbstractList.kt`). So the O(n) walk only runs when a new collection instance arrives; with `===` that same case recomposes instead. Note: the identity shortcut is compiler behavior, not a spec guarantee. The Kotlin spec's illustrative `equals` expansion has no identity check and the spec says its examples "are not guaranteed to exactly match the ones generated by kotlin compiler".
- No hidden tooling pressure: compose-lints `ComposeUnstableCollections`, which recommends immutable collections, "is disabled by default" (https://slackhq.github.io/compose-lints/rules/). It does not collide with the no-suppression must at `android/SKILL.md:114`.

**Smallest adjustment**
Keep "don't add". Replace the why with: annotating, or using `ImmutableList`, switches the comparison from instance to structural equality, which walks the collection whenever a new instance is emitted; prefer preserving the instance in the ViewModel. Add the docs' escape hatch: only when the instance can't be preserved and a benchmark shows it beats recomposing. Add: leave existing annotations in a project alone.

## Proposal 2: `rememberUpdatedState` for effects calling lambda params

**Verdict:** agree.

**Local evidence**
- `android/SKILL.md:62`: "Effects that call lambda params use `rememberUpdatedState`."
- The qualifier was lost during drafting: the gap analysis wrote "any **long-lived** effect that calls a lambda parameter uses `rememberUpdatedState`" at line 262, then dropped "long-lived" in checklist item A16 at line 741, which the spec (`specs/android-guidelines.md:98`) and SKILL.md inherited.

**External evidence**
- Side effects page (https://developer.android.com/develop/ui/compose/side-effects): "in some situations you might want to capture a value in your effect that, if it changes, you do not want the effect to restart. In order to do this, it is required to use `rememberUpdatedState`". "If the change of a variable shouldn't cause the effect to restart, the variable should be wrapped in `rememberUpdatedState`."
- android/skills performance skill: "Use `rememberUpdatedState` when long-running or looping coroutines in `LaunchedEffect` reference callbacks or parameters."
- Lifecycle compose page (https://developer.android.com/topic/libraries/architecture/compose): "`LifecycleStartEffect` accepts keys. When the key changes, it triggers the block to run again." Passing the lambda as a key is therefore the alternative when a restart is acceptable.

**Smallest adjustment**
"An effect that must not restart when a lambda param changes reads it through `rememberUpdatedState`; otherwise pass the lambda as a key."

## Proposal 3: inset prohibitions

**Verdict:** disagree with replacing the concrete rules; agree with narrowing line 83.

**Local evidence**
- `android/SKILL.md:85-86` entered with commit `088975e` (spec lines 90-91), sourced from android/skills.
- `android/SKILL.md:83` "never `systemBarsPadding()` on a NavHost or root" traces to the gap analysis's inference about the toolkit scaffold (`research/android/2026-10-android-toolkit-gap-analysis.md:23`, `:628`, `:654`), not to an upstream rule.
- Spec acceptance criterion 6 (`specs/android-guidelines.md:174`) wants musts that an audit can check; concrete prohibitions satisfy it, abstract ownership guidance does not.

**External evidence**
- android/skills `system/edge-to-edge/SKILL.md` backs lines 85-86 almost verbatim: adaptive scaffolds "don't propagate PaddingValues to their inner contents"; "You **MUST** apply insets to **individual** screens or components"; "**DO NOT** apply `safeDrawingPadding` or similar modifiers to the `NavigationSuiteScaffold` parent"; lists: "DO NOT apply it as a `Modifier.padding()` to the list's parent container"; "Add `android:windowSoftInputMode="adjustResize"`"; "The padding modifier **MUST** be placed before `Modifier.verticalScroll()`"; "Do NOT use `Modifier.imePadding()` if the parent already accounts for the IME with `contentWindowInsets`"; "Scaffold's default `contentWindowInsets` does NOT contain IME insets."
- The same skill does not mention `systemBarsPadding`, `NavHost` or root padding at all.
- Google's insets page (https://developer.android.com/develop/ui/compose/system/insets-ui) shows `Box(Modifier.safeDrawingPadding())` at the root as a valid approach and states the trade-off: "it also means that none of the app will draw behind the system UI to achieve an edge-to-edge effect."

**Smallest adjustment**
Reword line 83 to: "Don't pad the whole NavHost or root with system-bar insets; it blocks edge-to-edge on every screen. Apply insets per screen through Scaffold `contentWindowInsets` or per component." Leave lines 85-86 as they are.

## Proposal 4: Photo Picker

**Verdict:** agree. As written, the rule would make an agent refuse a legitimate gallery or media-management feature.

**Local evidence**
- `android/SKILL.md:98` and `specs/android-guidelines.md:137` (commit `088975e`): "Media selection uses the Photo Picker (`PickVisualMedia`), not `READ_MEDIA_*`."

**External evidence**
- Play Photo and Video Permissions policy (https://support.google.com/googleplay/android-developer/answer/14115180): the permissions may be requested only when "system pickers (like the Android Photo Picker), are not sufficient for your app to provide core functionality"; "Apps whose core functionality revolves around broad access to Photos and Videos may use the permissions above"; a custom picker still needs a declaration; compliance mandatory since 2025-05-28.
- Android 14 partial access page (https://developer.android.com/about/versions/14/changes/partial-photo-video-access): "If you maintain your own gallery picker using storage permissions… adapt your implementation to use the new `READ_MEDIA_VISUAL_USER_SELECTED` permission."
- android/skills `security/android-permissions-security/SKILL.md` says "Prefer `ActivityResultContracts.PickVisualMedia()` over requesting wide storage permissions", prefer rather than never, and names no exceptions.

**Smallest adjustment**
"Media selection uses the Photo Picker. Request `READ_MEDIA_*` only when broad library access is the app's core function, with Play's declaration and `READ_MEDIA_VISUAL_USER_SELECTED` handled." Keep the verify grep as a review prompt.

## Proposal 5: R8 consumer rules

**Verdict:** agree, and one stated fact is wrong.

**Local evidence**
- `android/SKILL.md:105` and `specs/android-guidelines.md:143`: "No rules for libraries that ship their own (AndroidX, kotlinx, coroutines, Retrofit 2.9+)".

**External evidence**
- Retrofit 2.9.0 is tagged 2020-05-20. The bundled rules that keep generic signatures under R8 full mode were committed after it: `51e81a8` 2021-06-22 "Add keep rule to retain generic signature of Call", `007a706` 2021-08-07 "Keep signature for Response<T>", `59d302a` 2023-05-05 "Add keep rule for keeping generic signature of return types (#3886)", `ef8d867` 2023-08-15 "…generic signatures of retrofit2.Response (#3910)". No release shipped between 2.9.0 and 2.10.0 (2024-03-18), whose changelog says "Add shrinker rules to retain the generic signatures of built-in types (`Call`, `Response`, etc.)". The shipped `retrofit2.pro` comments: "R8 full mode strips generic signatures from return types if not kept." So Retrofit 2.9.0 under full mode did need app-side rules.
- kotlinx.serialization README: "By default, proguard rules are supplied with the library." "However, these rules do not affect serializable classes if they have named companion objects."
- Gson `Troubleshooting.md`: default rules "included since Gson 2.11.0".
- R8 compatibility FAQ (https://r8.googlesource.com/r8/+/refs/heads/main/compatibility-faq.md) confirms the `<init>()` claim: "The default constructor (`<init>()`) is not implicitly kept when a class is kept."
- Library optimization page (https://developer.android.com/topic/performance/app-optimization/library-optimization): consumer rules "are bundled into library artifacts (AARs or JARs) and get consumed automatically"; JAR rules live in `META-INF/proguard/`. It lists no libraries, so the local list is the rule author's, not Google's.

**Smallest adjustment**
Change "Retrofit 2.9+" to "Retrofit 2.10+". Reword the sentence to "Don't duplicate rules the artifact already bundles; check its `META-INF/proguard` or `proguard.txt`. AndroidX, kotlinx.coroutines, kotlinx.serialization (except named companions), Retrofit 2.10+ and Gson 2.11+ bundle theirs."

## Proposal 6: version catalog as a must

**Verdict:** agree.

**Local evidence**
- `android/SKILL.md:21`, `specs/android-guidelines.md:21` (commit `9864981`, carried from the gap analysis's KEEP at line 97, which described the toolkit's own convention).
- `android/SKILL.md:10` lets a project convention override defaults only, so this must would force a catalog into a project that centralizes versions in `buildSrc` or `ext`.

**External evidence**
- Migrate-to-catalogs page (https://developer.android.com/build/migrate-to-catalogs): the only strength used is "we recommend"; "A build can consume dependencies and plugins from build scripts and catalogs simultaneously". Nothing states a requirement.

**Smallest adjustment**
Line 21: "Declare dependencies through the project's mechanism; never hard-code a version in a module build file when the project centralizes versions elsewhere. New projects use a version catalog." Keep the verify grep, scoped to projects with a catalog. Move the catalog default into `android/references/build-and-modules.md`.

## Proposal 7: `@Serializable` DTOs and "No Gson in new code"

**Verdict:** agree. Keep the DTO-separation must; make the serializer follow the project with kotlinx.serialization as the new-project default.

**Local evidence**
- `android/SKILL.md:42`, `specs/android-guidelines.md:80` (commit `9864981`).
- Internal inconsistency: `android/references/data-and-networking.md:5` already yields "unless the project already uses another networking library", and the DI must at `android/SKILL.md:47` already uses the "never introduce a second library" pattern. Line 42 contradicts both for a Gson or Moshi project.

**External evidence**
- Gson README: "Gson is currently in maintenance mode"; "Kotlin's non-`null` types or constructors with default arguments are not supported"; "When using languages other than Java, prefer a JSON library with explicit support for that language."
- Library optimization page: "the open ended reflection of Gson isn't appropriate for usage in Android apps."
- These justify the default, not a must that overrides an existing project.

**Smallest adjustment**
"DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer and never add a second one; new projects use kotlinx.serialization." Optional, contextual: in Gson projects, DTO fields stay nullable and the mapper validates, because Gson bypasses Kotlin null-safety and default values.

## Omissions in the handoff

- It did not check whether the recommended tooling pushes agents toward `ImmutableList`. It does not: `ComposeUnstableCollections` is off by default.
- It did not notice that Google's docs moved toward the local rule 64, which reverses the direction of its first proposal.
- It did not trace line 83 to the toolkit inference versus lines 85-86 to the upstream skill; that distinction is what decides proposal 3.
- Any SKILL.md change needs the matching spec line changed too, or acceptance criterion 4 (`specs/android-guidelines.md:172`) breaks. Affected spec lines: 21, 80, 98, 99, 126, 137, 143.
- Low priority, outside the seven: `android/SKILL.md:66` names "Material 3 Expressive" as experimental. True on stable material3 1.4.0, but the 1.5.0 alphas graduated buttons, menus, FABs, split buttons, floating toolbars, top app bars and list items to non-experimental, and the release page lists 1.5.0-beta01 on 2026-10-07. `specs/android-guidelines.md:113` already anticipates this. Optional fix: "APIs behind an `@Experimental` opt-in" instead of naming components.

## Unverified or out of scope

- Styles, Grid/FlexBox and `mediaQuery` status at `android/SKILL.md:66`.
- material3 release notes beyond the first 100k characters of the page.
- The handoff's statement that the `PendingIntent` rule is already correct: not re-checked, outside the seven proposals.
- Everything in the rules outside the seven proposals.

## Prioritized recommendation

1. Technical corrections, do first:
   - Line 105: "Retrofit 2.9+" to "2.10+", plus "check the artifact" wording.
   - Line 98: add the core-function exception and declaration.
   - Line 62: narrow to effects that must not restart; otherwise key on the lambda.
   - Line 64: keep the ban, rewrite the rationale, add the benchmark escape hatch and "leave existing annotations alone".
   - Line 83: root/NavHost wording; keep 85-86.
2. Reclassification for consistency with line 10:
   - Line 21: project mechanism first, catalog as the new-project default.
   - Line 42: project serializer first, no second library, kotlinx as the default.
3. Optional:
   - Line 66 phrasing.
   - Gson-project nullability note in the data reference.
4. Update the spec lines listed above in the same change.

## Reproduction

```sh
# Retrofit rule history and release dates
gh api 'repos/square/retrofit/commits?path=retrofit/src/main/resources/META-INF/proguard/retrofit2.pro&per_page=30' \
  --jq '.[] | "\(.commit.committer.date[:10]) \(.sha[:7]) \(.commit.message | split("\n")[0])"'
gh api repos/square/retrofit/releases/tags/2.10.0 --jq '.published_at[:10]'
sha=$(gh api repos/square/retrofit/git/refs/tags/2.9.0 --jq '.object.sha'); gh api "repos/square/retrofit/git/tags/$sha" --jq '.tagger.date[:10]'

# Data class equals short-circuits on identity (kotlin.Pair is a data class)
jar=$(find ~/.gradle -name 'kotlin-stdlib-[12]*.jar' | /usr/bin/grep -v sources | head -1)
javap -c -cp "$jar" kotlin.Pair | sed -n '/public boolean equals/,/ireturn/p'

# Local provenance
git show 088975e -- specs/android-guidelines.md | /usr/bin/grep -n '^+' | /usr/bin/grep -E 'Immutable|contentPadding|imePadding|Photo Picker|ship their own'
git log --oneline -S'No Gson in new code' -- specs/android-guidelines.md
sed -n '262p;271p;741p' research/android/2026-10-android-toolkit-gap-analysis.md

# Toolkit conventions
for p in CLAUDE.md .claude/skills/project-scaffold/SKILL.md; do
  gh api "repos/EdwinLovo/android-claude-toolkit/contents/$p" --jq .content | base64 -d \
    | /usr/bin/grep -n -i -E 'libs.versions|serializ|systemBarsPadding|@Immutable|rememberUpdatedState'
done
```

External pages used, all fetched 2026-10-21:
- https://developer.android.com/develop/ui/compose/performance/stability/strongskipping
- https://developer.android.com/develop/ui/compose/performance/stability/fix
- https://developer.android.com/develop/ui/compose/side-effects
- https://developer.android.com/topic/libraries/architecture/compose
- https://github.com/android/skills/blob/main/jetpack-compose/jetpack-compose-performance/SKILL.md
- https://github.com/android/skills/blob/main/system/edge-to-edge/SKILL.md
- https://github.com/android/skills/blob/main/security/android-permissions-security/SKILL.md
- https://developer.android.com/develop/ui/compose/system/insets-ui
- https://support.google.com/googleplay/android-developer/answer/14115180
- https://developer.android.com/about/versions/14/changes/partial-photo-video-access
- https://developer.android.com/topic/performance/app-optimization/library-optimization
- https://r8.googlesource.com/r8/+/refs/heads/main/compatibility-faq.md
- https://github.com/square/retrofit (CHANGELOG.md, README.md, retrofit2.pro)
- https://github.com/Kotlin/kotlinx.serialization/blob/master/README.md
- https://github.com/google/gson (README.md, Troubleshooting.md)
- https://developer.android.com/build/migrate-to-catalogs
- https://slackhq.github.io/compose-lints/rules/
- https://developer.android.com/jetpack/androidx/releases/compose-material3
- https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/collections/AbstractList.kt
- https://kotlinlang.org/spec/declarations.html
