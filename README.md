# Stay22 Android SDK

Native Android SDK for notification-driven accommodation discovery. When your app
captures a destination, dates or a hotel, the SDK can schedule one local notification
that opens a Stay22 booking page, or an installed booking app when one can open the offer.

- Kotlin · minSdk 24 (Android 7.0)
- No GPS permission required
- You control enablement, notification permission and user consent

## Installation

The SDK is published as a Maven artifact (`com.stay22:sdk`) to a Maven repository
hosted in this repository.

Add the Stay22 Maven repository alongside Google and Maven Central (e.g. in
`settings.gradle.kts`):

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = uri("https://raw.githubusercontent.com/Stay22/stay22-android-sdk/main/maven") }
    }
}
```

Then add the dependency in the app module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.stay22:sdk:3.1.1")
}
```

- Ensure `minSdk` is 24 or newer in the app module's `defaultConfig`.
- The published POM declares the SDK's transitive dependencies, so they resolve
  automatically as long as `google()` and `mavenCentral()` are present.
- The `.aar` merges its own `AndroidManifest.xml`, which contributes the `INTERNET`
  and `POST_NOTIFICATIONS` permissions, `<queries>` entries for the supported booking apps, and an
  internal notification-tap activity. Do not redeclare these in the host manifest.

### Manual (AAR)

If you prefer not to use the Maven repository, download `stay22-sdk.aar`, place it in
the app module's `libs/` directory (e.g. `app/libs/`), and depend on it directly. A
raw `.aar` carries no POM, so declare the transitive dependencies explicitly:

```kotlin
dependencies {
    implementation(files("libs/stay22-sdk.aar"))

    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.work:work-runtime-ktx:2.9.0")
    implementation("androidx.lifecycle:lifecycle-process:2.7.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
}
```

## Quick Start

Initialize from `Application.onCreate()`, with your stored consent choice, and register
that Application class in your manifest.

```kotlin
import android.app.Application
import com.stay22.sdk.*

fun startStay22(application: Application, userOptedIn: Boolean) {
    Stay22.setEnabled(application, userOptedIn)
    Stay22.initialize(application = application, aid = "your-partner-id")
}

fun submitTrip(): NotificationResult {
    // Illustrative app-chosen validity bounds, not SDK defaults.
    val now = System.currentTimeMillis()
    val request = NotificationRequest(
        trip = TravelContext(address = "Paris, France"),
        timing = NotificationTiming.After(30 * 60 * 1000L),
        expiresAtMs = now + 12 * 60 * 60 * 1000L,
        actionExpiresAtMs = now + 24 * 60 * 60 * 1000L,
    )
    return Stay22.notifications.ensure(request)
}
```

Submit each trip once and keep the request to retry it; a submitted result can still be
rejected during preparation. The two expiry times are required unless your Stay22 account
sets a validity policy.

Ask for notification permission from a resumed Activity. The callback reports the
user's answer:

```kotlin
Stay22.requestNotificationPermission { granted -> /* ... */ }
```

To keep the SDK off until the user opts in, add an application `meta-data` entry
`com.stay22.sdk.RequiresConsent` with `android:value="true"`, then call
`Stay22.setEnabled(application, true)` after consent. Otherwise the SDK starts enabled.

## Verify

In an internal build, request a test notification that skips the normal delay and
cooldowns (consent and permission still apply):

```kotlin
Stay22.testing.showLogs = true
Stay22.testing.scheduleNotification(TravelContext(address = "Paris, France"))
```

Keep test calls and logging out of public builds.

Upgrading from 1.x: the breaking changes and what to call instead are in this
repository's [Releases](https://github.com/Stay22/stay22-android-sdk/releases).

## License

This SDK is proprietary and closed source. © 2026 Stay22 Technologies Inc. All
Rights Reserved. Use is governed by the [LICENSE](LICENSE) and any separate written
agreement with Stay22. Contact your Stay22 representative for terms.

## Support

Full guide, privacy details and changelog: https://dev.stay22.com/docs/mobile-sdk.
Questions? Contact your Stay22 representative.
