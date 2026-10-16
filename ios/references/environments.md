# Environments and configuration

SKILL.md musts apply: no secrets in xcconfig or source, and required config fails fast at launch.

- Use one app target with build configurations and schemes for Dev, QA and Prod, unless the project already has another environment setup.
- Keep one xcconfig per configuration, unless the project already configures environments differently.
- Give each environment a bundle-ID suffix (for example `.dev`, `.qa`), unless the project already has a naming convention. Why: all environments install side by side.
- Give each configuration its own app icon through `ASSETCATALOG_COMPILER_APPICON_NAME`, unless the project already has a convention for this.
- Read required config once at launch into a typed `AppConfig`, unless the project already has a config loader.

```swift
struct AppConfig {
    let baseURL: URL

    init(bundle: Bundle = .main) {
        guard let raw = bundle.object(forInfoDictionaryKey: "API_BASE_URL") as? String,
              let url = URL(string: raw) else {
            preconditionFailure("Missing or invalid API_BASE_URL in Info.plist; check the xcconfig for this configuration.")
        }
        baseURL = url
    }
}
```

- Create `AppConfig` in the app's entry point.
