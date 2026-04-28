# Tripwire Android SDK

Tripwire is a native Android SDK for collecting encrypted device, app,
runtime, and behavioral signals that help your backend make risk decisions for
signup, login, checkout, account recovery, and other sensitive flows.

The public Android package is distributed through Maven Central as protected
binary AAR artifacts. Source code, private symbol mappings, native symbols,
and internal build artifacts are retained privately.

## Packages

| Artifact | Required | Description |
| --- | --- | --- |
| `com.tripwirejs:tripwire-android` | Yes | Core Tripwire Android SDK. |
| `com.tripwirejs:tripwire-android-gms` | No | Optional Google ecosystem helpers for apps that already ship Google Play Services. |

## Requirements

- Android 6.0 Marshmallow / API 23+
- Kotlin 1.9+
- Android Gradle Plugin 8.2+
- Java 17 toolchain
- A Tripwire publishable key, starting with `pk_live_` or `pk_test_`

## Installation

Add Maven Central and Google to dependency resolution:

```kotlin
// settings.gradle.kts
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
```

Add the SDK dependency:

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.tripwirejs:tripwire-android:1.0.0")
}
```

If your app already uses Google Play Services and wants the optional Google
continuity helpers, add:

```kotlin
dependencies {
    implementation("com.tripwirejs:tripwire-android-gms:1.0.0")
}
```

## Quick Start

Configure Tripwire once from your `Application`:

```kotlin
import android.app.Application
import com.tripwirejs.tripwire.TripwireClient
import com.tripwirejs.tripwire.TripwireConfiguration

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        TripwireClient.configure(
            context = this,
            configuration = TripwireConfiguration(
                publishableKey = "pk_live_your_publishable_key",
            ),
        )
    }
}
```

When the user performs a sensitive action, request a sealed session handoff
and send it to your backend with the rest of the action payload:

```kotlin
import androidx.lifecycle.lifecycleScope
import com.tripwirejs.tripwire.TripwireClient
import kotlinx.coroutines.launch

fun submitSignup() {
    lifecycleScope.launch {
        val handoff = TripwireClient.getSession()

        api.signup(
            SignupRequest(
                email = email,
                password = password,
                tripwireSessionId = handoff.sessionId,
                tripwireSealedToken = handoff.sealedToken,
            ),
        )
    }
}
```

Your server verifies the sealed token with Tripwire and decides whether to
allow, challenge, review, or deny the action. Do not make trust decisions in
the app itself.

## Configuration

```kotlin
val config = TripwireConfiguration(
    publishableKey = "pk_live_your_publishable_key",
    enableBehavioralSignals = true,
    enableHiddenWebView = true,
    enableCloudIdentifier = false,
    apiEndpoint = "https://api.tripwirejs.com",
)

TripwireClient.configure(context, config)
```

| Option | Default | Description |
| --- | --- | --- |
| `publishableKey` | Required | Your client-side Tripwire key. Must start with `pk_live_` or `pk_test_`. Secret keys are rejected. |
| `enableBehavioralSignals` | `true` | Captures native lifecycle, navigation, viewport, scroll, form, clipboard, selection, motion, and touch behavior. |
| `enableHiddenWebView` | `true` | Enables native-to-web handoff for Tripwire-protected `WebView` content. |
| `enableCloudIdentifier` | `false` | Enables an optional cloud continuity hint when the optional GMS helper is present and available. |
| `apiEndpoint` | Production API | Override only for development or private deployments. |

Runtime integrity and anti-tamper collection is always enabled. It is part of
the baseline SDK behavior rather than an app-side integration option.

## Session Handoff

`getSession()` returns a `SessionHandoff`:

```kotlin
data class SessionHandoff(
    val sessionId: String,
    val sealedToken: String,
)
```

- `sessionId` identifies the Tripwire session.
- `sealedToken` is the server-verifiable handoff token.
- The SDK does not expose scores, verdicts, visitor IDs, or risk decisions to
  the device.

Call `getSession()` immediately before the sensitive action. Avoid calling it
only at app launch, because Tripwire is most useful when it includes the
behavior leading up to the action.

## Backend Verification

Send `sessionId` and `sealedToken` to your own backend. Your backend should
verify the token with Tripwire, then apply your product policy.

Example request shape:

```json
{
  "email": "user@example.com",
  "tripwire": {
    "sessionId": "sess_...",
    "sealedToken": "..."
  }
}
```

Recommended policy:

- Keep Tripwire secret keys on your server only.
- Treat the mobile SDK as an evidence collector, not a policy engine.
- Make allow/challenge/deny decisions server-side.
- Decide explicitly whether each flow should fail open or fail closed if
  Tripwire is unavailable.

## Behavioral Capture

When `enableBehavioralSignals` is enabled, Tripwire starts native behavioral
capture automatically. The SDK avoids collecting raw form values, raw hints,
or raw view identifiers.

Touch dynamics can be attached manually for screens where gesture behavior is
important:

```kotlin
import com.tripwirejs.tripwire.TripwireClient

