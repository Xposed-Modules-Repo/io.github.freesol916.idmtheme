# 构建这个模块

**简体中文** | [English](BUILDING.en.md)

这个模块由两部分组成：

- **Hook 代码**（`ThemeHooks`、`DialogHooks`、`DrawableHooks`、`Palette`、`RemoteConfig`、
  `SquircleDrawable`、`IdmThemeModule`、`ConfigProvider`）—— 纯 Java，约 1k 行。
- **设置界面**（`SettingsActivity.kt`）—— Kotlin + Compose Multiplatform + Miuix。

因为引入了 Compose，工程改用 Gradle 构建，不再是 v1.0 那套
`aapt2 + javac + d8` 的裸管线。

## 工具链

| 组件 | 版本 |
|---|---|
| JDK | 17 |
| Gradle | 8.14.3 |
| Android Gradle Plugin | 8.13.2 |
| Kotlin | 2.4.21 |
| Compose Multiplatform | 1.10.3 |
| Miuix | `top.yukonga.miuix.kmp:miuix-ui:0.9.0` |
| LibXposed API | 102（`compileOnly`，运行时由 LSPosed 提供） |
| compileSdk / targetSdk / minSdk | 36 / 34 / 29 |

结果：R8 压缩后的 release APK 为 **2.0 MB**（不压缩是 19 MB）。

## 在 arm64 机器上绕过的四个坑

Google 只发布 `linux-x86_64` 的 `aapt2`，而 Debian 自带的 `aapt2` 是 2.19（2020 年），
它读不了新版 `android.jar`，会报：

```
RES_TABLE_TYPE_TYPE entry offsets overlap actual entry data
```

下面四处改动让构建能在一台纯 arm64 的机器上跑通：

1. **资源链接改用旧版 android.jar。** `compileSdk` 保持 36，但把
   `platforms/android-36/android.jar` 换成 API 32 的 jar —— 这一版 `aapt2` 2.19 能正常解析。
2. **`SysColors.java`。** API 32 的 `android.R.color` 里还没有 M3 语义令牌
   （`system_primary_light` 这一批），于是从 API 36 的 jar 里把它们的数值导出成
   `SysColors` —— 资源 ID 完全相同，仍然是编译期常量。
3. **修补 AAR 元数据。** 有三个 Miuix 的 aar 声明了 `minCompileSdk=37`，API 36 满足不了。
   本地建了 `patched-repo/`，存放把这几个 aar 的元数据改写成 36 的副本，
   并在 `dependencyResolutionManagement` 里排在最前面解析。
4. **指定 aapt2。** `android.aapt2FromMavenOverride=/usr/bin/aapt2` 让 AGP 使用本机的
   arm64 原版二进制，而不是去 Maven 拉 x86_64 版本。

用到的镜像（全程不需要访问 `dl.google.com`）：Gradle 走腾讯镜像，
AGP / Kotlin / Compose 走阿里云 Maven 镜像。

## 构建命令

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-arm64
export GRADLE_USER_HOME=$PWD/gradle-home

./gradle-8.14.3/bin/gradle assembleRelease --no-daemon
# -> app/build/outputs/apk/release/app-release.apk
```

签名沿用 v1.0 的同一把密钥（配置在 `app/build.gradle.kts` 里），所以新 APK 可以直接覆盖升级。

## 模块元数据

`app/src/main/resources/META-INF/xposed/` 会被打进 APK 根目录：

- `module.prop` —— `minApiVersion=102`、`targetApiVersion=102`、`staticScope=true`
- `java_init.list` —— `io.github.freesol916.idmtheme.IdmThemeModule`
- `scope.list` —— `idm.internet.download.manager.plus`

R8 会保留 `io.github.freesol916.idmtheme.**`（见 `app/proguard-rules.pro`），
因为入口类是被 `java_init.list` 按名字反射加载的，代码里没有任何直接引用。
