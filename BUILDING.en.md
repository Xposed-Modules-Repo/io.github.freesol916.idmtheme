# Building this module

[简体中文](BUILDING.md) | **English**

The module mixes two worlds:

- **Hooks** (`ThemeHooks`, `DialogHooks`, `DrawableHooks`, `Palette`, `RemoteConfig`,
  `SquircleDrawable`, `IdmThemeModule`, `ConfigProvider`) — plain Java, ~1 kLOC.
- **Settings UI** (`SettingsActivity.kt`) — Kotlin + Compose Multiplatform + Miuix.

Because of Compose, the project is built with Gradle instead of the plain
`aapt2 + javac + d8` pipeline used in v1.0.

## Toolchain

| Component | Version |
|---|---|
| JDK | 17 |
| Gradle | 8.14.3 |
| Android Gradle Plugin | 8.13.2 |
| Kotlin | 2.4.21 |
| Compose Multiplatform | 1.10.3 |
| Miuix | `top.yukonga.miuix.kmp:miuix-ui:0.9.0` |
| LibXposed API | 102 (`compileOnly`, provided at runtime by LSPosed) |
| compileSdk / targetSdk / minSdk | 36 / 34 / 29 |

Result: 2.0 MB release APK after R8 (19 MB without shrinking).

## Working around the arm64 build host

Google ships `aapt2` only for `linux-x86_64`, and the Debian `aapt2` is 2.19 (2020) which cannot
parse a modern `android.jar` (`RES_TABLE_TYPE_TYPE entry offsets overlap actual entry data`).
Four changes make the build work on an arm64-only machine:

1. **Old android.jar for resource linking.** `compileSdk` stays 36, but
   `platforms/android-36/android.jar` is replaced with the API 32 jar, which `aapt2` 2.19 can parse.
2. **`SysColors.java`.** The API 32 `android.R.color` does not contain the M3 semantic tokens yet
   (`system_primary_light` …), so their numeric values are exported from the API 36 jar into
   `SysColors` — same framework resource IDs, still compile-time constants.
3. **Patched AAR metadata.** Three Miuix artifacts declare `minCompileSdk=37`, which API 36 cannot
   satisfy. A local Maven repo (`patched-repo/`) holds copies of those AARs with the metadata
   rewritten to 36; it is listed first in `dependencyResolutionManagement`.
4. **aapt2 override.** `android.aapt2FromMavenOverride=/usr/bin/aapt2` points AGP at the native
   arm64 binary instead of downloading the x86_64 one.

Mirrors used (no access to `dl.google.com` needed): Gradle from Tencent mirror, AGP/Kotlin/Compose
from Aliyun Maven mirrors.

## Build commands

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-arm64
export GRADLE_USER_HOME=$PWD/gradle-home

./gradle-8.14.3/bin/gradle assembleRelease --no-daemon
# -> app/build/outputs/apk/release/app-release.apk
```

Signing uses the same key as the v1.0 release (configured in `app/build.gradle.kts`), so the APK
upgrades over previous versions.

## Module metadata

`app/src/main/resources/META-INF/xposed/` is packed into the APK root:

- `module.prop` — `minApiVersion=102`, `targetApiVersion=102`, `staticScope=true`
- `java_init.list` — `io.github.freesol916.idmtheme.IdmThemeModule`
- `scope.list` — `idm.internet.download.manager.plus`

R8 keeps `io.github.freesol916.idmtheme.**` (see `app/proguard-rules.pro`) because the entry class is
loaded by name from `java_init.list`, not referenced from code.
