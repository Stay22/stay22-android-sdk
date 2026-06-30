# Stay22 Android SDK - Integration Prompt

Use this prompt when integrating the Stay22 Android SDK into an Android app. The SDK
is distributed as a Maven artifact (`com.stay22:sdk`, recommended) or as a manually
embedded `stay22-sdk.aar`.

Provide the integrator with:

- the Maven coordinate `com.stay22:sdk:<VERSION>` and the Stay22 Maven repository URL
  (or `stay22-sdk.aar` from a release, for manual integration)
- this prompt
- the Stay22 partner ID (`aid`)

## Prompt

````text
Integrate the Stay22 Android SDK into this Android app.

Context:
- Maven coordinate: com.stay22:sdk:<VERSION>
- Minimum SDK: 26 (Android 8.0)
- Partner ID: <AID>

Tasks:

1. Confirm this is an Android app target.
   Look for `build.gradle` / `build.gradle.kts`, an `AndroidManifest.xml`, and
   Kotlin/Java sources.

2. Add the SDK. Prefer the Maven coordinate.

   Maven (recommended):
   - Add the Stay22 Maven repository alongside Google and Maven Central (e.g. in
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

   - Add the dependency in the app module's `build.gradle.kts`:

     ```kotlin
     dependencies {
         implementation("com.stay22:sdk:<VERSION>")
     }
     ```

   - The published POM declares the SDK's transitive dependencies, so they resolve
     automatically as long as `google()` and `mavenCentral()` are present.

   Manual (AAR), only if not using the Maven repository:
   - Place `stay22-sdk.aar` in the app module's `libs/` directory (e.g. `app/libs/`).
     A raw `.aar` carries no POM, so declare the transitive dependencies explicitly:

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

   - Ensure `minSdk` is 26 or newer in the app module's `defaultConfig`.
   - The `.aar` merges its own `AndroidManifest.xml`, which contributes the
     `INTERNET` and `POST_NOTIFICATIONS` permissions, a `booking` scheme `<queries>`
     entry, and an internal notification-tap activity. Do not redeclare these in the
     host manifest.

3. Initialize the SDK once, early in app startup.
   Initialize from `Application.onCreate()`.

   ```kotlin
   import android.app.Application
   import com.stay22.sdk.Stay22

   class MyApp : Application() {
       override fun onCreate() {
           super.onCreate()
           Stay22.isEnabled = userHasOptedInToStay22Offers
           Stay22.initialize(application = this, aid = "<AID>")
       }
   }
   ```

   Register the `Application` subclass in the manifest with
   `android:name=".MyApp"` if it is not already registered.

4. Add consent and notification permission handling.
   Treat Stay22 notifications as promotional. Do not make them required for the app
   to function.

   When enabling Stay22:
   - consider surfacing opt-in language before enabling Stay22;
   - store the user's choice;
   - provide an in-app opt-out control.

   Wire opt-in and opt-out to:

   ```kotlin
   Stay22.isEnabled = userHasOptedInToStay22Offers
   ```

   When the user opts in, request notification permission at an appropriate
   moment. On Android 13+ (API 33) this launches the system `POST_NOTIFICATIONS`
   prompt from the current resumed Activity; the call returns the current
   permission state and the grant/deny result is delivered by the platform
   asynchronously.

   ```kotlin
   Stay22.requestNotificationPermission()
   ```

   To check permission without prompting:

   ```kotlin
   val canNotify = Stay22.hasNotificationPermission()
   ```

5. Capture explicit travel intent.
   Search the app for screens, view models, routes, and analytics events where
   the app knows a destination, city, venue, hotel, coordinates, booking,
   itinerary, event, search result, map place, or travel date.

   Add `Stay22.setTravelContext(...)` where the app first has reliable intent:

   ```kotlin
   import com.stay22.sdk.TravelContext

   Stay22.setTravelContext(
       TravelContext(
           address = "Paris, France",
           latitude = 48.8566,
           longitude = 2.3522,
           checkinDate = "2026-03-20",
           checkoutDate = "2026-03-25"
       )
   )
   ```

   Address-only and coordinates-only are valid:

   ```kotlin
   Stay22.setTravelContext(TravelContext(address = "Paris, France"))
   Stay22.setTravelContext(TravelContext(latitude = 48.8566, longitude = 2.3522))
   ```

   Use `hotelName` only when the app knows a specific hotel:

   ```kotlin
   Stay22.setTravelContext(
       TravelContext(address = "Paris, France", hotelName = "Hotel Example")
   )
   ```

   Date format: `YYYY-MM-DD`.

   Important:
   - Notifications are scheduled from explicit travel intent only: address,
     coordinates, or hotel name.
   - The user's IP/home city is never used as a destination.
   - `setTravelContext` merges non-empty fields, so destination and dates can be
     provided across multiple screens.
   - Call `Stay22.clearTravelContext()` when a travel/search flow resets, the
     user signs out, or consent is revoked.

6. Optional: customize notification copy.

   ```kotlin
   import com.stay22.sdk.NotificationConfig

   Stay22.notificationConfig = NotificationConfig(
       title = "Hotels in {destination}",
       message = "Find stays for {checkin}-{checkout}."
   )
   ```

   Supported placeholders: `{destination}`, `{checkin}`, `{checkout}`. The SDK
   fills them from the current travel context at schedule time; `{checkin}` and
   `{checkout}` resolve to empty when no dates are set.

   For a reliable notification small icon, add a monochrome drawable to the host
   app and reference it from the app manifest:

   ```xml
   <application>
       <meta-data
           android:name="com.stay22.sdk.notification_small_icon"
           android:resource="@drawable/ic_stat_notification" />
   </application>
   ```

   Do not use the adaptive launcher icon for this resource.

7. Optional: add attribution.

   ```kotlin
   Stay22.campaignId = "android-pilot-2026" // optional
   ```

8. Optional: observe SDK events for QA and analytics.

   ```kotlin
   import com.stay22.sdk.Stay22Event

   Stay22.advanced.setEventHandler { event ->
       when (event) {
           is Stay22Event.LocationUpdated ->
               Log.d("Stay22", "context: ${event.destination}")
           is Stay22Event.NotificationScheduled ->
               Log.d("Stay22", "scheduled: ${event.destination}, delay=${event.delay}s")
           is Stay22Event.NotificationClicked ->
               Log.d("Stay22", "clicked: ${event.destination}, ${event.url}")
           is Stay22Event.NotificationSkipped ->
               Log.d("Stay22", "skipped: ${event.reason}")
           is Stay22Event.NotificationBlocked ->
               Log.d("Stay22", "blocked: ${event.reason}")
           is Stay22Event.NotificationCancelled ->
               Log.d("Stay22", "cancelled: ${event.reason}")
           else -> Unit
       }
   }
   ```

   Other emitted events include `NotificationShown`, `EnabledChanged`, and
   `TravelContextCleared`.

9. Optional: use manual scheduling only when the host app needs full timing
   control.

   ```kotlin
   Stay22.advanced.scheduleNotification()
   ```

   Automatic scheduling normally runs when fresh travel context exists and the
   app backgrounds. The notification delay comes from Stay22 partner config.

10. Development testing only:

   ```kotlin
   Stay22.testing.force = true
   Stay22.testing.showLogs = true
   ```

   Force mode uses a 5-second local notification delay and skips local gates for
   repeatable QA. Never ship production builds with force mode enabled.

Public API to use:
- `Stay22.initialize(application, aid)`
- `Stay22.isEnabled`
- `Stay22.requestNotificationPermission()`
- `Stay22.hasNotificationPermission()`
- `Stay22.setTravelContext(...)`
- `Stay22.clearTravelContext()`
- `Stay22.notificationConfig`
- `Stay22.campaignId`
- `Stay22.advanced.setEventHandler(...)`
- `Stay22.advanced.setEventListener(...)`
- `Stay22.advanced.scheduleNotification()`
- `Stay22.testing.force`
- `Stay22.testing.showLogs`
- `TravelContext(address, latitude, longitude, checkinDate, checkoutDate, hotelName)`
- `NotificationConfig(title, message, ...)`

Do not call non-public or internal APIs such as `setLocation`, `setKeywords`,
`Stay22.force`, `Stay22.showLogs`, or `scheduleNotification(delay)`.
````
