# Foil Android SDK

Foil is a native Android SDK for collecting encrypted device, app,
runtime, and behavioral signals that help your backend make risk decisions for
signup, login, checkout, account recovery, and other sensitive flows.

The public Android package is distributed through Maven Central as protected
binary AAR artifacts. Source code, private symbol mappings, native symbols,
and internal build artifacts are retained privately.

## Packages

| Artifact | Required | Description |
| --- | --- | --- |
| `com.usefoil:foil-android` | Yes | Core Foil Android SDK. |
| `com.usefoil:foil-android-gms` | No | Optional Google ecosystem helpers for apps that already ship Google Play Services. |

## What's New in 1.2.2

- Expanded native signal coverage for device, app, runtime, network, and
  integrity posture.
- Improved native behavioral collection for form, action, lifecycle, motion,
  scroll, WebView, and touch evidence while keeping raw user-entered values out
  of the payload.
- Better parity with the iOS and web SDKs so native and hybrid sessions can be
  interpreted consistently by Foil's server-side scoring pipeline.
- Additional integration diagnostics and companion-app polish for local and
  production validation.

## Requirements

- Android 6.0 Marshmallow / API 23+
- Kotlin 1.9+
- Android Gradle Plugin 8.2+
- Java 17 toolchain
- A Foil publishable key, starting with `pk_live_` or `pk_test_`

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
    implementation("com.usefoil:foil-android:1.2.2")
}
```

If your app already uses Google Play Services and wants the optional Google
continuity helpers, add:

```kotlin
dependencies {
    implementation("com.usefoil:foil-android-gms:1.2.2")
}
```

## Quick Start

Configure Foil once from your `Application`:

```kotlin
import android.app.Application
import com.usefoil.foil.FoilClient
import com.usefoil.foil.FoilConfiguration

