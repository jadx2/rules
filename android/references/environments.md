# Environments and configuration

The SKILL.md must (required config is read once at startup and fails fast) applies; this file adds the how.

- Use one `environment` flavor dimension with `dev`, `qa`, `prod` product flavors, unless the project already has another environment setup. `setDimension` is gone in AGP 9: assign `dimension = "environment"`.
- Add `applicationIdSuffix` `.dev` and `.qa`, unless the project already has a naming convention.
- Set base URLs through `buildConfigField`, unless the project already has a config mechanism. It needs `buildFeatures { buildConfig = true }`, and String values carry escaped quotes.
- Put app name and launcher icon per flavor in `src/<flavor>/res`, unless the project already has a convention for this.
- Use `debug`/`release` build types, unless the project already defines others.
- Set `isMinifyEnabled`, `isShrinkResources` and `proguard-android-optimize.txt` explicitly on release, unless the project already does. AGP 9 rejects `proguard-android.txt`.

```kotlin
buildFeatures { buildConfig = true }
flavorDimensions += "environment"
productFlavors {
    create("dev") {
        dimension = "environment"
        applicationIdSuffix = ".dev"
        buildConfigField("String", "BASE_URL", "\"https://api.dev.example.com/\"")
    }
}
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
    }
}
```
