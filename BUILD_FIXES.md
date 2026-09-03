# BuildExpress — complete project, build errors fixed

This is your **entire app**, with the new UI already merged in and the build
configuration repaired. Open the folder in Android Studio and press Run.

---

## The #1 most likely cause of your failure: **Java version**

Your project uses **Android Gradle Plugin 8.5.1**, which **requires JDK 17**.
If Android Studio is on JDK 11 or 8 you get errors like:

```
Android Gradle plugin requires Java 17 to run. You are currently using Java 11.
Unsupported class file major version 61
com/android/build/gradle/... UnsupportedClassVersionError
```

### Fix
**File → Settings → Build, Execution, Deployment → Build Tools → Gradle**
→ set **Gradle JDK** to **17** (pick "Download JDK…" → version 17 if absent)
→ OK → **File → Sync Project with Gradle Files**

macOS: *Android Studio → Settings →* same path.

---

## What I actually found and fixed

### 1. `package` in AndroidManifest — HARD ERROR on AGP 8
```
Error: package="com.build.buildexpress" found in source AndroidManifest.xml.
Setting the namespace via the package attribute is not supported.
```
AGP 8 removed it; `namespace` in `build.gradle` replaces it.

**Fixed:** removed the attribute. `namespace 'com.build.buildexpress'` was already
in `app/build.gradle`, so nothing else changes.

### 2. Missing `compileOptions` — lambdas fail
Your code uses lambdas (`v -> dialog.dismiss()`) in **21 places**, but the build
file never declared Java 8 source compatibility.

**Fixed:** added
```gradle
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
```

### 3. Four dependencies used but never declared
Verified by scanning imports and layout XML:

| Library | Where used | Was in gradle? |
|---|---|---|
| `androidx.viewpager2` | 2 Java files, 2 layouts | **no** |
| `androidx.drawerlayout` | 1 Java file, 1 layout | **no** |
| `androidx.cardview` | **37 layouts** | **no** |
| `androidx.core` | ContextCompat, FileProvider | **no** (transitive only) |

These produce `error: cannot find symbol class ViewPager2` /
`Didn't find class androidx.cardview.widget.CardView`.

**Fixed:** all four added explicitly.

### 4. Gradle heap — the original OOM
The `java_pid19016.hprof` you hit earlier was the Gradle JVM dying. Your
`gradle.properties` had **no heap setting at all**.

**Fixed:**
```properties
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m -Dfile.encoding=UTF-8
```
Lower `4096m` to `2048m` if your machine has 8 GB or less.

### 5. `android.enableJetifier=true` — unnecessary
Nothing here uses old support-library artifacts. Jetifier rewrites every
dependency on every build and slows it dramatically.

**Fixed:** set to `false`.

### 6. Missing proguard file
`buildTypes.release` referenced no proguard config. Added
`app/proguard-rules.pro` with keep-rules for Room, Glide, Gson and your
Firebase model classes.

---

## Verified before packaging

| Check | Result |
|---|---|
| All XML well-formed | **162/162** res files + manifest |
| Every `R.*` in Java resolves | **903/903** references, 79 Java files |
| Resource cross-references | all resolve |
| Duplicate resource names | none |
| Layout IDs after colour migration | **596** intact |
| Colour migration applied | 537 replacements, 3 alpha overlays left |
| Gradle wrapper jar | present (43 KB) |

I could not run a real `./gradlew assembleDebug` — this sandbox has no Android
SDK and only JDK 11. Everything above is static verification, which catches
resource and reference errors but not dependency-resolution problems.

---

## Build it

```bash
cd BuildExpress
./gradlew clean assembleDebug          # Windows: gradlew.bat clean assembleDebug
```

APK appears at:
```
app/build/outputs/apk/debug/BuildExpress-debug.apk
```

Install to a connected phone:
```bash
./gradlew installDebug
```

Or in Android Studio: **File → Open** this folder → wait for sync → **▶ Run**.

---

## Things you must supply

### `local.properties`
Not included (it's machine-specific and was leaking your paths on GitHub).
Android Studio creates it automatically on first open. To make it manually:

```properties
sdk.dir=/Users/you/Library/Android/sdk       # macOS
sdk.dir=C\:\\Users\\you\\AppData\\Local\\Android\\Sdk   # Windows
sdk.dir=/home/you/Android/Sdk                # Linux
```

### `google-services.json`
Your existing one **is** included at `app/google-services.json`. If Firebase
complains, re-download it from the Firebase console (Project settings → Your
apps → Android) and replace it.

---

## If it still fails

Run this and send me the output — it prints the real error instead of the
summary Android Studio shows:

```bash
./gradlew clean assembleDebug --stacktrace 2>&1 | tail -60
```

### Common remaining errors

**`SDK location not found`**
→ Open the project once in Android Studio; it writes `local.properties`.

**`Failed to install the following SDK components: platforms;android-34`**
→ Tools → SDK Manager → tick **Android 14 (API 34)** → Apply.

**`Could not resolve com.google.firebase:...`**
→ No internet, or a corporate proxy. Gradle must reach `google()` and
`mavenCentral()`.

**`Duplicate class ...`**
→ Tell me the class name; it means two libraries ship the same code.

**`Execution failed for task ':app:processDebugGoogleServices'`**
→ The `applicationId` in `build.gradle` must exactly match `package_name` in
`google-services.json`. Both should be `com.build.buildexpress`.

---

## What's in this zip

```
BuildExpress/
├── app/
│   ├── build.gradle              <- FIXED (deps + compileOptions)
│   ├── google-services.json
│   ├── proguard-rules.pro        <- NEW
│   └── src/main/
│       ├── AndroidManifest.xml   <- FIXED (package attr removed)
│       ├── assets/
│       ├── java/                 79 files, unchanged
│       └── res/                  87 layouts + new design system + app icon
├── build.gradle
├── settings.gradle
├── gradle.properties             <- FIXED (heap, jetifier)
├── gradle/wrapper/
├── gradlew / gradlew.bat
├── firestore.rules
├── .gitignore                    <- NEW
└── BUILD_FIXES.md                <- this file
```

Excluded on purpose: `.git`, `build/`, `.gradle/`, `.idea/`, `gradle-8.7/`,
`.artifacts/`, `local.properties`, and the 487 MB `.hprof` heap dumps.

---

## Still open — please check

**`firestore.rules` is public in your GitHub repo.** If it contains
`allow read, write: if true;` anyone can read and wipe your database. That is
more urgent than any build error.
