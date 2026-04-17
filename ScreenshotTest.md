# ScreenshotTest SKILL — Compose Preview Screenshot Testing

> **Goal:** Use this skill to introduce Compose Screenshot Testing into any Android project. Follow each phase in order. Read all notes carefully before executing commands.

---

## Overview

Compose Preview Screenshot Testing is a host-side screenshot testing tool that turns `@Preview` composables into automated visual regression tests. It generates reference images and an HTML diff report — no emulator or device needed.

**Key facts:**
- Plugin ID: `com.android.compose.screenshot`
- Current stable alpha: `0.0.1-alpha14` or find the latest version
- Tests live in a dedicated `screenshotTest` source set (separate from `main` and `test`)
- Reference images are stored in source control alongside code
- Two Gradle tasks drive the workflow: `update` (capture) → `validate` (compare)

---

## Phase 1 — Verify Requirements

Before touching any files, confirm the project meets these minimum versions:

| Requirement | Gradle-tasks only | Full IDE integration |
|---|---|---|
| Android Gradle Plugin (AGP) | ≥ 8.5.0 | ≥ 9.0.0 |
| Kotlin | ≥ 1.9.20 (2.0+ recommended) | ≥ 2.2.10 |
| JDK | ≥ 17 | ≥ 17 |
| Android Studio | any | Panda 1 Canary 4+ |
| Screenshot plugin | ≥ 0.0.1-alpha13 | ≥ 0.0.1-alpha13 |

Check versions:
```bash
./gradlew --version          # Shows AGP and Gradle version
java -version                # Must be 17+
```

If the project **cannot** meet the AGP 8.5+ requirement, stop and escalate — the plugin will not work.

---

## Phase 2 — Add Dependencies & Plugins

### 2a. If the project uses a TOML Version Catalog (`gradle/libs.versions.toml`)

Open `gradle/libs.versions.toml` and add the following entries. Insert into the **existing** `[versions]`, `[libraries]`, and `[plugins]` sections — do NOT duplicate section headers.

```toml
# ── [versions] ──────────────────────────────────────────────────────────────
agp = "9.0.0-rc03"          # bump if already higher; never downgrade
kotlin = "2.1.20"           # bump if already higher; never downgrade
screenshot = "0.0.1-alpha13"

# ── [libraries] ─────────────────────────────────────────────────────────────
screenshot-validation-api = { group = "com.android.tools.screenshot", name = "screenshot-validation-api", version.ref = "screenshot" }
androidx-ui-tooling         = { group = "androidx.compose.ui",         name = "ui-tooling" }

# ── [plugins] ────────────────────────────────────────────────────────────────
screenshot = { id = "com.android.compose.screenshot", version.ref = "screenshot" }
```

> **Note:** If `agp` or `kotlin` version refs already exist with higher values, keep the higher values. Only add the `screenshot` entries.

### 2b. If the project does NOT use a TOML catalog

Use hardcoded versions directly in the Gradle files (see Phase 3 below — substitute `libs.*` refs with string literals).

---

## Phase 3 — Configure Gradle Files

### 3a. Root-level `build.gradle.kts` (or `build.gradle`)

Add the screenshot plugin to the `plugins {}` block using `apply false` so it is available to submodules:

```kotlin
// build.gradle.kts  (root)
plugins {
    alias(libs.plugins.screenshot) apply false
    // ... existing plugins ...
}
```

**Without TOML:**
```kotlin
plugins {
    id("com.android.compose.screenshot") version "0.0.1-alpha13" apply false
}
```

### 3b. Module-level `build.gradle.kts` (typically `app/build.gradle.kts`)

**Step 1 — Apply the plugin:**
```kotlin
plugins {
    // ... existing plugins ...
    alias(libs.plugins.screenshot)
}
```

**Without TOML:**
```kotlin
plugins {
    id("com.android.compose.screenshot")
}
```

**Step 2 — Enable the experimental flag inside the `android {}` block:**
```kotlin
android {
    // ... existing config ...
    experimentalProperties["android.experimental.enableScreenshotTest"] = true
}
```

