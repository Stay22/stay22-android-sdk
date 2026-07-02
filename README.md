# Stay22 Android SDK

Native Android SDK for notification-driven accommodation discovery. When your app
captures a destination, dates, or a specific hotel, the SDK schedules a single,
well-timed local notification that opens a curated Stay22 booking experience. The
Android contract mirrors the iOS SDK public surface where platform differences allow it.

- Kotlin · minSdk 26 (Android 8.0)
- No GPS permission required
- You control enablement, notification permission, and user consent

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
    implementation("com.stay22:sdk:1.0.9")
}
```

- Ensure `minSdk` is 26 or newer in the app module's `defaultConfig`.
- The published POM declares the SDK's transitive dependencies, so they resolve
  automatically as long as `google()` and `mavenCentral()` are present.
- The `.aar` merges its own `AndroidManifest.xml`, which contributes the `INTERNET`
  and `POST_NOTIFICATIONS` permissions, a `booking` scheme `<queries>` entry, and an
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

Initialize once from `Application.onCreate()`. Android requires the `Application`
argument so the SDK can register lifecycle callbacks before activities start.

```kotlin
import android.app.Application
import com.stay22.sdk.Stay22
import com.stay22.sdk.TravelContext

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        Stay22.isEnabled = userHasOptedInToStay22Offers
        Stay22.initialize(application = this, aid = "your-partner-id")
    }
}

Stay22.requestNotificationPermission()

Stay22.setTravelContext(
    TravelContext(
        address = "Paris, France",
        checkinDate = "2026-03-20",
        checkoutDate = "2026-03-25"
    )
)
```

Register the `Application` subclass in the manifest with `android:name=".MyApp"` if it
is not already registered. Calling `initialize` more than once is safe; later calls
are ignored.

## Enablement And Consent

Recommended practices when enabling Stay22:

- consider surfacing opt-in language before enabling Stay22;
- do not require Stay22 or notification permission for core app functionality;
- provide an in-app opt-out control.

```kotlin
Stay22.isEnabled = userHasOptedInToStay22Offers
```

`isEnabled` is persisted and can be set before initialization. Setting it to `false`
clears pending Stay22 notification state and prevents new schedules.

## Notification Permission

```kotlin
val canShow = Stay22.hasNotificationPermission()
val currentStateAfterRequest = Stay22.requestNotificationPermission()
```

On Android 13 (API 33) and newer, `requestNotificationPermission()` asks from the
current resumed Activity when one is available. The platform returns the final user
choice asynchronously through the normal Android permission flow.

## Travel Context

Pass explicit travel context when your app knows the user's destination intent.

```kotlin
import com.stay22.sdk.TravelContext

Stay22.setTravelContext(
    TravelContext(
        address = "Paris, France",
        latitude = 48.8566,
        longitude = 2.3522,
        checkinDate = "2026-03-20",
        checkoutDate = "2026-03-25",
        hotelName = "Hotel Example"
    )
)
```

`TravelContext` fields match iOS:

| Field | Description |
|---|---|
| `address` | City, venue, hotel, or formatted destination. |
| `latitude` / `longitude` | Optional destination coordinates. |
| `checkinDate` / `checkoutDate` | Optional dates in `YYYY-MM-DD` format. |
| `hotelName` | Optional specific hotel name. |

Notes:

- `setTravelContext` merges non-empty fields, so destination and dates can arrive on
  different screens.
- Scheduling only uses explicit travel intent: address, coordinates, or hotel name.
- The IP/home city is used for suppression only; it is never used as a destination.

To clear explicit context:

```kotlin
Stay22.clearTravelContext()
// or
Stay22.advanced.clearTravelContext()
```

## Notification Configuration

```kotlin
import com.stay22.sdk.NotificationConfig
import com.stay22.sdk.NotificationInterruptionLevel

Stay22.notificationConfig = NotificationConfig(
    title = "Hotels in {destination}",
    subtitle = "{checkin} - {checkout}",
    message = "Tap to find the best deals nearby",
    categoryId = "stay22_destinations",
    threadId = "stay22",
    interruptionLevel = NotificationInterruptionLevel.active
)
```

Supported placeholders are `{destination}`, `{checkin}`, and `{checkout}`, which the
SDK fills from the current travel context at schedule time (`{checkin}`/`{checkout}`
resolve to empty when no dates are set). Unsupported tokens are left unchanged.

`NotificationConfig` fields match iOS:

| Field | Default |
|---|---|
| `title` | `"Hotels in {destination}"` |
| `subtitle` | `null` |
| `message` | `"Tap to find the best deals nearby"` |
| `categoryId` | `"stay22_destinations"` |
| `threadId` | `"stay22"` |
| `badge` | `null` |
| `launchImageName` | `null` |
| `targetContentIdentifier` | `null` |
| `interruptionLevel` | `null` |
| `relevanceScore` | `null` |
| `attachments` | `emptyList()` |
| `actions` | `emptyList()` |

Android-specific notification channel and small-icon selection are internal. The app
icon is used as the notification icon when available. For a reliable Android
notification small icon, provide a monochrome drawable in the host app and point
the SDK at it from the app manifest:

```xml
<application>
    <meta-data
        android:name="com.stay22.sdk.notification_small_icon"
        android:resource="@drawable/ic_stat_notification" />