override fun onResume() {
    super.onResume()
    TripwireClient.observeTouches(rootView, contextId = "checkout")
}
```

The observer wraps an existing `View.OnTouchListener` when present, so your
existing gestures and scroll handlers continue to work.

## WebView Correlation

If your app embeds Tripwire-protected web content, attach your `WebView` so
the browser SDK can reuse the native session:

```kotlin
import android.webkit.WebView
import com.tripwirejs.tripwire.TripwireClient

val webView = WebView(context)
TripwireClient.attach(webView)
webView.loadUrl("https://app.example.com/signup")
```

This is useful for hybrid apps where native and web surfaces participate in
the same user flow.

## Optional GMS Helpers

The `tripwire-android-gms` artifact is optional. Add it only if your app
already depends on Google Play Services and wants the additional Google
ecosystem continuity and attestation support.

Apps without the helper still use the core SDK normally.

## Permissions and Privacy

The SDK does not require dangerous Android permissions. Your app needs
`android.permission.INTERNET` to talk to the Tripwire API.

Tripwire collects encrypted signals that help distinguish trustworthy,
automated, tampered, replayed, or anomalous sessions. Signal categories
include:

- Device and OS characteristics
- App and installation characteristics
- Network and runtime environment shape
- Runtime integrity and tamper-resistance signals
- Native behavioral timing and interaction patterns
- Optional Google ecosystem continuity hints
- Android platform attestation support facts

The SDK is designed to avoid collecting raw form input values. Sensitive
values should stay in your app and backend; Tripwire receives encrypted
session evidence for server-side verification.

## API Reference

| API | Description |
| --- | --- |
| `TripwireClient.configure(context, configuration)` | Validates configuration and starts collection. |
| `TripwireClient.getSession()` | Flushes a bounded batch and returns a server-verifiable handoff. |
| `TripwireClient.waitForFingerprint()` | Waits for the durable fingerprint to be ready. Optional for most integrations. |
| `TripwireClient.observeTouches(view, contextId)` | Manually attaches touch dynamics to a view. |
| `TripwireClient.attach(webView)` | Connects a `WebView` to the native Tripwire session. |
| `TripwireClient.destroy()` | Stops collection and releases resources. Mostly useful in tests. |

## Error Handling

Tripwire throws `TripwireError` subclasses for SDK-defined failures. The
errors carry stable codes and retryability semantics.

Recommended fallback:

```kotlin
suspend fun tripwireHandoffOrNull(): SessionHandoff? {
    return try {
        TripwireClient.getSession()
    } catch (error: TripwireError) {
        if (error.retryable) {
            runCatching { TripwireClient.getSession() }.getOrNull()
        } else {
            null
        }
    }
}
```

If Tripwire fails, choose a product-appropriate policy. Many apps log the
error and continue without a handoff; high-risk flows may choose to challenge
or fail closed.

## Security and Distribution

The public repository contains package metadata and documentation only. It is
not a source mirror.

The public Maven artifacts include protected binary AARs plus placeholder
sources and javadocs jars for Maven Central compliance. Private R8 mappings,
native symbols, and debug artifacts are retained by Tripwire.

## Versioning

Tripwire follows semantic versioning for public mobile SDK packages.

```kotlin
implementation("com.tripwirejs:tripwire-android:1.0.0")
implementation("com.tripwirejs:tripwire-android-gms:1.0.0")
```

Pin exact versions in production builds and upgrade intentionally.

## Support

For SDK access, integration help, or production verification setup, contact
Tripwire support.
