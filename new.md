# Choicely RN — Android Integration Guide

Integrate the Choicely React Native SDK into your native Android project.

---

## Prerequisites

- Node.js >= 20
- Android SDK with CMake
- Git
- Java 17

---

## Project Structure (after setup)

```
Android/Java/
├── app/                        ← your Android app module
├── app-react-native/           ← cloned choicely-rn repo
│   ├── node_modules/
│   ├── package.json
│   └── react-native.config.js
├── build.gradle
├── settings.gradle
├── gradle.properties
└── gradle/libs.versions.toml
```

---

## Setup

### Step 1 — Clone the Choicely RN repo

```bash
cd Android/Java
git clone https://github.com/choicely/choicely-rn.git app-react-native
cd app-react-native && npm install && cd ..
```

> The folder **must** be named `app-react-native` by default.
> To use a different name, set `choicelyRnDir` in `gradle.properties` (see Step 2).

---

### Step 2 — `gradle.properties`

```properties
newArchEnabled=true
hermesEnabled=true

# RN module folder name — change this ONE line if you rename the folder
choicelyRnDir=app-react-native
```

---

### Step 3 — `settings.gradle`

```groovy
pluginManagement {
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        mavenCentral()
        gradlePluginPortal()
        maven { url 'https://jitpack.io' }
    }
    // Load the RN Gradle plugin from your local node_modules
    includeBuild("${settings.providers.gradleProperty('choicelyRnDir').orElse('app-react-native').get()}/node_modules/@react-native/gradle-plugin")
}
plugins {
    id "com.facebook.react.settings"
}
def rnDir = settings.providers.gradleProperty("choicelyRnDir").orElse("app-react-native").get()
reactSettings {
    autolinkLibrariesFromCommand(
        [ ["/opt/homebrew/bin/node", "/usr/local/bin/node"].find { new File(it).exists() } ?: "node",
          "node_modules/react-native/cli.js", "config" ],
        file(rnDir)
    )
}
dependencyResolutionManagement {
    // PREFER_SETTINGS required — RN libs add their own Maven repos dynamically
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
        maven { url 'https://central.sonatype.com/repository/maven-snapshots/' }
    }
}
rootProject.name = "YourApp"
include ':app'
```

> **Why `includeBuild` and `reactSettings` can't be automated by a plugin:**
> They run in Gradle's *settings phase* — before any plugin from Maven Central can load.
> These will always need to be in your `settings.gradle`.

---

### Step 4 — Root `build.gradle`

```groovy
buildscript {
    ext.kotlin_version = '2.2.20'
    ext {
        minSdkVersion      = 26
        compileSdkVersion   = 36
        buildToolsVersion   = "35.0.0"
        targetSdkVersion    = 36
        reactNativeVersion  = "0.82.0"
        REACT_NATIVE_NODE_MODULES_DIR = file("${rootDir}/${choicelyRnDir}/node_modules/react-native").canonicalPath
        REACT_NATIVE_WORKLETS_NODE_MODULES_DIR = file("${rootDir}/${choicelyRnDir}/node_modules/react-native-worklets").canonicalPath
    }
    repositories {
        google()
        mavenLocal()
        mavenCentral()
        maven { url "$rootDir/${choicelyRnDir}/node_modules/react-native/android" }
        maven { url 'https://central.sonatype.com/repository/maven-snapshots/' }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.13.2'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
        // Choicely RN Gradle plugin — provides com.choicely.rn.setup & com.choicely.react
        classpath 'com.choicely.sdk:choicely-rn-gradle:0.0.3-SNAPSHOT'
    }
}

plugins {
    alias(libs.plugins.android.application) apply false
    id "com.facebook.react" apply false
}

// REQUIRED: must be applied on the root project
apply plugin: 'com.choicely.rn.setup'
```

---

### Step 5 — App `build.gradle`

```groovy
plugins {
    alias(libs.plugins.android.application)
}
// com.facebook.react must be applied directly (Gradle ClassLoaderScope limitation)
apply plugin: 'com.choicely.react'
apply plugin: 'com.facebook.react'

// Vector icons
def rnDir = findProperty("choicelyRnDir") ?: "app-react-native"
project.ext.vectoricons = [iconFontsDir: "${rootDir}/${rnDir}/node_modules/react-native-vector-icons/Fonts"]
apply from: "${rootDir}/${rnDir}/node_modules/react-native-vector-icons/fonts.gradle"

android {
    namespace 'com.your.app'
    compileSdk 36

    defaultConfig {
        applicationId "com.your.app"
        minSdk 26
        targetSdk 36
        versionCode 1
        versionName "1.0"
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }

    buildFeatures {
        viewBinding true
    }

    // Required: resolve duplicate libworklets.so from reanimated & worklets
    packagingOptions {
        pickFirst 'lib/arm64-v8a/libworklets.so'
        pickFirst 'lib/x86/libworklets.so'
        pickFirst 'lib/x86_64/libworklets.so'
        pickFirst 'lib/armeabi-v7a/libworklets.so'
    }
}

dependencies {
    // Choicely SDK
    implementation("com.choicely.sdk:android-core:1.1.2-SNAPSHOT")
    implementation("com.choicely.sdk:android-rn:0.0.3-SNAPSHOT")

    // AndroidX
    implementation libs.constraintlayout
    implementation libs.lifecycle.livedata.ktx
    implementation libs.lifecycle.viewmodel.ktx
    implementation libs.navigation.fragment
    implementation libs.navigation.ui
}
```

