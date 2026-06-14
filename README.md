# Othento Android SDK

> Drop-in identity verification for Android — document capture, selfie / face
> match, liveness, and OTP — launched from your app with a few lines of Kotlin.

The SDK takes over the foreground in a full-screen flow, drives the whole
verification, and hands you a typed decision (`Approved` / `Declined` /
`InReview`) through a listener. The API surface mirrors the web SDK 1:1, so
error codes, session statuses, and analytics events line up across platforms.

- **Coordinate:** `com.othento:othento-core:0.1.0`
- **Min SDK:** 24 · **compile/target:** 34 · **Kotlin:** 2.0+
- **UI:** renders with Jetpack Compose internally — your app does **not** need Compose.

---

## Contents

1. [Requirements](#requirements)
2. [Permissions](#permissions)
3. [Install](#install)
4. [Quick start](#quick-start)
5. [Configuration reference](#configuration-reference)
6. [Lifecycle & callbacks](#lifecycle--callbacks)
7. [Status & decision values](#status--decision-values)
8. [Error codes](#error-codes)
9. [Cancellation reasons](#cancellation-reasons)
10. [ProGuard / R8](#proguard--r8)
11. [Testing your integration](#testing-your-integration)
12. [Troubleshooting](#troubleshooting)
13. [Versioning](#versioning)
14. [Support](#support)

---

## Requirements

Before wiring the SDK in, make sure you have:

- A **partner account** on the Othento platform with at least one **workflow**
  configured. Workflow IDs are issued from your dashboard.
- A **public API key** (`pk_sandbox_…` or `pk_live_…`). Sandbox keys hit the
  test environment; live keys bill against your plan. **There is no environment
  flag** — the key decides. See [Sandbox vs production](#sandbox-vs-production).

| Setting | Value |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| `compileSdk` / `targetSdk` | 34 |
| Kotlin | 2.0+ |
| JDK (build) | 17 |

If you don't have credentials yet, contact your account manager to be onboarded.

---

## Permissions

The SDK declares the permissions it needs in its own `AndroidManifest.xml`;
they are **merged automatically** into your app — you do not declare them
yourself.

| Permission | Why | Granted |
|---|---|---|
| `INTERNET` | API communication with the Othento backend. | Install-time (no prompt). |
| `CAMERA` | Document capture, selfie, and liveness. | **Runtime** — the SDK requests it at the moment of first capture, and shows an in-flow rationale + "Open Settings" path if permanently denied. |

Notes:
- It does **not** request `READ_MEDIA_*` / storage permissions; the optional
  "upload from library" fallback uses the system photo picker.
- `<uses-feature android:name="android.hardware.camera.any" android:required="true" />`
  is declared, so on devices without a camera the app is filtered on the Play
  Store. If you support camera-less devices, override `required` to `false` in
  your manifest.

---

## Install

The SDK is served from a GitHub static-Maven repository, and transitively
depends on one component hosted on **JitPack**, so add **both** repositories.

```kotlin
// settings.gradle.kts → dependencyResolutionManagement (or your root repositories block)
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        // Othento SDK:
        maven { url = uri("https://raw.githubusercontent.com/Othento/android-sdk/main") }
        // Required transitively (document-detection component):
        maven { url = uri("https://jitpack.io") }
    }
}
```

> ⚠️ Without `jitpack.io` the build fails to resolve `com.github.pqpo:SmartCropper`.

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("com.othento:othento-core:0.1.0")
}
```

---

## Quick start

Launch the flow from an `Activity` and observe the listener:

```kotlin
import com.othento.core.api.OthentoSDK
import com.othento.core.api.OthentoConfig
import com.othento.core.api.OthentoSDKListener
import com.othento.core.api.OthentoDecision
import com.othento.core.api.OthentoError
import com.othento.core.api.OthentoCancelReason
import com.othento.core.api.OthentoSessionStatus

val config = OthentoConfig.Builder()
    .create(
        workflowExternalId = "<<YOUR_WORKFLOW_ID>>",
        clientData = "user-123",           // your end-user identifier
    )
    .apiKey("<<YOUR_PUBLIC_API_KEY>>")     // pk_sandbox_… or pk_live_…
    .build()

OthentoSDK.launch(activity, config, object : OthentoSDKListener {
    override fun onReady() {}
    override fun onSessionCreated(externalId: String, sessionUrl: String) {
        // Persist externalId against your user record.
    }
    override fun onStatusChanged(status: OthentoSessionStatus) {}
    override fun onCompleted(decision: OthentoDecision, externalId: String) {
        // decision = Approved | Declined | InReview
    }
    override fun onCancelled(reason: OthentoCancelReason) {}
    override fun onError(error: OthentoError, displayMessage: String) {
        // error.code is a stable identifier; displayMessage is user-facing.
    }
})
```

`launch(activity, config, listener)` constructs and starts the SDK in one call.
`activity` is your current `Activity` (e.g. `this`). Configuration is validated
**synchronously** — missing or malformed fields throw `OthentoConfigException`
from `build()`; they never arrive via `onError`.

### Tearing down

```kotlin
val sdk = OthentoSDK(activity, config, listener)
sdk.start()
// later, to cancel before completion:
sdk.destroy()   // fires onCancelled(HostDestroy); a safe no-op after a terminal event
```

Exactly one of `onCompleted` / `onCancelled` / `onError` fires per session, and
only one SDK instance may run at a time.

### Sandbox vs production

There is **no environment flag**. The API-key prefix decides: `pk_sandbox_…`
hits the test environment, `pk_live_…` bills real verifications. Same SDK build.

---

## Configuration reference

`OthentoConfig.Builder`:

| Method | Required | Purpose |
|---|---|---|
| `create(workflowExternalId, clientData)` | ✅ | The workflow id + your end-user identifier. |
| `apiKey(String)` | ✅ | Public API key (`pk_sandbox_…` / `pk_live_…`). |
| `callbackUrl(String)` | optional | Redirect URL forwarded to the create-session call. |
| `callbackReceiver(OthentoCallbackReceiver)` | optional | Which webhook the platform invokes: `Initiator` / `Completer` / `Both`. |
| `metadata(String)` | optional | Free-form string round-tripped on session events / webhooks. |
| `expectedDetails(OthentoExpectedDetails)` | optional | Identity hints to compare against extracted data (see below). |
| `closeOnComplete(Boolean)` | optional (default `false`) | Auto-dismiss the SDK on any terminal screen instead of leaving it up. |
| `loggingEnabled(Boolean)` | optional (default `false`) | Verbose, auth-redacted HTTP logging in release builds for debugging. |

### `OthentoExpectedDetails`

All fields optional — send only what you already know. Used by the backend to
cross-check against the data extracted from the document/selfie.

```kotlin
OthentoExpectedDetails(
    firstName = null,
    lastName = null,
    dateOfBirth = null,   // ISO-8601, e.g. "1990-04-23"
    gender = null,
    nationality = null,
    country = null,
    address = null,
    documentNumber = null,
    ipAddress = null,
)
```

---

## Lifecycle & callbacks

```
launch() → onReady → onSessionCreated
                          ↓
                 onStatusChanged (deduped, n×)
                          ↓
        exactly one terminal callback:
        onCompleted | onCancelled | onError
```

All callbacks are dispatched on the **Android main thread**, asynchronously —
they never run synchronously inside `start()`/`launch()`. Exceptions thrown
from your callback are caught and logged; they do not crash the SDK.

| Callback | When it fires |
|---|---|
| `onReady()` | SDK is up. Fires once, before any other event. |
| `onSessionCreated(externalId, sessionUrl)` | A new session was minted. Save `externalId` against your user record. |
| `onStatusChanged(status)` | Session status transitioned. Deduplicated — never the same status twice in a row. |
| `onCompleted(decision, externalId)` | Terminal decision reached: `Approved`, `Declined`, or `InReview`. |
| `onCancelled(reason)` | User dismissed, host called `destroy()`, or the SDK cancelled before a terminal status. |
| `onError(error, displayMessage)` | Non-decision terminal error. `error.code` is stable; `displayMessage` is the localized user-facing string. |

---

## Status & decision values

```kotlin
enum class OthentoSessionStatus { Created, InProgress, Processing, Completed, Expired, Failed }
enum class OthentoDecision { Approved, Declined, InReview }
```

These names are identical to the web SDK's, so the same analytics and webhook
payloads work across platforms.

---

## Error codes

`OthentoError` is a sealed class; `error.code` is the stable wire string. Switch
on the subtype (or `code`) for exhaustive handling.

| `code` | Meaning | Recommended action |
|---|---|---|
| `config_invalid` | Required field missing/malformed (thrown sync from `build()`). | Fix at build time. |
| `missing_api_key` | No `apiKey`. | Provide the public API key. |
| `missing_workflow` | No `workflowExternalId`. | Provide the workflow ID. |
| `missing_client_data` | No `clientData`. | Pass your end-user identifier. |
| `session_not_found` | Session id didn't resolve (HTTP 404). | Start a fresh session. |
| `network` | Connectivity / DNS / TLS failure. | Retry. |
| `create_failed` | Session could not be created. | Verify `apiKey` + `workflowExternalId`. |
| `documents_failed` | Document list/processing failed. | Prompt retry with better lighting. |
| `initiate_failed` | Flow failed to start after session creation. | Retry on a fresh instance. |
| `upload_failed` | Evidence upload failed; `error.stage` says which step (`UploadUrl` / `S3Put` / `ConfirmUpload`). | Retry; check connectivity. |
| `poll_failed` | Polling for the result failed. | Retry; the session may still resolve. |
| `file_too_large` | Picked file exceeded the upload limit (>10 MB). | Prompt the user to retry capture. |
| `unknown` | Unclassified runtime error. | Show `displayMessage`; capture logs + `externalId` for support. |

```kotlin
override fun onError(error: OthentoError, displayMessage: String) {
    when (error) {
        is OthentoError.Network      -> showRetry()
        is OthentoError.SessionNotFound -> restartSession()
        is OthentoError.UploadFailed -> showRetry()   // error.stage available
        else                         -> showGenericError(displayMessage)
    }
}
```

---

## Cancellation reasons

`onCancelled(reason)` fires once with a typed `OthentoCancelReason`:

| `reason` | Trigger |
|---|---|
| `HostDestroy` | Host called `sdk.destroy()` before a terminal event. |
| `SdkInternal` | The SDK's own UI cancelled the session. |
| `BackButton` | System back button / back gesture. |

---

## ProGuard / R8

Nothing to add. The AAR ships `consumer-rules.pro` keeping the public API, the
Moshi DTOs + generated adapters, and the bundled native ML components — your
release build works without host-side keep-rules.

---

## Testing your integration

A healthy integration produces, in order:

1. `onReady` shortly after `launch()`.
2. `onSessionCreated` with a non-empty `externalId`.
3. `onStatusChanged(InProgress)` at least once.
4. Exactly one terminal callback.

Use the sandbox test documents from your dashboard to exercise approved /
declined / in-review paths deterministically.

- [ ] Sandbox key in place; the flow launches and `onReady` fires
- [ ] Approved-path document → `onCompleted(Approved)`
- [ ] Declined-path document → `onCompleted(Declined)`
- [ ] In-review document → `onCompleted(InReview)`
- [ ] Back out mid-flow → `onCancelled(BackButton)`

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build error: cannot resolve `com.github.pqpo:SmartCropper` | `jitpack.io` not in repositories. | Add `maven { url = uri("https://jitpack.io") }`. |
| Build error: cannot resolve `com.othento:othento-core` | Othento Maven repo missing or wrong branch. | Add `https://raw.githubusercontent.com/Othento/android-sdk/main`. |
| `onError(create_failed)` immediately | Wrong/expired `apiKey` or `workflowExternalId`. | Verify credentials and that the key matches the environment. |
| Camera screen never appears | Camera permission permanently denied. | The SDK shows an "Open Settings" path; the user must grant it. |
| Release build crashes during JSON parse | Aggressive R8 in a non-standard setup stripping DTOs. | The shipped consumer rules cover this; file an issue if you've customized R8. |

---

## Versioning

[SemVer](https://semver.org/). While on `0.x` the public API may change between
minor versions; the first frozen API ships as `1.0.0`. Pin an exact version in
production. Pre-`1.0.0` versions may be re-published on the GitHub repo.

---

## Support

Include the SDK version (`0.1.0`), the `externalId` of the affected session, and
a logcat capture (enable `loggingEnabled(true)` while reproducing) when
contacting your account manager or opening a ticket.

---

## License

MIT — see [LICENSE](./LICENSE). Release history in [CHANGELOG.md](./CHANGELOG.md).
