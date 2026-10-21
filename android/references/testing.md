# Testing

The SKILL.md musts (test doubles in test source sets, `runTest`) apply; this file adds the how.

- Use JUnit 4 for new projects, local and instrumented. Why: Robolectric, `AndroidJUnitRunner` and the Compose, Hilt and AndroidX Test rules are JUnit 4 only, so JUnit 5 means two test styles, plus the vintage engine wherever those JUnit 4 tests share a JVM test task. A project already on JUnit 5 or 6 keeps it for pure-JVM tests.
- Write fakes by hand for types we own, unless the project already standardizes on a mocking library. Use MockK only for hard-to-fake third-party types.
- Use a main-dispatcher rule or extension, and Turbine for genuine streams, unless the project already has other test utilities. Assert `StateFlow` on the final `uiState.value` or latest item, never an intermediate emission (conflation).
- Name JVM tests (Robolectric included) with backtick sentences, and `androidTest` tests `subject_condition_expected`, unless the project already has a naming convention.
- Write one Compose UI test per main screen against `ScreenContent`, unless the project already has a UI-test setup. Run it in `test` on Robolectric unless it needs a device. Run `enableAccessibilityChecks()` only in instrumented tests on API 34+; it is unsupported on Robolectric.
  - Find nodes by semantics (text, role, content description); use `testTag` only when semantics can't single the node out. Screens with saved state get a `StateRestorationTester` check. `HiltAndroidRule` is `order = 0`.
- Keep screenshot tests off, unless the project already uses them. When a project opts in: Roborazzi (1.56.0+ on AGP 9), or Compose Preview Screenshot Testing (experimental; AGP 9.0+ for IDE support). Never upgrade AGP for it. On AGP 9, Paparazzi only from 2.0.0-alpha05.1.
