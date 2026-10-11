# PipePipe 构建环境装配记录

- 日期：2026-10-11
- 目的：让本机能编译 `PipePipeClient` 的 debug APK
- 变更范围：**全部在用户 HOME 下，未修改任何仓库文件**

## 1. 原环境缺口

| 缺口 | 原状 | 影响 |
|---|---|---|
| JDK 25 | 只有 17 / 21 / 27 | `JavaLanguageVersion.of(25)` toolchain 无法解析 |
| Android platform 37 | 只有 android-35 | `compileSdk 37` 无法编译 |
| Android build-tools 37 | 只有 34.0.0 | 同上 |
| cmdline-tools | 12.0（读不懂 SDK XML v4） | 连 `sdkmanager --list` 都看不到新包 |
| Gradle 9.5.1 | 只有 8.6 / 8.10.2 | wrapper 需现下（services.gradle.org 仅 ~50 KB/s） |
| Maven Central 速度 | 322 KB/s（JitPack 26 KB/s） | 依赖拉取会拖到几十分钟 |

> 注意：新版 Android 的平台包名是 **`platforms;android-37.0`**，没有裸的 `platforms;android-37`。

## 2. 已安装内容（全部在 HOME 下）

| 项 | 位置 | 来源 |
|---|---|---|
| Temurin JDK 25.0.4.1 | `~/.jdks/temurin-25` | 清华 Adoptium 镜像（6 秒） |
| cmdline-tools 21.0 | `~/Android/Sdk/cmdline-tools/21.0` | dl.google.com |
| platform 37.0 | `~/Android/Sdk/platforms/android-37.0` | sdkmanager 21.0 |
| build-tools 37.0.0 / 36.0.0 | `~/Android/Sdk/build-tools/` | sdkmanager（36 由 AGP 自动补装） |
| Gradle 9.5.1-all | `~/.gradle/wrapper/dists/gradle-9.5.1-all/3mo7ofu40rhxhyro6vr9xd6jp/` | 腾讯镜像（11 秒 / 240MB） |

配置文件（均在 HOME 下，可随时删除）：

```properties
# ~/.gradle/gradle.properties
org.gradle.java.installations.paths=/home/yangsanyi/.jdks/temurin-25
```

```groovy
// ~/.gradle/init.gradle  —— 国内镜像加速
def ALIYUN = [
    'https://maven.aliyun.com/repository/public',
    'https://maven.aliyun.com/repository/google',
    'https://maven.aliyun.com/repository/gradle-plugin',
]
beforeSettings { settings ->
    settings.pluginManagement.repositories { ALIYUN.each { u -> maven { url = u } } }
    settings.dependencyResolutionManagement.repositories { ALIYUN.each { u -> maven { url = u } } }
}
```

```properties
# PipePipeClient/local.properties （已被 .gitignore 忽略）
sdk.dir=/home/yangsanyi/Android/Sdk
```

## 3. 实测速度（选镜像的依据）

| 目标 | 速度 |
|---|---|
| 腾讯 gradle 镜像 | 17.4 MB/s |
| 清华 Adoptium | —（141MB / 6s） |
| dl.google.com（Google Maven） | 12.2 MB/s ← 直连即可 |
| maven.aliyun.com | 3.7 MB/s |
| repo1.maven.org（Maven Central） | 0.32 MB/s ← 需要镜像 |
| jitpack.io | 0.026 MB/s ← 仍需直连，但只涉及小包 |
| services.gradle.org | ~0.05 MB/s ← 需要镜像 |
| socks5 127.0.0.1:7897 | ~0.09 MB/s（比直连还慢，未采用） |

## 4. 构建命令

```bash
cd PipePipeClient
ANDROID_HOME=$HOME/Android/Sdk JAVA_HOME=$HOME/.jdks/temurin-25 ./gradlew :app:assembleDebug
```

产物：`PipePipeClient/app/build/outputs/apk/debug/app-debug.apk`

## 5. 如何回退

```bash
rm ~/.gradle/init.gradle
rm -rf ~/.jdks/temurin-25
rm -rf ~/Android/Sdk/cmdline-tools/21.0 ~/Android/Sdk/platforms/android-37.0
rm -rf ~/.gradle/wrapper/dists/gradle-9.5.1-all
# ~/.gradle/gradle.properties 删掉最后一行 org.gradle.java.installations.paths
```