---

### Also needed — `react-native.config.js`

**File:** `app-react-native/react-native.config.js`

> Must use dots in the filename — NOT `react-native-config.js`.

```js
module.exports = {
  project: {
    android: {
      sourceDir: "../",
      appName: "app",
      packageName: "com.your.app",  // must match applicationId
    },
  },
};
```

---

## Artifacts Overview

### Gradle Plugin: `com.choicely.sdk:choicely-rn-gradle`

Distributed via Maven Central Snapshots. Provides two plugin IDs:

| Plugin ID | Applied in | Purpose |
|---|---|---|
| `com.choicely.rn.setup` | Root `build.gradle` | Node binary detection, lib patching, CMake fix, PATH injection |
| `com.choicely.react` | App `build.gradle` | Configures `react{}` paths, ext vars, `autolinkLibrariesWithApp()` |

### Runtime Library: `com.choicely.sdk:android-rn`

AAR library containing:

| Component | Description |
|---|---|
| `ChoicelyRNApplication` | Abstract `Application` implementing `ReactApplication`, manages multiple `ReactHost` instances |
| `ChoicelyRNHost` | Extends `DefaultReactNativeHost`, handles JS bundle loading (asset or remote) |
| `ChoicelyDefaultReactHost` | Factory for creating `ReactHost` instances |
| `ChoicelyReactNativeFragment` | Fragment for embedding RN views in native screens |
| `RNFragmentWrapper` | Wrapper around the RN fragment |
| `ChoicelyDeepLinkScreenActivity` | Activity for deep link handling |
| `ChoicelyBridgePackage` / `ChoicelyRNBridge` | Native-to-RN bridge module |
| `ChoicelyRNConfig` | Config for current version/app key for remote bundles |
| `ChoicelyRemoteBundle` | Handles remote JS bundle downloading |

**Transitive dependencies** (pulled automatically):
- `com.choicely.sdk:android-core`
- `com.facebook.react:react-android:0.82.0`
- `com.facebook.react:hermes-android:0.82.0`
- `androidx.databinding:*:8.13.0`
- `org.jetbrains.kotlin:kotlin-stdlib:2.1.21`

---

## Build Commands

```bash
# Build
./gradlew :app:assembleDebug

# Clean + Build
./gradlew clean :app:assembleDebug
```

---

## Renaming the RN folder

Change **one line** in `gradle.properties`:

```properties
choicelyRnDir=my-custom-folder-name
```

---

## What the Gradle plugin handles automatically

| What | Plugin ID |
|---|---|
| Node binary detection (macOS Homebrew / `/usr/local`) | `com.choicely.rn.setup` |
| Patching third-party libs that hardcode `"node"` | `com.choicely.rn.setup` |
| Injecting Node into PATH for all Exec tasks | `com.choicely.rn.setup` |
| CMake cache clean fix | `com.choicely.rn.setup` |
| `react{}` block (node, cliFile, reactNativeDir, codegenDir) | `com.choicely.react` |
| `autolinkLibrariesWithApp()` | `com.choicely.react` |

## What still requires manual setup (Gradle hard limits)

| What | Why |
|---|---|
| `includeBuild(".../gradle-plugin")` in `settings.gradle` | Settings phase — no plugin can run before this |
| `reactSettings { autolinkLibrariesFromCommand(...) }` | Settings phase — same reason |
| `apply plugin: 'com.facebook.react'` in `app/build.gradle` | ClassLoaderScope — can't be applied from another plugin |

---

## Troubleshooting

| Error | Fix |
|---|---|
| `Could not find project.android.packageName` | Rename to `react-native.config.js` (dots, not dashes) |
| `includeBuild path not found` | Run `npm install` in `app-react-native/` first |
| `pluginManagement must appear first` | No code before `pluginManagement {}` in `settings.gradle` |
| `Node binary not found` | Plugin finds Node automatically; ensure Homebrew or nvm is installed |
| `Cannot find module '@react-native/codegen'` | `choicelyRnDir` may point to wrong folder — check `gradle.properties` |
| `Failed to apply plugin 'com.choicely.react'` | Add `apply plugin: 'com.choicely.rn.setup'` in root `build.gradle` |
| `2 files found with path 'lib/arm64-v8a/libworklets.so'` | Add `packagingOptions { pickFirst 'lib/*/libworklets.so' }` in app `build.gradle` |
| `Included build 'choicely-rn-android/rn-gradle-plugin' does not exist` | Remove the `includeBuild("choicely-rn-android/rn-gradle-plugin")` line — the plugin is now distributed via Maven |
