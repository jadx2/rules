# Tooling and project structure

Musts in SKILL.md apply here: project.yml/Project.swift handling, module placement, and running the formatter and linter before finishing.

- Use SwiftFormat plus SwiftLint, unless the project already uses other formatting or linting tools.
- Run them in pre-commit and CI, not as an Xcode build phase, unless the project already runs them in a build phase. Why: user script sandboxing, and build time.
- Use plain Xcode with buildable folders, with no XcodeGen or Tuist, unless the project already uses one of them.
- New projects have an app target plus one local `Core` SPM package holding networking, models, design system and utilities, unless the project already has another structure.
- Grow into feature packages only when a feature gets big or is shared, unless the project already splits by feature.
- Add files inside buildable folders and let Xcode pick them up, unless the project already lists files explicitly.

```
App/            app target (buildable folder)
Packages/Core/  networking, models, design system, utilities
```
