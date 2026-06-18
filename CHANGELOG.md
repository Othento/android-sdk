# Changelog

All notable changes to the Othento Android SDK are documented here. The format
is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this
project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

While on `0.x` the public API may still change between minor versions; the first
frozen API ships as `1.0.0`.

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
