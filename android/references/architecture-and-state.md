# Architecture and state

The SKILL.md musts (private mutable state, no Context in ViewModels, no Channel/SharedFlow events) apply; this file adds the how.

## Screen layout
- Package by feature, unless the project already has another layout. Per screen: `<Screen>Route.kt` (`@Serializable` `<Screen>Route` key), `<Screen>Contract.kt` (`<Screen>UiState` + `sealed interface <Screen>Event`), `<Screen>ViewModel.kt`, `<Screen>Screen.kt` (entry composable, `internal` `<Screen>ScreenContent`, `private` previews).
- Unless the project already has another layout: add `components/` only when the feature has its own components and `delegates/` only per the delegates rule, add no `contracts/` subfolder, and keep UI models in the UI layer, not in `domain/`.

## State
- Use MVVM with UDF and one immutable `<Screen>UiState` data class with defaults, unless the project already uses another architecture.
- Route events through `handleEvent(event)` over a `sealed interface <Screen>Event` with an exhaustive `when`, unless the project already handles events differently. `ScreenContent` takes a single `onEvent` lambda.
- Use explicit backing fields on Kotlin 2.4+, unless the project already uses another pattern. Below 2.4 use `private val _uiState` + `asStateFlow()`. Never add an experimental compiler flag or upgrade Kotlin for this without asking.
- Update with `MutableStateFlow.update { it.copy(...) }` and no custom `reduce` extension, unless the project already has one. Derive state with `stateIn(viewModelScope, WhileSubscribed(5_000), initial)`.
- Prefer `stateIn` or an idempotent load over async work in `init`, unless the project already loads another way.
- Use `SavedStateHandle` for small input that must survive process death, unless the project already restores state differently. Navigation arguments come from the route key (see navigation.md for Navigation 3).

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(private val repo: ProfileRepository) : ViewModel() {
    // Kotlin 2.4+ explicit backing field; below 2.4 use _uiState + asStateFlow().
    val uiState: StateFlow<ProfileUiState>
        field = MutableStateFlow(ProfileUiState())

    fun handleEvent(event: ProfileEvent) = when (event) {
        is ProfileEvent.NameChanged -> uiState.update { it.copy(name = event.name) }
        ProfileEvent.MessageShown -> uiState.update { it.copy(userMessage = null) }
    }
}
```

## Delegates
- Optional pattern, only for a stateful sub-flow (own loading/error/data state) used where a use case alone would duplicate state handling. Skip it where the project already has another pattern.
  - Inject the delegate into the ViewModel and never expose it to the UI. The ViewModel combines its state into `uiState`.
  - No `init(scope)`: expose `fun state(scope: CoroutineScope): StateFlow<X>`, or make the delegate a plain class the ViewModel creates.
  - Stateless shared logic goes into use cases.

## Use cases
- Add one only when logic is non-trivial or reused across ViewModels, unless the project already routes all logic through use cases. Use `operator fun invoke`, stateless and main-safe; it may call other use cases. Domain code has no Android framework or UI imports.

## Messages and errors in the UI
- Per-screen errors are state: `uiState.userMessage: UiText?`, shown by the screen and cleared with a `MessageShown` event, unless the project already has a message mechanism.
- App-wide messages (session expired, offline) go through one injected `@Singleton` holder exposing `StateFlow<List<UserMessage>>`, drained by the root composable, which shows each message and then marks it shown, unless the project already has one. No Channel, no global `object` bus.
- `UiText` (string resource with format args, or dynamic string) carries text produced outside composables, unless the project already has an equivalent.
