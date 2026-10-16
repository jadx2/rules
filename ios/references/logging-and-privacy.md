# Logging and privacy

The SKILL.md musts apply: `os.Logger` with private data marked, current privacy manifest, project's crash SDK.

## Logger
- Define one `Logger` per feature or layer, with subsystem = bundle ID, unless the project already has a logging wrapper or convention.

```swift
extension Logger {
    private static let subsystem = Bundle.main.bundleIdentifier ?? "app"
    static let networking = Logger(subsystem: subsystem, category: "networking")
    static let profile = Logger(subsystem: subsystem, category: "profile")
}
```

- Mark sensitive values `privacy: .private` explicitly, using `.private(mask: .hash)` for IDs you need to correlate, and mark non-sensitive strings `.public`, because interpolated strings are redacted by default. Follow the project's redaction convention instead if it already has one.

## Privacy manifest
- Keep `PrivacyInfo.xcprivacy` in the app target and in each package that uses required-reason APIs.
- Required-reason API categories to watch:
  - file timestamp (`creationDate`, `modificationDate`, `stat`)
  - system boot time (`ProcessInfo.processInfo.systemUptime`, `mach_absolute_time()`)
  - disk space (`volumeAvailableCapacityKey`, `systemFreeSize`, `statfs`)
  - active keyboards (`UITextInputMode.activeInputModes`)
  - UserDefaults (any read or write)

## Crash reporting
- No default crash reporter. Use the project's SDK and keep its privacy-manifest entries (collected data types, tracking domains) current when it is upgraded. If the project has none, ask before adding one.