class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()

        FoilClient.configure(
            context = this,
            configuration = FoilConfiguration(
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
import com.usefoil.foil.FoilClient
import kotlinx.coroutines.launch

fun submitSignup() {
    lifecycleScope.launch {
        val handoff = FoilClient.getSession()

        api.signup(
            SignupRequest(
                email = email,
                password = password,
                foilSessionId = handoff.sessionId,
                foilSealedToken = handoff.sealedToken,
            ),
        )
    }
}
```

Your server verifies the sealed token with Foil and decides whether to
allow, challenge, review, or deny the action. Do not make trust decisions in
the app itself.

## Configuration

```kotlin
val config = FoilConfiguration(
    publishableKey = "pk_live_your_publishable_key",
    enableBehavioralSignals = true,
    enableHiddenWebView = true,
    enableCloudIdentifier = false,
    enableAutoAttachTouches = false,
    lockAgainstExternalDebuggers = false,
    apiEndpoint = "https://api.usefoil.com",
)

FoilClient.configure(context, config)
```

| Option | Default | Description |
| --- | --- | --- |
| `publishableKey` | Required | Your client-side Foil key. Must start with `pk_live_` or `pk_test_`. Secret keys are rejected. |
| `enableBehavioralSignals` | `true` | Captures native lifecycle, navigation, viewport, scroll, form, clipboard, selection, motion, and touch behavior. |
| `enableHiddenWebView` | `true` | Enables native-to-web handoff for Foil-protected `WebView` content. |
| `enableCloudIdentifier` | `false` | Enables an optional cloud continuity hint when the optional GMS helper is present and available. |
| `enableAutoAttachTouches` | `false` | Automatically attaches touch dynamics to resumed activities. Leave off if you attach touches manually. |
| `lockAgainstExternalDebuggers` | `false` | Optional production hardening that can block debugger/profiler attachment while the app is running. Test carefully before enabling. |
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

- `sessionId` identifies the Foil session.
- `sealedToken` is the server-verifiable handoff token.
- The SDK does not expose scores, verdicts, visitor IDs, or risk decisions to
  the device.

Call `getSession()` immediately before the sensitive action. Avoid calling it
only at app launch, because Foil is most useful when it includes the
behavior leading up to the action.

## Backend Verification

Send `sessionId` and `sealedToken` to your own backend. Your backend should
verify the token with Foil, then apply your product policy.

Example request shape:

```json
{
  "email": "user@example.com",
  "foil": {
    "sessionId": "sess_...",
    "sealedToken": "..."
  }
}
```

Recommended policy:

- Keep Foil secret keys on your server only.
- Treat the mobile SDK as an evidence collector, not a policy engine.
- Make allow/challenge/deny decisions server-side.
- Decide explicitly whether each flow should fail open or fail closed if
  Foil is unavailable.

## Behavioral Capture

When `enableBehavioralSignals` is enabled, Foil starts native behavioral
capture automatically. The SDK avoids collecting raw form values, raw hints,
or raw view identifiers.

Touch dynamics can be attached manually for screens where gesture behavior is
important:

```kotlin
import com.usefoil.foil.FoilClient

override fun onResume() {
    super.onResume()
    FoilClient.observeTouches(rootView, contextId = "checkout")
}
```

The observer wraps an existing `View.OnTouchListener` when present, so your
existing gestures and scroll handlers continue to work.

## WebView Correlation

If your app embeds Foil-protected web content, attach your `WebView` so
the browser SDK can reuse the native session:

```kotlin
import android.webkit.WebView
import com.usefoil.foil.FoilClient

val webView = WebView(context)
FoilClient.attach(webView)
webView.loadUrl("https://app.example.com/signup")
```

This is useful for hybrid apps where native and web surfaces participate in
the same user flow.

## Optional GMS Helpers

The `foil-android-gms` artifact is optional. Add it only if your app
already depends on Google Play Services and wants the additional Google
ecosystem continuity and attestation support.

Apps without the helper still use the core SDK normally.

## Permissions and Privacy

The SDK does not require dangerous Android permissions. Your app needs
`android.permission.INTERNET` to talk to the Foil API.

Foil collects encrypted signals that help distinguish trustworthy,
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
values should stay in your app and backend; Foil receives encrypted
session evidence for server-side verification.

## API Reference

| API | Description |
| --- | --- |
| `FoilClient.configure(context, configuration)` | Validates configuration and starts collection. |
| `FoilClient.getSession()` | Flushes a bounded batch and returns a server-verifiable handoff. |
| `FoilClient.waitForFingerprint()` | Waits for the durable fingerprint to be ready. Optional for most integrations. |
| `FoilClient.pauseCollection()` | Temporarily pauses active collection without destroying SDK state. |
| `FoilClient.resumeCollection()` | Resumes active collection after a pause. |
| `FoilClient.observeTouches(view, contextId)` | Manually attaches touch dynamics to a view. |
| `FoilClient.attach(webView)` | Connects a `WebView` to the native Foil session. |
| `FoilClient.resetLocalState()` | Clears local SDK session state for development and test flows. |
| `FoilClient.destroy()` | Stops collection and releases resources. Mostly useful in tests. |

## Error Handling

Foil throws `FoilError` subclasses for SDK-defined failures. The
errors carry stable codes and retryability semantics.

Recommended fallback:

```kotlin
suspend fun foilHandoffOrNull(): SessionHandoff? {
    return try {
        FoilClient.getSession()
    } catch (error: FoilError) {
        if (error.retryable) {
            runCatching { FoilClient.getSession() }.getOrNull()
        } else {
            null
        }
    }
}
```

If Foil fails, choose a product-appropriate policy. Many apps log the
error and continue without a handoff; high-risk flows may choose to challenge
or fail closed.

## Security and Distribution

The public repository contains package metadata and documentation only. It is
not a source mirror.

The public Maven artifacts include protected binary AARs plus placeholder
sources and javadocs jars for Maven Central compliance. Private mappings,
native symbols, and debug artifacts are retained by Foil.

## Versioning

Foil follows semantic versioning for public mobile SDK packages.

```kotlin
implementation("com.usefoil:foil-android:1.2.2")
implementation("com.usefoil:foil-android-gms:1.2.2")
```

Pin exact versions in production builds and upgrade intentionally.

## Support

For SDK access, integration help, or production verification setup, contact
Foil support.
