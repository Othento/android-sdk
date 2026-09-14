# Changelog

All notable changes to the Othento Android SDK are documented here. The format
is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this
project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

While on `0.x` the public API may still change between minor versions; the first
frozen API ships as `1.0.0`.

## [0.1.3] — 2026-09-14

Patch release. Coordinate: `com.othento:othento-core:0.1.3`.

### Action required when upgrading

- **Versions 0.1.0, 0.1.1 and 0.1.2 have been removed from this repository**
  and can no longer be downloaded. They pointed at an API host that is no
  longer supported. Change your dependency to
  `com.othento:othento-core:0.1.3`. Builds that still request an older version
  fail with `Could not find com.othento:othento-core:0.1.x`. Their entries
  below are kept for reference only.

### Changed
- **Production API host moved to `https://sdk-api.othento.com`.** `pk_live_…`
  keys now reach the new production endpoint. Nothing changes for you: there
  is no new configuration, and sandbox keys work the same as before.

### Docs
- **`OthentoExpectedDetails` field formats documented.** The README now lists
  what each field expects: `dateOfBirth` as `yyyy-MM-dd`, `gender` as `"M"` or
  `"F"`, `nationality` and `country` as ISO 3166-1 alpha-3 codes, and
  `ipAddress` as the expected end-user IPv4 address.

## [0.1.2] — 2026-08-25

Feature release. Coordinate: `com.othento:othento-core:0.1.2`.

### Action required when upgrading

- **Core-library desugaring is now required in your app module.** The new AWS
  Face Liveness component needs it, and Gradle does not inherit the setting
  from a library module, so the build fails with
  `Dependency 'com.amplifyframework:core:2.29.0' requires core library
  desugaring to be enabled for :app` until you enable it. Add
  `isCoreLibraryDesugaringEnabled = true` plus
  `coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")` — see
  "Core-library desugaring is required" in the README.
- **Two transitive versions moved up**, both pulled by the liveness component:
  OkHttp `4.12.0` → `5.0.0-alpha.14` (the AWS streaming client needs
  `okhttp-coroutines`, which has no 4.x equivalent) and Compose BOM
  `2024.10.01` → `2025.03.01`. If you pin either yourself, pin at or above
  these.

### Added
- **Localization** — all in-flow copy, icons, and tenant branding now resolve
  from the backend content bundle instead of being compiled into the SDK, with
  a local fallback bundle when the content call fails. Users can switch
  language mid-flow when the tenant publishes more than one, and right-to-left
  languages lay the entire flow out RTL.
- **`OthentoConfig.language(String)`** — BCP-47 code (`"en"`, `"ar"`, …)
  recorded on the session at create time and used to resolve SDK copy. Omit it
  and the backend applies the tenant default; the backend stays the authority,
  so an unpublished language falls back rather than erroring. In token mode the
  session already carries a language, so this acts only as a fallback.
- **AWS Rekognition Face Liveness** — a new liveness screen selected by the
  workflow, running the AWS Face Liveness component against backend-minted
  credentials, with typed mapping from AWS failures onto `OthentoError`.
- **"Update required" screen** — a workflow that asks for a step this SDK
  version cannot render now ends on an explicit update prompt instead of an
  opaque error.
- **Searchable country picker** — the document-select country dropdown gained
  inline search across localized and English names and ISO codes.

### Changed
- **AML / inference results surface much faster.** When a session is processing
  server-side with nothing left for the user to do, polling switches to a
  background profile: the first poll fires immediately (no leading delay), then
  every 1 s for up to ~60 s, instead of the default 2 s cadence.
- **Post-submit decision wait raised from ~10 s to ~16 s** (5 → 8 polls).
  Timing out here is terminal — it routes to "Verification under review" — so
  the shorter window was ending sessions whose decision was seconds away.
- **Capture raised to full HD.** Document capture and liveness video moved from
  HD to 1080p, and the front camera used for selfie / face-match was raised to
  1080p as well, with a reworked detection pipeline, frame analyzer, and MRZ /
  passport gating.

### Fixed
- **OTP retry stranding.** If the send-code response came back with a session
  that had left the OTP step — a late identity-inference `Failed`, an expiry,
  or an advance to another step — the user was left on a dead code-entry
  screen. Those responses now route through the normal screen resolver, so the
  flow reaches the retry, terminal, or next screen as it should.

## [0.1.1] — 2026-06-18

Patch release. Coordinate: `com.othento:othento-core:0.1.1`.

### Fixed
- **OTP input overflow on narrow screens** — long verification codes (8–10
  cells) no longer run off the screen edge. Cell size, spacing, and digit
  text now shrink uniformly to fit the available width, capped at the spec
  size so shorter codes (4–6) stay full-size.
  
## [0.1.0] — 2026-06-14

First published release. Coordinate: `com.othento:othento-core:0.1.0`.

### Added
- **Identity-verification flow**: document capture (camera + library-upload
  fallback), back-side + selfie / face-match, liveness video, and OTP
  (phone / email), with full session orchestration and polling.
- **Two session modes** — `create` (apiKey + workflow + client data) and
  `token` (a server-minted SAT, e.g. delivered via deeplink).
- **Host integration surface**: `OthentoSDK`, `OthentoConfig` (+ `Builder`),
  `OthentoSDKListener` with `onReady` / `onSessionCreated` / `onStatusChanged`
  / `onCompleted` / `onCancelled` / `onError`, and typed `OthentoDecision`,
  `OthentoSessionStatus`, `OthentoError`, `OthentoCancelReason`.
- **`OthentoConfig.closeOnComplete`** — auto-dismiss the SDK on any terminal
  screen instead of leaving it up for the user to dismiss.
- **`OthentoConfig.loggingEnabled`** — opt into verbose, auth-redacted HTTP
  logging in release builds for integration debugging.
- **Maven publishing** — Central-compliant artifact set (AAR + sources JAR +
  Dokka javadoc JAR + complete POM + optional GPG signing), distributed via a
  GitHub static-Maven repository.
- **Consumer ProGuard/R8 rules** shipped in the AAR (public API, Moshi DTOs +
  generated adapters, and the bundled native ML components) — no host-side
  keep-rules required.

### Notes
- Sandbox vs. production is selected by the API-key prefix
  (`pk_sandbox_…` / `pk_live_…`); there is no environment flag.
- Visual branding is configured server-side (workflow / tenant), not through
  the SDK — there is no integrator branding API, by design.
- The SDK transitively requires the **JitPack** repository (for the bundled
  document-detection component). See the integration guide.
