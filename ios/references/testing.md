# Testing

- Write every new unit test with Swift Testing (`@Test`, `#expect`), even when the project already has XCTest unit tests. Both frameworks run in the same target, so leave existing XCTest tests alone unless asked to migrate them.
- Use XCTest only for UI and performance tests; Apple keeps those APIs XCTest-only. Use another UI test framework if the project already does.
- Add one `performAccessibilityAudit()` UI test per main screen, unless the project already has an accessibility testing convention.
- Keep snapshot tests off, unless the project already opts in to snapshot testing.
- Test view models and services, unless the project already has a different test scope.
- Inject fakes through Factory's `.container` trait, unless the project already uses another DI approach.
- Leave views out of unit tests, unless the project already unit-tests views. Cover view behavior through the view model and the UI audit.

```swift
import FactoryTesting

@Suite(.container) @MainActor
struct ProfileViewModelTests {
    @Test func showsErrorWhenLoadFails() async {
        Container.shared.profileService.register { FailingProfileService() }
        let viewModel = ProfileViewModel()
        await viewModel.load()
        #expect(viewModel.errorMessage != nil)
    }
}
```

```swift
func testAccessibilityAudit() throws {
    let app = XCUIApplication()
    app.launch()
    try app.performAccessibilityAudit()
}
```