**Step 3 — Add dependencies:**
```kotlin
dependencies {
    // ... existing dependencies ...
    screenshotTestImplementation(libs.screenshot.validation.api)
    screenshotTestImplementation(libs.androidx.ui.tooling)
}
```

**Without TOML:**
```kotlin
dependencies {
    screenshotTestImplementation("com.android.tools.screenshot:screenshot-validation-api:0.0.1-alpha13")
    screenshotTestImplementation("androidx.compose.ui:ui-tooling")
}
```

### 3c. `gradle.properties` (project root)

Add this line:
```properties
android.experimental.enableScreenshotTest=true
```

---

## Phase 4 — Create the `screenshotTest` Source Set

Create the directory structure:
```
app/
└── src/
    └── screenshotTest/
        └── kotlin/
            └── com/example/yourapp/          ← match your app's package
                └── ExamplePreviewScreenshotTest.kt
```

**Shell commands to scaffold:**
```bash
mkdir -p app/src/screenshotTest/kotlin/com/example/yourapp
```

Replace `com/example/yourapp` with the actual package path of the project.

---

## Phase 5 — Write a Sample Screenshot Test

Create `ExamplePreviewScreenshotTest.kt` in the path created above. Use the package that matches the project.

```kotlin
package com.example.yourapp   // ← update to actual package

import androidx.compose.runtime.Composable
import androidx.compose.ui.tooling.preview.Preview
import com.android.tools.screenshot.PreviewTest
import com.example.yourapp.ui.theme.MyApplicationTheme   // ← update to actual theme

// ─── Single preview test ─────────────────────────────────────────────────────

@PreviewTest
@Preview(showBackground = true, name = "Light mode")
@Composable
fun GreetingPreviewLight() {
    MyApplicationTheme {
        Greeting("Android!")   // ← replace with a real composable from the project
    }
}

// ─── Dark mode variant ────────────────────────────────────────────────────────

@PreviewTest
@Preview(
    showBackground = true,
    name = "Dark mode",
    uiMode = android.content.res.Configuration.UI_MODE_NIGHT_YES
)
@Composable
fun GreetingPreviewDark() {
    MyApplicationTheme {
        Greeting("Android!")
    }
}

// ─── Multi-preview example (optional) ────────────────────────────────────────

@PreviewTest
@Preview(name = "Small font",  fontScale = 0.85f, showBackground = true)
@Preview(name = "Large font",  fontScale = 1.30f, showBackground = true)
@Composable
fun GreetingPreviewFontScale() {
    MyApplicationTheme {
        Greeting("Android!")
    }
}
```

> **Important rules:**
> - Every function to be tested **must** be annotated with both `@PreviewTest` **and** at least one `@Preview`.
> - All screenshot test files **must** live inside `src/screenshotTest/`, not `src/main/` or `src/test/`.
> - Renaming a `@PreviewTest` function breaks the link to its reference image — regenerate references after any rename.

---

## Phase 6 — Sync & Build Verification

Before running screenshot tasks, confirm the project compiles cleanly:

```bash
./gradlew assembleDebug
```

If this fails, fix compile errors first. Screenshot tasks depend on a successful build.

---

## Phase 7 — Generate Reference Images (Update Task)

Run the **update** task to render and store reference screenshots. The task detects the correct variant automatically; fall back to a `QADebug` or other debug variant if `Debug` is not available.

```bash
# Standard debug variant
./gradlew updateDebugScreenshotTest

# If the project uses a custom variant (e.g., QADebug):
./gradlew updateQADebugScreenshotTest

# Specific module (monorepo / multi-module):
./gradlew :app:updateDebugScreenshotTest
```

**What happens:**
- Composables are rendered on the host JVM (no device needed).
- Reference PNG files are written to:
  ```
  app/src/screenshotTestDebug/reference/
  ```
- File names are formed from the fully-qualified function name + a hash of preview parameters, e.g.:
  ```
  com.example.yourapp.GreetingPreviewLight_da39a3ee_c2200e98_0.png
  ```

**After success:**
1. Open the first generated reference image in Android Studio:
   - In the **Project** panel, navigate to `app/src/screenshotTestDebug/reference/`
   - Double-click the first `.png` file to open it in the image viewer.

> **Commit reference images to source control.** They are the baseline for future comparisons.