</application>
```

Use a simple white/transparent status-bar style drawable rather than the adaptive
launcher icon.

## Advanced Namespace

```kotlin
Stay22.advanced.setEventHandler { event ->
    when (event) {
        is Stay22Event.NotificationScheduled -> Unit
        is Stay22Event.NotificationClicked -> Unit
        else -> Unit
    }
}

Stay22.advanced.scheduleNotification()
```

`scheduleNotification()` runs the same scheduling pipeline as automatic background
scheduling, using the current travel context and server-configured delay. The SDK
keeps at most one pending Stay22 notification; a changed travel context replaces it,
while the same context keeps the existing timer.

## Events

```kotlin
Stay22.advanced.setEventListener { event ->
    when (event) {
        is Stay22Event.LocationUpdated -> Unit
        is Stay22Event.NotificationScheduled -> Unit
        is Stay22Event.NotificationShown -> Unit
        is Stay22Event.NotificationClicked -> Unit
        is Stay22Event.NotificationBlocked -> Unit
        is Stay22Event.NotificationSkipped -> Unit
        is Stay22Event.NotificationCancelled -> Unit
        is Stay22Event.EnabledChanged -> Unit
        Stay22Event.TravelContextCleared -> Unit
    }
}
```

## Testing

For local and staging validation only:

```kotlin
Stay22.testing.force = true
Stay22.testing.showLogs = true
```

Force mode uses a 5-second notification delay and skips local gates so the same flow
can be tested repeatedly. Do not ship production builds with force mode on.

## Behavior Summary

| Condition | Behavior |
|---|---|
| SDK disabled | Pending notification state is cleared and no new notification is scheduled. |
| Missing destination | Scheduling is skipped. |
| App foregrounded | Automatic scheduling waits for background; refreshed context can update an existing pending notification, and advanced scheduling can run foreground. |
| Home destination | Scheduling is skipped when the destination matches the server-detected home city. |
| Partner config disabled | Pending notification state is cleared. |
| Duplicate context | Existing pending notification timer is kept. |
| Notification permission denied | Scheduling/showing is blocked and an event is emitted. |
| Local cooldown | Scheduling is skipped. |

The SDK keeps at most one pending Stay22 notification at a time. Server partner config
and Nova decisions control remote enablement, delay, distance, cooldown, and frequency.

## Privacy

The SDK does not request device GPS permission and is not used for cross-app tracking.
It may send the partner ID, explicit destination context, and anonymous SDK
interaction events to Stay22. The host app is responsible for consent, privacy policy
disclosures, and app-store privacy answers.

## Public API

| API | Purpose |
|---|---|
| `Stay22.initialize(application, aid)` | Start the SDK from `Application.onCreate()`. Android requires the `Application` for lifecycle callbacks. |
| `Stay22.isEnabled` | Persisted enable/disable flag for consent or settings. Can be set before initialization. |
| `Stay22.isInitialized` | Whether initialization completed. |
| `Stay22.notificationConfig` | Notification text and notification metadata. |
| `Stay22.campaignId` | Optional attribution value added to generated URLs. |
| `Stay22.hasNotificationPermission()` | Current Android notification permission state. |
| `Stay22.requestNotificationPermission()` | Request Android 13+ notification permission from the resumed Activity when available. |
| `Stay22.setTravelContext(...)` | Provide destination context. |
| `Stay22.clearTravelContext()` | Clear explicit travel context and pending notification state. |
| `Stay22.advanced.setEventListener(...)` | Register an event listener. |
| `Stay22.advanced.setEventHandler(...)` | Register a closure-style event handler. |
| `Stay22.advanced.clearTravelContext()` | Advanced namespace equivalent of clearing travel context. |
| `Stay22.advanced.scheduleNotification()` | Run the normal scheduling pipeline for the current context. |
| `Stay22.testing.force` | Testing-only force flag. |
| `Stay22.testing.showLogs` | Testing-only SDK logging flag. |

## License

This SDK is proprietary and closed source. © 2026 Stay22 Technologies Inc. All
Rights Reserved. Use is governed by the [LICENSE](LICENSE) and any separate written
agreement with Stay22. Contact your Stay22 representative for terms.

## Support

Questions? Contact your Stay22 representative or visit
[stay22.com](https://www.stay22.com).
