# Final verification verdict: portable Android rules

**Date:** 2026-10-21  
**Audit target:** `research/android/portable-rules-evaluation-review.md`  
**Earlier handoff:** `research/android/portable-rules-evaluation-handoff.md`  
**Local baseline:** `02e2f97037925c7cdd022bcc1dd70dbb54227074`  
**Status:** verification results and proposed edits only. The rules and spec have not been changed. This document records the preceding verification run; writing it did not rerun the external checks.

## Instructions for the final independent pass

- Review this verdict critically; do not assume either this document or the earlier review is correct.
- Scope remains the seven proposals and the flagged `android/SKILL.md:66` omission. Do not introduce unrelated rules, topics, files or refactors.
- Check the evidence-to-conclusion relationship, not just whether quotations exist. A quoted upstream rule is evidence of that source's recommendation, not proof of a universal technical invariant.
- Line numbers below refer to the baseline, before edits. Preserve the distinction between current file lines, historical file lines and numbered diff output.
- Do not implement the proposed edits during the review. Report remaining factual errors, missing qualifications or inconsistencies.

## Executive verdict

The review is substantially correct, but its proposed replacements should not be accepted verbatim. Retrofit's reported dates reproduce. The `javap` recipe selects a different stdlib version here, but reproduces the identity shortcut. Several local references and two uses of external evidence need correction.

| # | Decision on the review | Final position |
|---|---|---|
| 1 | Accept with changes | Keep the restriction on adding stability annotations or immutable collections as an automatic performance fix, with a measured exception. Correct the equality-cost explanation and avoid indiscriminate changes to existing annotations. |
| 2 | Accept with changes | Use `rememberUpdatedState` when a callback change must not restart a keyed effect. Use the callback as a key when its change should restart that effect; do not generalize the alternative to every effect API. |
| 3 | Accept with changes | Keep lines 85–86; change line 83 only. Unlike the review's replacement, permit intentional root padding when all content should remain inside safe bounds. |
| 4 | Accept with changes | Allow justified broad photo/video access. Name `READ_MEDIA_IMAGES` and `READ_MEDIA_VIDEO`, not `READ_MEDIA_*`; distinguish Play requirements from Android 14 partial-access behavior. |
| 5 | Accept with changes | Verify consumer rules per artifact/version. Retrofit 2.9.0 lacked the identified later rules, but `2.10+` is not a timeless guarantee of complete coverage. Remove the blanket library list. |
| 6 | Accept | Follow the project's dependency mechanism; retain catalogs as a new-project default in the existing reference file. |
| 7 | Accept with changes | Keep DTO separation and the project's serializer, with kotlinx.serialization as the default. Introducing another serializer or migrating requires asking, rather than an absolute prohibition even when authorized. |

### Disagreement with the handoff: proposal 1

The evidence supports the review's practical direction: do not add annotations or immutable collection types automatically to obtain skipping. Current documentation recommends preserving collection references and permits a benchmark-backed exception when that is not possible.

The handoff was correct to challenge the equality-cost rationale. Neither the current rule's “O(n) on every recomposition” nor the review's replacement claim that a new collection instance necessarily causes a complete traversal is accurate. Equality can terminate through identity, size, earlier field differences or an early element mismatch. A new wrapper can also retain the same collection reference. Structural comparison **can** cost O(n); it does not necessarily do so.

Therefore the final rule is a qualified restriction, not an unconditional `never`, and the exception must still satisfy the annotation's stability contract.

### Disagreement with the handoff: proposal 3

The evidence supports retaining the concrete list/IME rules at lines 85–86 and correcting line 83, rather than replacing the entire block with abstract guidance. The cited upstream skill contains the specific list, scaffold and IME instructions.

However, the review's proposed “Don't pad the whole NavHost or root” is still too broad. Google's insets page presents root `safeDrawingPadding()` as a valid approach and describes its trade-off. The prohibition should be conditional on wanting screens to draw behind system bars. Upstream provenance and auditability support retaining useful concrete guidance; neither alone establishes universal correctness.

## External quotation and claim verification

“Matches” allows Markdown/HTML formatting and whitespace normalization. An explicitly marked ellipsis is an excerpt, not a complete quotation. Checks were against the cited source content, including complete-document checks where an excerpt-only retrieval could produce a false absence.

