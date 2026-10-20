# Data and networking

The SKILL.md musts (`CancellationException`, typed failures, DTO separation, no hard-coded URLs or keys) apply; this file adds the how.

- Use Retrofit 3 + OkHttp + the kotlinx.serialization converter, unless the project already uses another networking library.
- Use Apollo Kotlin for GraphQL, unless the project already uses another client. Use Ktor only for KMP.
- Repositories call the service directly, unless the project already has data sources. Add a data source class only when coordinating several sources (remote + local).
- Go offline-first when the app has a local cache: the local store is the source of truth, unless the project already has another caching approach.
- Repositories return a sealed result, unless the project already has a result type. Use no `Loading` case and prefer composition over a `BaseRepository` base class.

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>
    data class Failure(val error: DataError) : ApiResult<Nothing>
}

sealed interface DataError {
    data object Network : DataError
    data object Unauthorized : DataError
    data class Server(val code: Int) : DataError
    data object Serialization : DataError
    data object Unknown : DataError
}
```
