# Navigation

The SKILL.md musts (`@Serializable` routes, callbacks instead of `NavController`, Navigation 3 entry decorators, back handling) apply; this file adds the how.

- Use Navigation 3 for new Compose projects. Existing projects keep their library (Navigation Compose 2.x, Fragment/XML); never migrate without asking.
- Navigation that depends on logic is a UI-state flag the screen observes, acts on through its callback, and clears, unless the project already has a navigation-action mechanism. No `ViewModelNav`/`NavigationAction` delegate as a default.
- Return results to a previous screen as state (Google's results-as-state recipe), unless the project already has a result mechanism. Avoid the in-memory `ResultEventBus`: it loses results on process death.
- Top-level destinations use `NavigationSuiteScaffold`, unless the project already has its own scaffold.
- On Navigation 3, multi-pane layouts use `ListDetailSceneStrategy`/`SupportingPaneSceneStrategy` with entry metadata, not `ListDetailPaneScaffold`, unless the project already uses the latter.
- On Navigation 3 the key does not reach `SavedStateHandle`: pass it through an assisted Hilt factory, unless the project already has another way. Import `hiltViewModel` from `androidx.hilt.lifecycle.viewmodel.compose`; the `androidx.hilt.navigation.compose` one is deprecated since Hilt 1.3.0.

```kotlin
@HiltViewModel(assistedFactory = DetailViewModel.Factory::class)
class DetailViewModel @AssistedInject constructor(
    @Assisted private val route: DetailRoute,
    private val repo: DetailRepository,
) : ViewModel() {
    @AssistedFactory interface Factory { fun create(route: DetailRoute): DetailViewModel }
}

entry<DetailRoute> { route ->
    DetailScreen(
        viewModel = hiltViewModel<DetailViewModel, DetailViewModel.Factory>(
            creationCallback = { it.create(route) },
        ),
    )
}
```

- Use `NavigationBackHandler` for new back handling outside navigation, unless the project already uses `BackHandler` or `PredictiveBackHandler`.