| Review lines | Source | Verification result |
|---|---|---|
| 32 | [Toolkit at `7004bb1`](https://github.com/EdwinLovo/android-claude-toolkit/tree/7004bb1612da69538d5729ad2dc20c263a087b3e) | Catalog quotation at `CLAUDE.md:160`, scaffold padding at `:266`, serialization/converter imports at `:317/:321`, and network checklist at `:412` match. An in-memory scan of 43 text files found zero occurrences of `@Immutable`, `@Stable`, `ImmutableList` and `rememberUpdatedState`. The only matching padding call was `Modifier.systemBarsPadding()` in the scaffold. |
| 44 | [Strong skipping](https://developer.android.com/develop/ui/compose/performance/stability/strongskipping) | The Kotlin 2.0.20 default, unstable-parameter `===` comparison, collection warning and “Only annotate classes with cheap…” passages match. The ellipsis omits lazy-layout examples. The source warning is not proof that every comparison traverses every element. |
| 45 | [Fix stability issues](https://developer.android.com/develop/ui/compose/performance/stability/fix) | All three quotations match, including the collection-reference/benchmark exception. Complete-document checking confirmed the sentence that an earlier partial retrieval had missed. |
| 46, 63 | [Performance skill at `95ea931`](https://github.com/android/skills/blob/95ea931a1ad8f9b2be132ae0cfb2aad964d71118/jetpack-compose/jetpack-compose-performance/SKILL.md) and [at `ed0a272`](https://github.com/android/skills/blob/ed0a2728ff531174383f910c649694de1eaebcd6/jetpack-compose/jetpack-compose-performance/SKILL.md) | The two prohibitions, two existing-annotation fragments and long-running-coroutine `rememberUpdatedState` quotation match in both snapshots. Omitted context: the upstream existing-usage guidance allows removal for models that both feed lazy layouts and contain large datasets or deeply nested collections. “Leave existing annotations alone” is not the entire upstream policy. |
| 47 | [AbstractList](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/collections/AbstractList.kt), [Kotlin specification](https://kotlinlang.org/spec/declarations.html), [Pair 2.2.21 source](https://github.com/JetBrains/kotlin/blob/v2.2.21/libraries/stdlib/src/kotlin/util/Tuples.kt) | `if (other === this) return true` and the spec's “are not guaranteed to exactly match…” qualification match. `Pair` is declared as a `data class` in the versioned source. The bytecode observation is not a universal compiler guarantee. |
| 48 | [Compose-lints rules](https://slackhq.github.io/compose-lints/rules/) | Immutable-collection guidance and `ComposeUnstableCollections` being disabled by default are confirmed. A project can explicitly enable it; default-off does not prove there can never be a project-specific tooling conflict. |
| 62 | [Compose side effects](https://developer.android.com/develop/ui/compose/side-effects) | Both quotations match. The first is an excerpt. The second also appears in the complete document; a partial retrieval had incorrectly reported it absent. |
| 64 | [Lifecycle in Compose](https://developer.android.com/topic/libraries/architecture/compose) | Both sentences about `LifecycleStartEffect` keys match, with the introductory comparison to other effects omitted. Passing a callback as a key is an application of that behavior, not an additional source quotation. |
| 79–80 | [Edge-to-edge at `95ea931`](https://github.com/android/skills/blob/95ea931a1ad8f9b2be132ae0cfb2aad964d71118/system/edge-to-edge/SKILL.md) and [at `ed0a272`](https://github.com/android/skills/blob/ed0a2728ff531174383f910c649694de1eaebcd6/system/edge-to-edge/SKILL.md) | All eight quotations match in both snapshots: `adjustResize` at :34; adaptive scaffold guidance at :106; IME ordering/duplication at :112; default insets excluding IME at :160; list-parent padding at :341. No `systemBarsPadding` or `NavHost` occurs. Absence of the literal phrase “root padding” is not semantic proof about every possible root-padding arrangement. |
| 81 | [Insets UI](https://developer.android.com/develop/ui/compose/system/insets-ui) | The root `Box(Modifier.safeDrawingPadding())` example and quoted trade-off match. They do not support an unconditional prohibition of root padding. |
| 94 | [Play photo/video policy](https://support.google.com/googleplay/android-developer/answer/14115180) | Both quotations, the custom-picker declaration explanation and 2025-05-28 compliance date match. The URL now redirects to [answer 16935362](https://support.google.com/googleplay/android-developer/answer/16935362?hl=en). This passage concerns `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO`, not every `READ_MEDIA_*` permission. The policy also explicitly exempts certain private and enterprise device-management apps. |
| 95 | [Android 14 partial access](https://developer.android.com/about/versions/14/changes/partial-photo-video-access) | The quotation with ellipsis matches. The context says the change is enabled for apps targeting API 34+. This does not require Photo Picker clients to request `READ_MEDIA_VISUAL_USER_SELECTED`. |
| 96 | [Permissions skill](https://github.com/android/skills/blob/95ea931a1ad8f9b2be132ae0cfb2aad964d71118/security/android-permissions-security/SKILL.md#L430) | “Prefer `ActivityResultContracts.PickVisualMedia()`…” matches; the recommendation names no exceptions. |
| 109 | [Retrofit 2.10.0 changelog](https://github.com/square/retrofit/blob/2.10.0/CHANGELOG.md), [2.9.0 rules](https://github.com/square/retrofit/blob/2.9.0/retrofit/src/main/resources/META-INF/proguard/retrofit2.pro), [2.10.0 rules](https://github.com/square/retrofit/blob/2.10.0/retrofit/src/main/resources/META-INF/proguard/retrofit2.pro) | The cited commit dates/messages, changelog quotation and R8 full-mode comment match. The release sequence goes from 2.9.0 to 2.10.0. **2.9.0 already shipped consumer rules**, but lacked the identified later additions; it was not a release without rules. |
| 110 | [kotlinx.serialization README](https://github.com/Kotlin/kotlinx.serialization/blob/master/README.md) | Both quotations match: rules are supplied by default, with the named-companion limitation. |
| 111 | [Gson troubleshooting](https://github.com/google/gson/blob/main/Troubleshooting.md) | “included since Gson 2.11.0” matches. Its context discusses problems that can remain even with those rules; it does not establish complete coverage. |
| 112 | [R8 FAQ](https://r8.googlesource.com/r8/+/refs/heads/main/compatibility-faq.md) | The `<init>()` quotation matches, under R8 full mode. |
| 113, 142 | [Library optimization](https://developer.android.com/topic/performance/app-optimization/library-optimization) | Both the automatic-consumption quotation and Gson reflection warning match. JAR `META-INF/proguard/`, legacy AAR `proguard.txt`, and targeted locations are documented. **“It lists no libraries” is literally false:** the page names Gson, WorkManager and Room. It does not establish the proposed blanket inventory of libraries needing no app-side rules. |
| 127 | [Version catalog migration](https://developer.android.com/build/migrate-to-catalogs) | The simultaneous script/catalog quotation matches. **“we recommend” is taken out of context:** it recommends the default filename `libs.versions.toml`, not catalog adoption. The conclusion that the page establishes no catalog requirement remains valid. |
| 141 | [Gson README](https://github.com/google/gson/blob/main/README.md) | All three quotations match: maintenance mode, Kotlin limitations and preference for explicit language support. They do not prescribe nullable types for every DTO field. |
| 154 | [Material 3 releases](https://developer.android.com/jetpack/androidx/releases/compose-material3) | Stable 1.4.0, beta 1.5.0-beta01 and 2026-10-07 match. Relevant graduations occur in separate 1.5.0 alphas: expressive buttons/menus/FABs in alpha19; split buttons in alpha20; floating toolbars in alpha22; list items and enumerated TopAppBar APIs in alpha23. These concern specific APIs, not guaranteed stability of every API in each family. |

### Source-version distinctions

- `android/skills` current snapshot checked: `95ea931a1ad8f9b2be132ae0cfb2aad964d71118`.
- Historical snapshot named in the local spec: `ed0a2728ff531174383f910c649694de1eaebcd6`.
- Toolkit checked: **`EdwinLovo/android-claude-toolkit`**, `7004bb1612da69538d5729ad2dc20c263a087b3e`. This is a different repository from `android/skills`.
- Current documentation and `main`/`master` URLs can change. The historical comparisons above apply only to the explicitly checked snapshots/artifacts.

## Local references and provenance

| Proposal | Current mapping | Additional checked evidence |
|---|---|---|
| 1 | `android/SKILL.md:64` ↔ `specs/android-guidelines.md:99` | Gap analysis :269–271; `android/references/compose-ui.md:16`. The hard restriction entered the spec in `088975e`. |
| 2 | SKILL :62 ↔ spec :98 | Gap analysis :262 contains “long-lived”; :741 drops it. |
| 3 | SKILL :83/:85/:86 ↔ spec :126/:127/:128 | Gap analysis :23/:628/:654 supports the scaffold inference. The latter two spec rules were added in `088975e`. |
| 4 | SKILL :98 ↔ spec :137 | Photo Picker rule added to the spec in `088975e`. |
| 5 | SKILL :104–105 ↔ spec :142–143 | Both current files contain the `Retrofit 2.9+` statement. |
| 6 | SKILL :21 ↔ spec :21 | Gap analysis :97. At `9864981`, the spec rule was at **:19**, not :21. |
| 7 | SKILL :42 ↔ spec :80 | At `9864981`, the spec rule was at **:75**, not :80. `data-and-networking.md:5` and SKILL :47 support respecting established conventions, but networking and DI rules are not themselves serializer rules. |
| Optional line 66 | SKILL :66 ↔ spec **:101** | Spec :113 is a related default, not the exact counterpart. |

### Corrections to the review's local citations

- **Review :74:** “spec lines 90–91” are numbered lines of `git show` output, not lines in the spec. The actual file lines are **127–128**.
- **Commit attribution:** `android/SKILL.md` was introduced entirely in **`9ff248e`**. `088975e` and `9864981` establish spec provenance, not introduction of lines into SKILL.md. The latter file did not yet exist in those commits.
- **Review :30:** the provenance block ends at **spec :196**. The file has 196 lines; :197 does not exist.
- **Review :76:** criterion 6 at spec :174 requires retaining specified `Verify:` hints; it does **not** prohibit ownership guidance or require categorical prohibitions.
- **Review :153:** criterion 4 at spec :172 requires traceability, not identical wording. The proposed semantic/strength changes should still be reflected in the spec. Its affected-line list also needs **:101** if the optional line-66 edit is accepted.

SKILL :9–10 confirms that the override clause covers defaults, and :114 contains the no-suppression rule. The `CLAUDE.md` quotation at review :32 refers to the external toolkit, not this repository's local `CLAUDE.md`.

## Reproduction results

All commands in the review's Reproduction section were executed. They exited 0 without stderr. The supplemental inspections below are identified separately.

### Retrofit dates: no discrepancy

| Object | Review date | Reproduced date |
|---|---|---|
| `51e81a8` | 2021-06-22 | 2021-06-22 |
| `007a706` | 2021-08-07 | 2021-08-07 |
| `59d302a` | 2023-05-05 | 2023-05-05 |
| `ef8d867` | 2023-08-15 | 2023-08-15 |
| 2.10.0 publication | 2024-03-18 | 2024-03-18 |
| 2.9.0 tagger | 2020-05-20 | 2020-05-20 |

Author and committer dates also agree for the four identified commits. The 2.9.0 tag object is `56c2a0ac04f87a36689aca31f9a06018358c8d37`.

### javap: different selected JAR, same identity shortcut

The exact `find ... | head -1` recipe selected:

```text
~/.gradle/wrapper/dists/gradle-8.14.3-bin/cv11ve7ro1n3o1j4so8xd9n66/gradle-8.14.3/lib/kotlin-stdlib-2.0.21.jar
```

It did **not** select Kotlin 2.2.21 bundled with Gradle 9.3.1. The recipe does not pin the version; its result depends on discovery order.

Exact printed output:

```text
  public boolean equals(java.lang.Object);
    Code:
       0: aload_0
       1: aload_1
       2: if_acmpne     7
       5: iconst_1
       6: ireturn
```

The `sed` range stops at the **first `ireturn`**. This output confirms the identity shortcut but does not show the field comparisons. Supplemental disassembly of the full method, explicitly selecting 2.2.21 as well as 2.0.21, produced the same method instructions: identity first, then field comparisons.

### Local provenance and toolkit commands

- The `git show ... | grep -n` command reproduced **90/91 as patch-stream line numbers**, explaining the erroneous spec citation.
- `git log -S'No Gson in new code'` returned `9864981 docs(specs): add Android guidelines spec`.
- The `sed` command reproduced gap-analysis lines 262, 271 and 741, including the loss of “long-lived”.
- The toolkit command reproduced `CLAUDE.md:160` and scaffold :266/:317/:321/:412, plus the other serialization/catalog references selected by its pattern.

## Final proposed edit list

These are proposals, not applied edits. All locations use baseline line numbers. Mirror the revised decisions in the matching spec entries; retain the Must/Default distinction rather than putting new-project defaults inside the universal must section.

### A. Technical corrections

#### Proposal 1 — android/SKILL.md:64

**Matching spec:** `specs/android-guidelines.md:99`.

Current:

```markdown
- Never add `@Immutable`, `@Stable` or `ImmutableList` to models that hold collections: strong skipping (default since Kotlin 2.0.20) already skips on instance equality, and the annotations force an O(n) `equals` on every recomposition. Classes without collection fields (e.g. theme tokens) may keep `@Immutable`.
```

Replacement:

```markdown
- With strong skipping (default since Kotlin 2.0.20), don't add `@Immutable`, `@Stable` or `ImmutableList` to collection-bearing models as an automatic performance fix; preserve collection instances instead. Making an otherwise unstable parameter stable switches instance comparison to `equals`, whose collection comparisons can cost O(n), not on every recomposition. Make an exception only when the stability contract is valid, the collection reference cannot be preserved and benchmarks show structural comparison beats recomposition. Don't change existing annotations or collection types merely to comply with this rule. Collection-free classes (e.g. theme tokens) may keep valid `@Immutable`.
```

#### Proposal 2 — android/SKILL.md:62

**Matching spec:** `specs/android-guidelines.md:98`.

Current:

```markdown
- Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. Effects that call lambda params use `rememberUpdatedState`.
```

Replacement:

```markdown
- Lifecycle reactions use `LifecycleResumeEffect`/`LifecycleStartEffect`/`LifecycleEventEffect`. For keyed effects, read a lambda parameter through `rememberUpdatedState` when changing it must not restart the effect; use the lambda as a key when changing it should restart the effect.
```

#### Proposal 3 — android/SKILL.md:83

**Matching spec:** `specs/android-guidelines.md:126`.

Current:

```markdown
- Enable edge-to-edge (`enableEdgeToEdge()`) and apply insets once; never `systemBarsPadding()` on a NavHost or root.
```

Replacement:

```markdown
- Enable edge-to-edge (`enableEdgeToEdge()`) and apply each inset once. When screens should draw behind system bars, don't apply system-bar padding to the whole NavHost or root; apply the required insets per screen or component. Root padding is valid when keeping all content within safe bounds is intentional.
```

Keep SKILL lines **84–86** and spec lines **127–128** unchanged.

#### Proposal 4 — android/SKILL.md:98

**Matching spec:** `specs/android-guidelines.md:137`.

Current:

```markdown
- Media selection uses the Photo Picker (`PickVisualMedia`), not `READ_MEDIA_*`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```

Replacement:

```markdown
- Use the Photo Picker (`PickVisualMedia`) for user-selected photos and videos. Request `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO` only when broad library access is essential to core functionality and system pickers are insufficient; satisfy applicable Play declaration and review requirements. For a permission-based gallery on Android 14+ targeting API 34+, handle partial access and reselection with `READ_MEDIA_VISUAL_USER_SELECTED`. No `QUERY_ALL_PACKAGES` (declare `<queries>`), no `MANAGE_EXTERNAL_STORAGE`, `USE_EXACT_ALARM` only for alarm or calendar apps, and every foreground service declares a `foregroundServiceType`.
```

Keep the line-99 grep as a review prompt, not a zero-match requirement.

#### Proposal 5 — android/SKILL.md:105

**Matching spec:** `specs/android-guidelines.md:143`.

Current:

```markdown
- Keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. No rules for libraries that ship their own (AndroidX, kotlinx, coroutines, Retrofit 2.9+) or for Activity and View classes. Under full mode `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.
```

Replacement:

```markdown
- Keep rules are narrow and class-specific. Never `-keep class pkg.** { *; }`, `!` inversions, `-dontoptimize`/`-dontobfuscate`/`-dontshrink` or `android.enableR8.fullMode=false`. Don't duplicate consumer rules bundled by the exact artifact/version: inspect its JAR or AAR (including locations such as `META-INF/proguard/` or `proguard.txt`) and add only missing, verified narrow rules. No rules for Activity and View classes. Under full mode `-keep class A` doesn't keep the no-arg constructor; write `{ <init>(); }`.
```

Keep SKILL **104** and spec **142** unchanged: they already require verification and a tested minified release. Removing the list corrects `Retrofit 2.9+` without substituting another universal library/version whitelist. The existing Activity/View and other prohibitions are preserved, not independently re-audited here.

### B. Reclassifications

#### Proposal 6 — android/SKILL.md:21–22

**Matching spec:** `specs/android-guidelines.md:21`.

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

At spec :21, separate that must from the catalog default. The existing reference file identified by the review needs the corresponding reclassification; this is not a new topic or new reference file.

**android/references/build-and-modules.md:3 — current:**

```markdown
The SKILL.md musts (ask before changing toolchain or SDK levels, version catalog, new code in the implied module, no module without asking) apply; this file adds the how.
```

**Replacement:**

```markdown
The SKILL.md musts (ask before changing toolchain or SDK levels, follow the project's dependency mechanism, new code in the implied module, no module without asking) apply; this file adds the how.
```

**Default relocated into that existing reference after baseline line 8**, with its decision recorded at spec :21:

```markdown
- New projects use a version catalog unless the project or template already establishes another dependency-management mechanism.
```

#### Proposal 7 — android/SKILL.md:42

**Matching spec:** `specs/android-guidelines.md:80`.

Current:

```markdown
- DTOs are `@Serializable` data-layer types mapped to domain models, never in domain or UI signatures. No Gson in new code.
```

Replacement:

```markdown
- DTOs are data-layer types mapped to domain models, never in domain or UI signatures. Use the project's serializer; don't introduce a second serializer or migrate without asking.
```

Make the existing serializer default explicitly defer to the project in the existing reference:

**android/references/data-and-networking.md:5 — current:**

```markdown
- Use Retrofit 3 + OkHttp + the kotlinx.serialization converter, unless the project already uses another networking library.
```

**Replacement:**

```markdown
- Use Retrofit 3 + OkHttp unless the project already uses another networking library. Use kotlinx.serialization for DTOs and its converter when using Retrofit, unless the project already uses another serializer.
```

**Matching default spec: :76.** Preserve its Apollo/Ktor text and change its final sentence to: `Existing projects keep their networking library and serializer.`

### C. Optional items

#### Flagged omission — android/SKILL.md:66

**Matching spec:** `specs/android-guidelines.md:101`, not :113.

Current:

```markdown
- No experimental or alpha Compose APIs (Material 3 Expressive, Styles, Grid/FlexBox, `mediaQuery`) and no new `@OptIn` or module-wide opt-in flag without asking. Check the project's Compose version before using an API that needs a newer one.
```

Replacement:

```markdown
- No experimental Compose APIs requiring an opt-in, alpha Compose APIs, new `@OptIn` annotations or module-wide opt-in flags without asking. Check the project's Compose version before using an API that needs a newer one.
```

This removes aging component labels. It does not independently reclassify or research Styles, Grid/FlexBox or `mediaQuery`.

#### Optional Gson nullability note: do not add

Do not add a universal requirement that every Gson DTO field be nullable. The verified limitations justify careful modeling and validation, but the cited sources do not prescribe that blanket rule. Existing DTO mapping remains in scope; an additional nullability policy is unnecessary here.

## Scope and execution record

The proposed implementation would touch only the existing `android/SKILL.md`, `specs/android-guidelines.md`, `android/references/build-and-modules.md` and `android/references/data-and-networking.md`. The last two changes are the default-placement/consistency consequences of proposals 6 and 7, not new subjects. This report is the only file created in response to the request to persist the verdict.

At the end of verification, tracked repository files were unchanged. The handoff and review remained untracked as they were at the start. Execution incident disclosed in the original response: an exploration agent created and immediately removed an empty temporary stderr file outside the repository, despite the strict no-file-creation instruction. No temporary file remained.
