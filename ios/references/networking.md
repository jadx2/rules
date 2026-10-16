# Networking

SKILL.md musts apply: `Sendable` `Codable` models, and existing projects keep their library.

- Use `URLSession` with async/await and no networking dependency, unless the project already uses a networking library. Why: fewer dependencies, and the platform covers the common cases.
- Alamofire is the approved exception when a project needs interceptors, retry or certificate pinning, unless the project already uses another library.
- Leave Moya out of new projects, unless the project already uses it. Why: last release 2022-08.
- In a module with MainActor default isolation, declare models `nonisolated` so their `Codable` conformance works off the main actor.

```swift
nonisolated struct User: Codable, Sendable {
    let id: String
    let displayName: String
}

func user(id: String) async throws -> User {
    let (data, _) = try await session.data(from: baseURL.appending(path: "users/\(id)"))
    return try decoder.decode(User.self, from: data)
}
```
