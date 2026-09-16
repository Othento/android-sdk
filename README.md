# Othento Android SDK

> Drop-in identity verification for Android — document capture, selfie / face
> match, liveness, and OTP — launched from your app with a few lines of Kotlin.

The SDK takes over the foreground in a full-screen flow, drives the whole
verification, and hands you a typed decision (`Approved` / `Declined` /
`InReview`) through a listener. The API surface mirrors the web SDK 1:1, so
error codes, session statuses, and analytics events line up across platforms.

- **Coordinate:** `com.othento:othento-core:0.1.4`
- **Min SDK:** 24 · **compile/target:** 34 · **Kotlin:** 2.0+
- **UI:** renders with Jetpack Compose internally — your app does **not** need Compose.

---

## Contents

1. [Requirements](#requirements)
2. [How it works](#how-it-works)
3. [Permissions](#permissions)
4. [Install](#install)
5. [Step 1 — Create a session on your server](#step-1--create-a-session-on-your-server)
6. [Step 2 — Launch the SDK in your app](#step-2--launch-the-sdk-in-your-app)
7. [Configuration reference](#configuration-reference)
8. [Lifecycle & callbacks](#lifecycle--callbacks)
9. [Status & decision values](#status--decision-values)
10. [Error codes](#error-codes)
11. [Cancellation reasons](#cancellation-reasons)
12. [ProGuard / R8](#proguard--r8)
13. [Testing your integration](#testing-your-integration)
14. [Troubleshooting](#troubleshooting)
15. [Versioning](#versioning)
16. [Support](#support)

---

## Requirements

Before wiring the SDK in, make sure you have:

- A **partner account** on the Othento platform with at least one **workflow**
  configured. Workflow IDs are issued from your dashboard.
- Your application's **API key** from the dashboard. It stays on **your
  server** — never embed it in the app, where anyone can extract it from the
  APK. Sandbox and live keys decide the environment; see
  [Sandbox vs production](#sandbox-vs-production).
- A **server endpoint** of your own that creates a verification session and
  returns its session access token to the app. See
  [Step 1](#step-1--create-a-session-on-your-server).

| Setting | Value |
|---|---|
| `minSdk` | 24 (Android 7.0) |
| `compileSdk` / `targetSdk` | 34 |
| Kotlin | 2.0+ |
| JDK (build) | 17 |

If you don't have credentials yet, contact your account manager to be onboarded.

---

## How it works

```
 Your app                   Your server                    Othento API
 ────────                   ───────────                    ───────────
 user taps "Verify" ────▶  POST /your/verify-session
                           (user is authenticated)
                                    │
                                    ├──▶ POST /api/v1/SessionToken/session
                                    │    X-API-KEY: <your API key>
                                    │◀── { externalId, sessionToken: "sat:…" }
                                    │
                           save externalId on the user
 receive { sat } ◀─────────  return { sat }

 OthentoConfig.Builder().tokenMode(sat)
 onCompleted(decision, …) ◀── the SDK runs the verification

                           webhook: session.completed ◀── final result
```

1. **Your server** creates the session with your API key and gets back a
   **session access token** (SAT, prefixed `sat:`).
2. **Your app** passes only that SAT to the SDK.
3. The SDK runs the verification and reports the decision. Your server also
   receives it through webhooks, keyed by the `externalId` you saved in step 1.

Your API key never ships inside your app, and the session is bound to the user
your server authenticated — not to whatever identifier the device sends.

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
- The SDK records **audio-less** video for liveness — it does **not** request
  `RECORD_AUDIO`.
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
    implementation("com.othento:othento-core:0.1.4")
}
```

### Core-library desugaring is required

The SDK's AWS Face Liveness component requires **core-library desugaring**.
Gradle does not inherit this setting from a library module, so you must enable
it in your own app module:

```kotlin
// app/build.gradle.kts
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    implementation("com.othento:othento-core:0.1.4")
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

Without it the build fails with:

```
Dependency 'com.amplifyframework:core:2.29.0' requires core library desugaring
to be enabled for :app.
```

### Versions the SDK brings onto your classpath

The SDK pulls these transitively. If you pin any of them yourself, pin at or
above these versions:

| Dependency | Version | Why |
|---|---|---|
| OkHttp | `5.0.0-alpha.14` | Required by the AWS streaming client (it needs `okhttp-coroutines`, which has no 4.x equivalent). |
| Compose BOM | `2025.03.01` | Carried by the AWS Face Liveness UI component. |

---

## Step 1 — Create a session on your server

Call this endpoint **from your backend only**, once per verification attempt.

```
POST https://sdk-api.othento.com/api/v1/SessionToken/session
X-API-KEY: <your API key>
Content-Type: application/json
```

### Request body

| Field | Type | Required | Notes |
|---|---|---|---|
| `workflowExternalId` | `String` | yes | The workflow to run, from your dashboard. |
| `clientData` | `String` | yes | Your identifier for the end user. Take it from **your authenticated session**, never from the app's request body. Echoed in webhooks. |
| `language` | `String` | optional | BCP-47 code (`"en"`, `"ar"`, …) the flow starts in. Omit for your tenant default. Right-to-left languages lay the whole flow out RTL. |
| `callbackUrl` | `String` | optional | Where the user is redirected on completion (if applicable). |
| `callbackReceiver` | `"Initiator"` / `"Completer"` / `"Both"` | optional | Which party receives the `callbackUrl` redirect. |
| `metadata` | `String` | optional | Free-form string round-tripped on session events and webhooks. |
| `expectedDetails` | object | optional | Identity hints compared against the document and selfie. See [Expected details](#expected-details). |

### Response

The response describes the new session. The fields you need:

| Field | Type | Use it for |
|---|---|---|
| `sessionToken` | `String` | The session access token (`sat:…`). **Return this to your app** as `sat`. |
| `externalId` | `String` | The session ID. **Save it on the user record** — webhooks reference it. |
| `expiresAt` | `String` | ISO-8601 time after which the session can no longer be started. |

### Example — Node.js (Express)

```ts
app.post('/api/verify-session', requireLogin, async (req, res) => {
  const response = await fetch('https://sdk-api.othento.com/api/v1/SessionToken/session', {
    method: 'POST',
    headers: {
      'X-API-KEY': process.env.OTHENTO_API_KEY!, // server-side secret
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      workflowExternalId: process.env.OTHENTO_WORKFLOW_ID,
      clientData: req.user.id, // from YOUR session, not from req.body
      language: req.user.language,
    }),
  });

  if (!response.ok) {
    return res.status(502).json({ error: 'verification_unavailable' });
  }

  const session = await response.json();
  await db.users.update(req.user.id, { othentoSessionId: session.externalId });

  res.json({ sat: session.sessionToken });
});
```

### Example — cURL

```bash
curl -X POST https://sdk-api.othento.com/api/v1/SessionToken/session \
  -H "X-API-KEY: <your API key>" \
  -H "Content-Type: application/json" \
  -d '{
    "workflowExternalId": "<YOUR_WORKFLOW_ID>",
    "clientData": "user-123"
  }'
```

### Server-side errors

The endpoint uses the same API-key authentication as the Othento REST API, so
it answers with the same error statuses:

| Status | Meaning | What to do |
|---|---|---|
| `400` | A field is missing or invalid. | Check `workflowExternalId`, `clientData`, and field formats. |
| `401` | Missing or invalid API key. | Check the key and that it belongs to this application. |
| `402` | No active subscription on your account. | Check billing in your dashboard. |
| `403` | Application or organization suspended. | Contact your account manager. |
| `404` | `workflowExternalId` not found for this key. | Check the workflow ID and the environment of the key. |

Don't forward these to the end user as-is — show a generic "verification is
unavailable, try again" message and log the detail.

### Handling the session access token

- **Create a new session for every attempt.** Don't persist a SAT in the app
  for a later launch, and never share one between users.
- **Send it only to the user it was created for**, over HTTPS, as the response
  to that user's authenticated request.
- **Don't log it** or send it to analytics or crash reporting. Anyone holding
  the SAT can open that session until it expires.
- **If the SDK reports `session_not_found`** or the session has expired, call
  your endpoint again for a fresh SAT and launch a new SDK instance.

### Expected details

All fields are optional. Send only what you already know about the user. The
backend compares these values with the data read from the document, the selfie,
and the user's connection. Leave out any field you don't know. Don't send an
empty string or a placeholder, because it is compared like a real value.

```json
{
  "workflowExternalId": "<YOUR_WORKFLOW_ID>",
  "clientData": "user-123",
  "expectedDetails": {
    "firstName": "Sara",
    "lastName": "Haddad",
    "dateOfBirth": "1988-01-01",
    "gender": "F",
    "nationality": "JOR",
    "country": "JOR",
    "address": "Amman, Jordan",
    "documentNumber": "A1234567",
    "ipAddress": "203.0.113.10"
  }
}
```

| Field | Type | What to send | Example |
|---|---|---|---|
| `firstName` | `String` | The user's first (given) name. | `"Sara"` |
| `lastName` | `String` | The user's last (family) name. | `"Haddad"` |
| `dateOfBirth` | `String` | Date of birth. **Must be `yyyy-MM-dd`**: 4-digit year, 2-digit month, 2-digit day, zero-padded. | `"1988-01-01"` |
| `gender` | `String` | **`"M"` or `"F"`**, a single uppercase letter. | `"M"` |
| `nationality` | `String` | The user's nationality as an **ISO 3166-1 alpha-3** country code (3 uppercase letters). | `"JOR"` |
| `country` | `String` | The user's country as an **ISO 3166-1 alpha-3** country code (3 uppercase letters). | `"JOR"` |
| `address` | `String` | The user's address as free text. | `"Amman, Jordan"` |
| `documentNumber` | `String` | The ID document number. | `"A1234567"` |
| `ipAddress` | `String` | The **IPv4** address you expect the end user to connect from. | `"203.0.113.10"` |

### Sandbox vs production

There is **no environment flag** in the SDK. The API key your server uses
decides: a sandbox key creates sandbox sessions, a live key creates billable
production sessions. The SAT carries that environment with it, so the app code
is identical in both.

---

## Step 2 — Launch the SDK in your app

Fetch a SAT from your server when the user taps "Verify", then launch the flow
from an `Activity` and observe the listener:

```kotlin
import com.othento.core.api.OthentoSDK
import com.othento.core.api.OthentoConfig
import com.othento.core.api.OthentoSDKListener
import com.othento.core.api.OthentoDecision
import com.othento.core.api.OthentoError
import com.othento.core.api.OthentoCancelReason
import com.othento.core.api.OthentoSessionStatus

// `sat` is the sessionToken your server returned (Step 1).
val config = OthentoConfig.Builder()
    .tokenMode(sat)
    .build()

OthentoSDK.launch(activity, config, object : OthentoSDKListener {
    override fun onReady() {}
    override fun onSessionCreated(externalId: String, sessionUrl: String) {
        // Not called in token mode — your server already has the externalId.
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
**synchronously** — an empty `sat` throws `OthentoConfigException` from
`build()`; it never arrives via `onError`.

> **Treat the in-app result as a UI signal, not proof.** Update your records
> from the webhook your server receives (or by looking the session up
> server-side), not from a value reported by the device.

### Tearing down

```kotlin
val sdk = OthentoSDK(activity, config, listener)
sdk.start()
// later, to cancel before completion:
sdk.destroy()   // fires onCancelled(HostDestroy); a safe no-op after a terminal event
```

Exactly one of `onCompleted` / `onCancelled` / `onError` fires per session, and
only one SDK instance may run at a time.

---

## Configuration reference

`OthentoConfig.Builder`:

| Method | Required | Purpose |
|---|---|---|
| `tokenMode(String)` | ✅ | The `sessionToken` your server received in [Step 1](#step-1--create-a-session-on-your-server). |
| `closeOnComplete(Boolean)` | optional (default `false`) | Auto-dismiss the SDK on any terminal screen instead of leaving it up. |
| `loggingEnabled(Boolean)` | optional (default `false`) | Verbose, auth-redacted HTTP logging in release builds for debugging. |

Everything about the session itself — workflow, user, language, callback URL,
metadata, expected details — is set **on your server** when you create it. See
[Request body](#request-body). Users can still switch language in-flow when the
tenant publishes more than one.

---

## Lifecycle & callbacks

```
launch() → onReady
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
| `onStatusChanged(status)` | Session status transitioned. Deduplicated — never the same status twice in a row. |
| `onCompleted(decision, externalId)` | Terminal decision reached: `Approved`, `Declined`, or `InReview`. |
| `onCancelled(reason)` | User dismissed, host called `destroy()`, or the SDK cancelled before a terminal status. |
| `onError(error, displayMessage)` | Non-decision terminal error. `error.code` is stable; `displayMessage` is the localized user-facing string. |

`onSessionCreated` is part of the listener interface but is not called in token
mode: you already have the session's `externalId` from
[Step 1](#step-1--create-a-session-on-your-server).

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
| `config_invalid` | Required field missing/malformed — for example an empty `sat` (thrown sync from `build()`). | Fix at build time. |
| `session_not_found` | The SAT didn't resolve to a session (HTTP 404). | Check the SAT was passed unmodified; request a new one from your server. |
| `network` | Connectivity / DNS / TLS failure. | Retry. |
| `documents_failed` | Document list/processing failed. | Prompt retry with better lighting. |
| `initiate_failed` | Flow failed to start. | Retry on a fresh instance. |
| `upload_failed` | Evidence upload failed; `error.stage` says which step (`UploadUrl` / `S3Put` / `ConfirmUpload`). | Retry; check connectivity. |
| `poll_failed` | Polling for the result failed. | Retry; the session may still resolve. |
| `file_too_large` | Picked file exceeded the upload limit (>10 MB). | Prompt the user to retry capture. |
| `unknown` | Unclassified runtime error. | Show `displayMessage`; capture logs + `externalId` for support. |

```kotlin
override fun onError(error: OthentoError, displayMessage: String) {
    when (error) {
        is OthentoError.Network         -> showRetry()
        is OthentoError.SessionNotFound -> restartWithNewSession()   // fetch a fresh SAT
        is OthentoError.UploadFailed    -> showRetry()               // error.stage available
        else                            -> showGenericError(displayMessage)
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

1. Your server endpoint returns a `sat` beginning with `sat:` and stores the
   session's `externalId` on the user.
2. `onReady` shortly after `launch()`.
3. `onStatusChanged(InProgress)` at least once.
4. Exactly one terminal callback.
5. Your webhook endpoint receives the final session event for the same
   `externalId`.

Use the sandbox test documents from your dashboard to exercise approved /
declined / in-review paths deterministically.

- [ ] Server creates sessions with the **sandbox** API key; no API key anywhere in the app module
- [ ] `clientData` comes from the logged-in user on the server, not from the app's request body
- [ ] The flow launches and `onReady` fires
- [ ] Approved-path document → `onCompleted(Approved)`
- [ ] Declined-path document → `onCompleted(Declined)`
- [ ] In-review document → `onCompleted(InReview)`
- [ ] Back out mid-flow → `onCancelled(BackButton)`
- [ ] An expired SAT → `onError`, handled by fetching a new one

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Build error: cannot resolve `com.github.pqpo:SmartCropper` | `jitpack.io` not in repositories. | Add `maven { url = uri("https://jitpack.io") }`. |
| Build error: cannot resolve `com.othento:othento-core` | Othento Maven repo missing or wrong branch. | Add `https://raw.githubusercontent.com/Othento/android-sdk/main`. |
| Your server gets `401` from `/SessionToken/session` | Missing or wrong `X-API-KEY`, or a key from a different application. | Check the key your server sends and the application it belongs to. |
| `build()` throws `OthentoConfigException` | `sat` is empty — usually the server response was read with the wrong field name. | Return `session.sessionToken` from your server and pass it to `tokenMode(...)`. |
| `onError(session_not_found)` right after launch | The SAT was altered (trimmed, re-encoded) or has expired. | Pass the token exactly as received; create the session when the user taps "Verify", not at app start. |
| Camera screen never appears | Camera permission permanently denied. | The SDK shows an "Open Settings" path; the user must grant it. |
| Release build crashes during JSON parse | Aggressive R8 in a non-standard setup stripping DTOs. | The shipped consumer rules cover this; file an issue if you've customized R8. |

---

## Versioning

[SemVer](https://semver.org/). While on `0.x` the public API may change between
minor versions; the first frozen API ships as `1.0.0`. Pin an exact version in
production. Pre-`1.0.0` versions may be re-published on the GitHub repo.

---

## Support

Include the SDK version (`0.1.4`), the `externalId` of the affected session, and
a logcat capture (enable `loggingEnabled(true)` while reproducing) when
contacting your account manager or opening a ticket.

---

## License

MIT — see [LICENSE](./LICENSE). Release history in [CHANGELOG.md](./CHANGELOG.md).
