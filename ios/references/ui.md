# UI: design system, accessibility, localization, UIKit

The SKILL.md musts (colors, UI framework, accessibility, localization) apply; this file adds the how.

## Design system and framework
- Take design tokens (colors, type, spacing) from the client's Figma, unless the project already has a token scheme. No default design system exists.
- Build UI in SwiftUI first, unless the project already uses UIKit for its screens.
- Prefer SwiftUI. Drop to UIKit behind `UIViewRepresentable` or `UIViewControllerRepresentable` where UIKit does the job clearly better (finer control, performance, a more mature API), with a one-line "why" comment, unless the project already has a convention for this.
- Write layouts in code, with no storyboards or XIBs, unless the project already uses them.

```swift
struct MapView: UIViewRepresentable {
    // MKMapView: custom overlay renderers give finer control than SwiftUI Map.
    func makeUIView(context: Context) -> MKMapView { MKMapView() }
    func updateUIView(_ view: MKMapView, context: Context) {}
}
```

## Colors
- Define each color as a named asset-catalog color set, with a High Contrast variant for brand colors, unless the project already has a color system.
- Reference colors through the generated symbols (`Color.brandPrimary`), unless the project already wraps them.

## Accessibility
- Use text styles (`.font(.body)`); scale custom metrics with `@ScaledMetric`, unless the project already has a type scale.
- Add `.accessibilityLabel` to icon-only buttons and `.accessibilityHidden(true)` to decorative images.
- Apply `.frame(minWidth: 44, minHeight: 44)` plus `.contentShape(Rectangle())` to small controls.

## Localization
- Keep strings in `Localizable.xcstrings`; use `Text("key")` or `String(localized:)`, unless the project already has a localization setup.
- Use format arguments and plural variations for dynamic text, e.g. `"\(count) items"`, with the sentence kept whole.
