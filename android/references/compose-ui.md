# Compose UI

The SKILL.md musts (lifecycle-aware collection, modifier parameter, no ViewModel below the entry, theme tokens, no `Icons.*`) apply; this file adds the how.

## Structure
- Use a stateful entry composable plus a stateless `internal` `<Screen>ScreenContent`, with the ViewModel only at screen level, unless the project already has another screen structure.
- Small `private` helper composables may share a file; a composable used from two places gets its own file and becomes `internal`, unless the project already has a file convention.

## Previews
- Add previews for reusable components and screen content: `private`, at the bottom of the file, unless the project already has a preview convention.
- Use `@PreviewLightDark` plus `@PreviewFontScales` on screen content, unless the project already has a wrapper annotation.
- Use `PreviewParameterProvider` only for real state variants, in the same file unless the project already uses sibling files.

## Theme and design system
- There is no default design system: tokens come from the project's design (Figma).
- Extend MaterialTheme, unless the project already has a theme setup: map the palette onto the M3 `ColorScheme`, typography and shapes. Put extra tokens (spacing, extended colors) in a `staticCompositionLocalOf` of `@Immutable` classes behind an `AppTheme.*` accessor. `MaterialTheme.colorScheme` stays allowed. Use a fully custom theme object only when the design isn't Material-based.
- Dynamic color is a per-project decision: follow the project's choice. Use Material 3 Expressive only when the project is on material3 1.5+ and opts in (see the experimental-API must).

## Icons and language
- Use Material Symbols vector drawables referenced as `R.drawable.ic_*`, unless the project already has an icon registry object, which it keeps.
- Support per-app language through `generateLocaleConfig` in apps with more than one locale, unless the project already configures locales another way.
