# Build and modules

The SKILL.md musts (ask before changing toolchain or SDK levels, follow the project's dependency mechanism, new code in the implied module, no module without asking) apply; this file adds the how.

## Toolchain and SDK
- New projects use minSdk 26, unless the project already sets another minSdk. Why: `java.time` without desugaring and no compat noise.
- New projects set compileSdk and targetSdk to the latest stable API level, unless the project already sets them. Check the latest stable API level before setting it.
- New projects use the latest stable Kotlin and AGP, unless the project already pins other versions.
- New projects use a version catalog, unless the project or template already establishes another dependency-management mechanism.

## Modules
- New projects start as a single `:app` module, unless the project already has a module structure.
- Growth path, unless the project already has another split:
  - `:core`: domain models, repository interfaces, use cases, design system, utilities. No network or database dependencies.
  - `:data`: network client, DTOs, mappers, repository implementations, local storage, data-layer Hilt bindings. Depends on `:core`.
  - `:app`: UI layer (screens, ViewModels, navigation, Application). Depends on both.
  - Propose the split only when asked or when a second module is needed anyway (widget, Wear, shared SDK).
- Convention plugins in `build-logic/` arrive with the split, not before, unless the project already has them. They use only public `com.android.build.api.*` types and `androidComponents.onVariants`/`beforeVariants`; `applicationVariants` and `variantFilter` are gone in AGP 9.
