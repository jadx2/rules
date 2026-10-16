---
name: ios-guidelines
description: Swift/iOS coding rules. Load before writing, changing or reviewing Swift or iOS code: SwiftUI or UIKit screens, view models, services, networking, tests, Xcode project or package changes.
---

# iOS guidelines

## How to apply
- **Must** rules apply in every project.
- **Default** rules apply only where the project has no established convention for that area. When a project convention (or its AGENTS.md/CLAUDE.md) conflicts with a default, the project wins.
- Infra (CI, signing, fastlane, crash-reporting SDK) is follow-only: use what exists, never introduce or replace it.

## Must rules

### Project settings
- Ask before changing an existing project's Swift language mode or concurrency settings.
- Minimum iOS is not enforced on existing projects; keep the project's deployment target.

### Architecture and state
- In new code use `@Observable`, unless the project already uses `ObservableObject`.
- DI audit: any DI approach is accepted. Flag a view that resolves services directly, and route resolution through its view model.

### Networking
- Request and response models are `Sendable` `Codable` types, never `[String: Any]`.
- Existing projects keep their networking library.

### Project structure
- If `project.yml` or `Project.swift` exists, edit it and regenerate instead of hand-editing `project.pbxproj`.
- Put new code in the module the project's structure implies. Ask before creating a package or target.

### Configuration
- Keep secrets out of xcconfig files and source. Base URLs are fine.
- Read required config once at launch and fail fast with a clear message. Never use `fatalError` inside a lazily evaluated property for this.

### UI
- Use named asset-catalog colors only, with no hard-coded hex in views. Give brand colors increased-contrast variants.
- Follow the existing project's UI framework. Ask before rewriting UIKit screens into SwiftUI.

### Platform: privacy, localization, accessibility, safety
- Keep `PrivacyInfo.xcprivacy` in the project. When code uses a required-reason API, add the matching reason code in the same change.
- Route user-facing strings through a String Catalog (`String(localized:)` / `LocalizedStringKey`). Build sentences as whole localized strings, never by concatenation.
- Accessibility:
  - Dynamic Type via text styles, with no fixed font sizes.
  - A label on every icon-only control.
  - Tap targets of at least 44x44 pt.
  - Pair color with text, shape or icon so no information rides on color alone.
- Force unwraps and `try!` appear only in tests.

### Tooling
- Before finishing, run the project's formatter and linter and fix what they report. Fix the code to satisfy a rule; leave the rule enabled.

### Logging
- Use `os.Logger` per subsystem and category, with private data marked `privacy: .private`. Use the logger in committed code in place of `print`.
- Crash reporting: follow the project's SDK and keep its privacy-manifest entries current.

## Defaults (reference files)
- [references/concurrency.md](references/concurrency.md): open for new projects or when setting language mode, actor isolation or minimum iOS.
- [references/architecture-and-di.md](references/architecture-and-di.md): open before adding a screen, view model, service or navigation route.
- [references/networking.md](references/networking.md): open before adding or changing API calls or choosing a networking library.
- [references/testing.md](references/testing.md): open before writing or changing tests, fakes or UI audits.
- [references/tooling-and-project.md](references/tooling-and-project.md): open before changing project layout, packages, targets, formatting or linting.
- [references/environments.md](references/environments.md): open before touching build configurations, schemes, xcconfig, bundle IDs or app icons.
- [references/ui.md](references/ui.md): open before building UI: design tokens, colors, accessibility, localization, UIKit interop.
- [references/logging-and-privacy.md](references/logging-and-privacy.md): open before adding logging, using a required-reason API or editing the privacy manifest.