---

## Phase 8 — Validate Screenshots (Validate Task)

Run the **validate** task to compare freshly rendered screenshots against the reference images:

```bash
# Standard debug variant
./gradlew validateDebugScreenshotTest

# Custom variant:
./gradlew validateQADebugScreenshotTest

# Specific module:
./gradlew :app:validateDebugScreenshotTest
```

**Outcome:**
- **PASS** → all rendered images match references within the configured threshold.
- **FAIL** → at least one image differs. An HTML report is generated at:
  ```
  app/build/reports/screenshotTest/preview/debug/index.html
  ```

**After the task completes:**

Open the HTML report in the default web browser:

```bash
# macOS
open app/build/reports/screenshotTest/preview/debug/index.html

# Linux
xdg-open app/build/reports/screenshotTest/preview/debug/index.html

# Windows (PowerShell)
Start-Process app\build\reports\screenshotTest\preview\debug\index.html
```

The report shows side-by-side **Reference / Actual / Diff** images for every test, including the percentage difference.

---

## Phase 9 — Optional: Configure Image Difference Threshold

By default, any pixel difference causes a failure. To allow minor rendering variations (e.g., anti-aliasing), set a threshold in the module's `build.gradle.kts`:

```kotlin
android {
    testOptions {
        screenshotTests {
            imageDifferenceThreshold = 0.0001f  // 0.01% — adjust as needed
        }
    }
}
```

---

## Phase 10 — CI Integration

Add to your CI pipeline (GitHub Actions example):

```yaml
- name: Generate reference screenshots
  run: ./gradlew updateDebugScreenshotTest

- name: Validate screenshots
  run: ./gradlew validateDebugScreenshotTest

- name: Upload HTML report on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: screenshot-test-report
    path: app/build/reports/screenshotTest/
```

> Store reference images in the repository so CI can compare against them on every pull request.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Could not find com.android.compose.screenshot` | Plugin not in classpath | Add `apply false` entry to root `build.gradle.kts` and confirm `mavenGoogle()` is in `pluginManagement` repositories |
| `Unresolved reference: screenshotTestImplementation` | Experimental flag missing | Add `android.experimental.enableScreenshotTest=true` to `gradle.properties` AND `experimentalProperties[...]` inside `android {}` |
| `@PreviewTest not found` | Missing `screenshot-validation-api` dependency | Verify `screenshotTestImplementation` dependency is present |
| Reference images not generated | Wrong source set path | Files must be under `src/screenshotTest/`, not `src/test/` |
| Tests pass locally but fail on CI | Missing reference images in repo | Commit `src/screenshotTestDebug/reference/` to version control |
| JDK incompatibility error | JDK < 17 | Set `JAVA_HOME` to JDK 17+ or configure in `gradle.properties`: `org.gradle.java.home=/path/to/jdk17` |
| Renamed function breaks tests | Reference image linked to old name | Re-run `updateDebugScreenshotTest` after any `@PreviewTest` function rename |

---

## Quick Reference — File Checklist

```
✅ gradle/libs.versions.toml          → versions, libraries, plugins entries added
✅ gradle.properties                   → android.experimental.enableScreenshotTest=true
✅ build.gradle.kts (root)             → screenshot plugin with apply false
✅ app/build.gradle.kts                → plugin applied, experimentalProperties, dependencies
✅ app/src/screenshotTest/kotlin/…     → @PreviewTest annotated composable(s)
✅ app/src/screenshotTestDebug/        → reference images (generated + committed)
```

---

## Quick Reference — Gradle Task Cheat Sheet

```bash
# First time setup — generate reference images
./gradlew updateDebugScreenshotTest

# Every subsequent run — compare against references
./gradlew validateDebugScreenshotTest

# Custom variant pattern
./gradlew update{Variant}ScreenshotTest
./gradlew validate{Variant}ScreenshotTest

# Scoped to a specific module
./gradlew :{module}:update{Variant}ScreenshotTest
./gradlew :{module}:validate{Variant}ScreenshotTest
```

---

*Based on the official Android documentation: https://developer.android.com/studio/preview/compose-screenshot-testing (last updated 2026-01-22)*
