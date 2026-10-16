# Spec: iOS guidelines skill (agent-agnostic)

**Date:** 2026-10-15 · **Status:** Settled (grilling session)
**Context:** A portable Agent Skill that tells coding agents (Claude Code, Codex, OpenCode, Pi) how to write and review iOS code in any project Jaim works on, including client projects. It is derived from the gap analysis of uguizu/ios-project-generator (`DotConfigs/docs/ios-guidelines/research/2026-10-ios-generator-gap-analysis.md`), used as inspiration only: no code or text is copied from that GPL-3.0 repo.

## Decisions

### Scope and strength
- **Two strengths of rule.** *Must* rules apply to every project. *Default* rules apply only when the project has no established convention for that area. Why: client projects can't adopt opinionated choices, but platform rules apply everywhere.
- **Project conventions win over defaults.** Every default rule says so in its own text, because when a user-level rule and a project rule conflict, agents may follow either one (Claude Code memory docs).
- **Infra is follow-only.** CI, signing and fastlane: the agent follows whatever the project already has and never introduces or replaces it. Why: an agent writing a feature doesn't choose a CI platform.

### Delivery
- **Format:** one Agent Skill (`SKILL.md` plus reference files), the only format all four tools read. Why: path-scoped `.claude/rules` and `@imports` are Claude-only.
- **Source:** `~/Developer/DotConfigs/rules/ios/`, in a new git repo `DotConfigs/rules` that will also hold `android/` and `react-native/`. Why: the rules are personal and used across companies and clients.
- **No global install (revised 2026-10-15, after slice 1).** The rules are added to a project manually, or by asking an agent to copy them, only when needed. Why: Jaim chose per-project opt-in over symlinks and trigger lines in every tool.
- **Per project:** copy `ios/` into the project's skills folder, or paste `SKILL.md` into its `AGENTS.md`/`CLAUDE.md`, committed or local-only per client. Project-specific overrides go next to it.

### Language and concurrency
- **Default:** Swift 6 language mode, `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor`, `SWIFT_APPROACHABLE_CONCURRENCY = YES`. Why: the compiler enforces the concurrency rules, and MainActor by default keeps the noise low. Xcode 27's template still starts in Swift 5 mode, so this must be set explicitly.
- **Must:** never change an existing project's language mode or concurrency settings without asking.

### Minimum iOS
- **Default:** iOS 26 for new projects, with a documented floor of iOS 17 when a client needs wider reach. Why: the current-and-previous-major convention, and Liquid Glass plus current SwiftUI with no `#available` noise.
- **Not enforced** on existing projects.

### Architecture and state
- **Default:** MVVM with `@MainActor @Observable` view models. A view model is optional for screens that only display data.
- **Must:** no `ObservableObject` in new code, unless the project already uses it.

### Dependency injection
- **Default:** Factory 3. Services are registered as protocols. View models and services use `@Injected`, views never resolve services, and test suites use the `.container` trait. Why: compile-time safe, Swift Testing support, and consistency with MyWalkup.
- **`@Entry` (environment)** only for values that depend on the view-tree position, such as a per-tab router, and only when that need comes up.
- **Audit:** any DI is accepted, but a view that resolves services directly is flagged.

### Navigation
- **Default:** `NavigationStack` with a path-based router, one router per tab, passed through `@Entry`.

### Networking
- **Default:** `URLSession` with async/await, no networking dependency. Alamofire is the approved exception when a project needs interceptors, retry or pinning. No Moya in new projects (last release 2022-08).
- **Must:** request and response models are `Sendable` `Codable` types, never `[String: Any]`.
- **Existing projects** keep their library.

### Project structure
- **Default:** plain Xcode with buildable folders, no XcodeGen or Tuist.
- **Default:** an app target plus one local `Core` SPM package (networking, models, design system, utilities). Grow into feature packages only when a feature gets big or is shared.
- **Must:** if `project.yml` or `Project.swift` exists, edit that and regenerate; never hand-edit `project.pbxproj`.
- **Must:** put new code in the module the project's structure implies; never create a package or target without asking.

### Environments and configuration
- **Default:** one app target, with build configurations plus schemes for Dev, QA and Prod, an xcconfig per configuration, a bundle-ID suffix per environment, and an app icon per configuration through `ASSETCATALOG_COMPILER_APPICON_NAME`.
- **Must:** no secrets in xcconfig files or source code. Base URLs are fine.
- **Must:** required config is read once at launch and fails fast with a clear message, never through a `fatalError` inside a lazily evaluated property.

