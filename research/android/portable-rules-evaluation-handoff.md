# Handoff: Evaluate a Portable Android Ruleset

## Objective

Evaluate the proposed approach and recommendations for curating an existing Android coding ruleset against external sources.

The user wants **portable coding recommendations that can be reused across projects**, not task-oriented skills, agents, commands, scaffolding, or toolkit workflows.

The central principle is:

> Recommend a default only when the project has no established convention for that area. Otherwise, respect the project's existing approach.

The files happen to use `SKILL.md` packaging. **Do not interpret that as a request to build additional skills.**

## Local Sources

Repository:

```text
/Users/jaimdelmarravn.co/Developer/DotConfigs/rules
```

Rules to evaluate:

```text
android/SKILL.md
android/references/architecture-and-state.md
android/references/build-and-modules.md
android/references/compose-ui.md
android/references/data-and-networking.md
android/references/di-and-concurrency.md
android/references/environments.md
android/references/navigation.md
android/references/testing.md
android/references/tooling-and-logging.md
```

Existing decision history:

```text
specs/android-guidelines.md
research/android/2026-10-android-toolkit-gap-analysis.md
```

Read the rules and spec directly. Treat previous research as leads, **not independently verified evidence**.

## External Sources

- https://github.com/android/skills — extract applicable technical recommendations, not its task workflows.
- https://github.com/EdwinLovo/android-claude-toolkit/tree/main/.claude — compare conventions without adopting the author's entire architecture.
- https://github.com/android — selectively inspect relevant repositories, not the entire organization.
- Useful samples: `architecture-samples`, `nowinandroid`, `compose-samples`, and `performance-samples`.
- Use current Android, AndroidX, Kotlin, and Gradle documentation to adjudicate technical claims. An official sample demonstrates an approach; it does not automatically establish a universal requirement.

## Proposed Curation Approach

Evaluate whether this approach is sound:

- Keep the existing ruleset as the baseline rather than replacing it with either upstream repository.
- Distinguish technical requirements, contextual recommendations, and personal/team preferences.
- Review each rule for applicability, correctness, project-convention compatibility, version assumptions, and verifiability.
- Maintain provenance and review dates separately from the concise rules agents load.
- Avoid adding topics or abstractions merely to achieve “complete Android coverage.”

Existing choices such as MVVM/UDF, `handleEvent`, Hilt, Retrofit, initial single-module structure, and JUnit 4 are recorded defaults. Do not reopen them merely because a sample uses something different. Technical errors or conflicts with portability should still be identified.

## Recommendations Requiring Independent Evaluation

These are **proposals, not approved edits or final conclusions**.

| Local location | Proposed adjustment |
|---|---|
| `android/SKILL.md:64` | Reconsider the blanket prohibition on `@Immutable`, `@Stable`, and immutable collections in collection-bearing models. The explanation overgeneralizes equality costs. Distinguish identity versus structural comparison, valid stability contracts, and measured performance. |
| `android/SKILL.md:62` | Narrow the requirement for `rememberUpdatedState`: use it when an effect should retain its lifetime while observing updated captured values, not automatically for every lambda. |
| `android/SKILL.md:83–86` | Replace overly categorical inset-location prohibitions with guidance about ownership, consumption, double application, and appropriate scrolling behavior. Verify whether existing restrictions have useful contextual justification. |
| `android/SKILL.md:98` | Prefer Photo Picker for user-selected media, while recognizing justified broad-library-access use cases. |
| `android/SKILL.md:104–105` | Verify consumer rules for the actual dependency artifact/version instead of categorically assuming named libraries never need additional rules. |
| `android/SKILL.md:21` | Consider version catalogs a default rather than requiring their introduction into projects using another dependency-management convention. |
| `android/SKILL.md:42` | Preserve the project's established serializer rather than universally requiring `@Serializable` and prohibiting new Gson usage within an existing Gson-based project. |

Useful verification starting points:

```text
https://developer.android.com/develop/ui/compose/performance/stability/strongskipping
https://developer.android.com/develop/ui/compose/performance/stability/fix
https://developer.android.com/develop/ui/compose/side-effects
https://github.com/android/skills/blob/main/system/edge-to-edge/SKILL.md
https://developer.android.com/training/data-storage/shared/media
https://developer.android.com/topic/performance/app-optimization/library-optimization
```

## Prior Findings and Limitations

- The toolkit includes choices such as custom `reduce`, `ViewModelNav`, a global error bus, and restrictive theme conventions. Their existence does not justify importing them.
- Official samples inspected demonstrated model mapping and tests using fakes, not universal requirements for those implementations.
- The existing `PendingIntent` rule already accommodates mutable intents with an explicit component; do not propose a redundant correction.
- Previous review covered all ten local files, but **did not independently verify every rule, API, or version claim**.
- No files were modified during the comparison. Installation and agent integration are outside this evaluation.

## Requested Output

- Assess whether the proposed approach matches the user's objective.
- For each recommendation: **agree, disagree, or needs qualification**, with exact local references and supporting evidence.
- Identify important omissions or incorrect conclusions without expanding scope unnecessarily.
- Propose the smallest justified set of changes, separating technical corrections from optional preferences.
- Do not edit files or implement anything.