### Design system and UI
- **No default design system.** The project's token scheme comes from the client's Figma.
- **Must:** named colors from the asset catalog only, never hard-coded hex values in views. Brand colors have increased-contrast variants.
- **Default:** SwiftUI first. UIKit only behind `UIViewRepresentable`/`UIViewControllerRepresentable` for something SwiftUI can't do, with a one-line "why" comment. No storyboards or XIBs.
- **Must:** follow an existing project's UI framework, and don't rewrite UIKit screens into SwiftUI without asking.

### Platform musts
- **Privacy manifest:** `PrivacyInfo.xcprivacy` exists. When code uses a required-reason API, the matching reason code is added in the same change.
- **Localization:** user-facing strings go through a String Catalog (`String(localized:)` or `LocalizedStringKey`), with no sentence concatenation.
- **Accessibility:**
  - Dynamic Type through text styles, with no fixed font sizes.
  - A label on every icon-only control.
  - Tap targets of at least 44×44 pt.
  - No information conveyed by color alone.
- **Safety:** no force unwraps or `try!` outside tests.

### Testing
- **Default:** Swift Testing for new unit tests. In an XCTest project, new tests follow the project's framework.
- **Default:** XCTest for UI and performance tests, including one `performAccessibilityAudit()` UI test per main screen.
- **Default:** snapshot tests off, allowed when a project opts in.
- **Scope:** tests cover view models and services, with fakes injected through Factory's `.container` trait. Views are not unit-tested.

### Tooling
- **Default:** SwiftFormat plus SwiftLint, run in pre-commit and CI, not as an Xcode build phase. Why: user script sandboxing, and build time.
- **Must:** before finishing, run the project's formatter and linter and fix what they report. Never disable a rule to get a clean run.

### Logging and crash reporting
- **Must:** `os.Logger` per subsystem and category, with private data marked `privacy: .private`. No `print` in committed code.
- **No crash-reporting default.** Follow the project's SDK, and keep its privacy-manifest entries current.

## Acceptance criteria
1. `DotConfigs/rules` is a git repo, and `rules/ios/SKILL.md` has valid Agent Skill frontmatter (`name: ios-guidelines` and a `description` that names Swift/iOS work).
2. `SKILL.md` lists every **must** rule above. It states the must/default distinction and the "project conventions win" clause, and indexes the reference files. It stays under 150 lines.
3. Every **default** rule lives in a reference file, and each one's text includes "unless the project already uses X" or equivalent.
4. Traceability: every decision in this spec maps to at least one rule. Check with a table or grep in the slice-1 PR description.
5. No text or code is copied from uguizu/ios-project-generator, and no client name, scheme or target appears in the skill.
6. `rules/install.sh` creates the 4 symlinks. `install.sh --check` exits 0 when all links point to `rules/ios` and non-zero, naming the problem, otherwise.
7. The trigger line is present in `dotclaude/CLAUDE.md`, `dotopencode/AGENTS.md`, `~/.codex/AGENTS.md` and Pi's global `AGENTS.md`.
8. Load check: in a personal iOS project (not a client repo), each of the 4 tools, asked "add a settings screen", loads `ios-guidelines` before writing Swift. The evidence is a transcript excerpt or session log per tool.
9. The README documents install, the override line format for projects, and how to export the rules for a client.

## Slices
1. **Skill content.**
   - Scope: `git init` `DotConfigs/rules`, then write `ios/SKILL.md` and 7 reference files: concurrency, architecture-and-di, networking, testing, tooling-and-project, environments, ui (design system, accessibility, localization, UIKit), logging-and-privacy.
   - Estimate: about 9 files and 500 lines.
   - Depends on: nothing.
   - Covers criteria 1–5.
2. **Install and wiring.** Dropped on 2026-10-15, before any work: rules are added per project instead. Criteria 6–9 no longer apply.

## Non-goals / Deferred
- Global install, trigger lines and the per-tool load check (the former slice 2).
- Changing or forking uguizu/ios-project-generator, and the GPL question that comes with it.
- A new-project generator or template. These rules may later feed one.
- Android and React Native rules: same repo, separate specs.
- Choosing CI, signing, fastlane or crash-reporting tools.
- Migrating existing client projects toward the defaults.
- Unverified research claims, which recheck before relying on them: the Liquid Glass opt-out being ignored on the iOS 27 SDK, Xcode 27 not debugging iOS 15/16, and the iOS 18 adoption figure.
