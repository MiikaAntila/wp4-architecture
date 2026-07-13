# WE BUILD - Pre-flight Conformance Specification: Credential Presentation via the Digital Credentials API

Version 0.1 / Pre-flight Draft
Date: 4 July 2026

**Authors**: WP4 Architecture

* Leif Johansson <leifj@siros.org>

Table Of Contents

- [1. Introduction](#1-introduction)
- [2. Scope](#2-scope)
- [3. Normative Language](#3-normative-language)
- [4. Roles and Components](#4-roles-and-components)
- [5. Protocol Overview](#5-protocol-overview)
- [6. High-level Flows](#6-high-level-flows)
  - [6.1 Same-device Presentation via DC API](#61-same-device-presentation-via-dc-api)
  - [6.2 Cross-device Presentation via DC API Hybrid Transport](#62-cross-device-presentation-via-dc-api-hybrid-transport)
- [7. Normative Requirements](#7-normative-requirements)
  - [7.1 Wallet Unit Requirements](#71-wallet-unit-requirements)
  - [7.2 Verifier Requirements](#72-verifier-requirements)
- [8. Platform and Browser Support Considerations](#8-platform-and-browser-support-considerations)
  - [8.1 Browser Extension Polyfill (Wallet-side)](#81-browser-extension-polyfill-wallet-side)
  - [8.2 Verifier-side Polyfill](#82-verifier-side-polyfill)
- [9. Conformance](#9-conformance)
- [References](#references)

# 1. Introduction

This document is a **pre-flight conformance specification** as defined in the [Pre-flight CS ADR](../adr/pre-flight-CS.md). It is intended to enable early testing of credential presentation using the W3C Digital Credentials API (DC API) [1] within the WE BUILD ecosystem. The goal is to gather implementation experience and testing feedback that will inform a future full conformance specification.

The Digital Credentials API provides a browser-native mechanism for verifiers to request credential presentations from wallet units. For same-device flows, this removes the need for custom protocol schemes (such as `openid4vp://`). For cross-device flows, the DC API leverages CTAP2 hybrid transport to connect the verifier's browser to a remote wallet, with the browser mediating the entire interaction. Both modes integrate credential presentation into the browser's security model.

This specification complements **CS-002 (Credential Presentation)** [2] by defining how the same OpenID4VP-based presentation protocol operates when the browser's DC API serves as the invocation and transport layer, rather than custom URL schemes or redirect flows.

# 2. Scope

This specification defines the conformance expectations for credential presentation using the Digital Credentials API:

* **In scope:**
  * Same-device web presentation flows using `navigator.identity.get()` with the `"digital-credentials"` provider
  * Cross-device presentation flows using DC API hybrid transport (CTAP2 / BLE + tunnel)
  * Integration of OpenID4VP request/response with the DC API transport
  * Verifier-side JavaScript API usage
  * Wallet unit registration and response handling via the DC API
  * Limitations of web-based wallet units and known mitigations

* **Out of scope:**
  * Cross-device presentation flows via QR code scanning without browser mediation (covered by CS-002 §6.2)
  * Proximity-based presentation (e.g. ISO 18013-5 / BLE)
  * Credential issuance via DC API (`navigator.credentials.create()`) — not yet mature
  * Detailed trust evaluation and trust list resolution (covered by other WE BUILD specifications)

# 3. Normative Language

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

> **Note:** As a pre-flight specification, the normative requirements herein are preliminary and subject to revision based on testing feedback.

# 4. Roles and Components

| Role | Description |
|------|-------------|
| **Wallet Unit (WU)** | A native application or web application acting on behalf of the Holder, registered with the platform as a digital credential provider. |
| **Holder** | The person controlling the Wallet Unit. |
| **Verifier (Relying Party)** | A web application that requests credential presentations via the DC API. |
| **User Agent (Browser)** | The browser mediating the DC API interaction between the Verifier and the Wallet Unit. |
| **Wallet Companion** | A browser extension that bridges the DC API to web-based wallet units that cannot natively register as credential providers (see §8.1). |

# 5. Protocol Overview

The Digital Credentials API [1] extends the W3C Credential Management API [3] to support digital identity credentials. A verifier calls `navigator.identity.get()` with a request containing an OpenID4VP presentation request. The browser mediates the interaction:

1. The verifier constructs an OpenID4VP request object and passes it to the DC API.
2. The browser identifies registered wallet units capable of fulfilling the request.
3. The browser presents a wallet selection UI to the user (if multiple wallets are available).
4. The selected wallet unit receives the request, processes it, obtains holder consent, and returns the presentation response.
5. The browser delivers the response back to the verifier's JavaScript context.

This flow keeps the OpenID4VP request/response semantics from CS-002 intact while replacing the invocation and transport mechanism with the browser-native DC API.

The key specification governing this interaction is the **W3C Digital Credentials API** [1], which at the time of writing is a Working Draft. Browser support is available in Chrome 141+ on Android, macOS, and desktop platforms, and is progressing in other browsers. Cross-device support via hybrid transport is at an earlier stage (see §6.2 and [6]).

# 6. High-level Flows

## 6.1 Same-device Presentation via DC API

This flow describes how a verifier web application requests a credential presentation from a wallet unit using the Digital Credentials API.

### 6.1.1 Verifier Constructs Presentation Request

The verifier constructs an OpenID4VP authorization request as specified in CS-002 §6.1.1. The request is encoded as a JSON object suitable for the DC API.

### 6.1.2 DC API Invocation

The verifier invokes the DC API:

```javascript
const presentationResponse = await navigator.identity.get({
  digital: {
    providers: [{
      protocol: "openid4vp",
      request: JSON.stringify(openid4vpRequest)
    }]
  }
});
```

The `protocol` field MUST be set to `"openid4vp"`. The `request` field contains the serialized OpenID4VP authorization request.

### 6.1.3 Browser Mediation

The browser:
1. Identifies installed wallet units registered for the `"openid4vp"` protocol.
2. Presents a selection UI to the user if multiple wallets are available.
3. Forwards the request to the selected wallet unit.

### 6.1.4 Wallet Processing and Holder Consent

The wallet unit:
1. Parses and validates the OpenID4VP request as specified in CS-002 §6.1.3.
2. Identifies matching credentials.
3. Presents a consent screen to the holder, showing the requested attributes and the verifier's identity.
4. Upon consent, generates the verifiable presentation with selective disclosure as appropriate.

### 6.1.5 Response Delivery

The wallet unit returns the OpenID4VP response via the DC API. The browser delivers the response to the verifier's JavaScript context as the resolved value of the `navigator.identity.get()` promise.

### 6.1.6 Verifier Validation

The verifier validates the presentation response as specified in CS-002 §6.1.7, including:
- Signature verification
- Credential status checks
- Trust chain validation

## 6.2 Cross-device Presentation via DC API Hybrid Transport

The DC API supports cross-device presentation using CTAP2 hybrid transport [7]. This is architecturally distinct from the QR-based cross-device flow in CS-002 §6.2: the browser on the verifier's device mediates the entire interaction rather than the wallet connecting directly to the verifier's backend.

### 6.2.1 Verifier Constructs Presentation Request

The verifier constructs the OpenID4VP authorization request identically to §6.1.1. No changes to the request format are required for cross-device operation.

### 6.2.2 DC API Invocation with Hybrid Transport

The verifier invokes the DC API as in §6.1.2. The browser determines that no local wallet is available (or that the user selects a remote device) and initiates hybrid transport:

1. The browser displays a QR code or uses BLE advertisement to establish a CTAP2 hybrid connection to the holder's remote device.
2. The holder scans the QR code or accepts the BLE pairing on their mobile device.
3. A secure tunnel is established between the verifier's browser and the remote wallet unit.

### 6.2.3 Remote Wallet Processing

The remote wallet unit:
1. Receives the OpenID4VP request via the hybrid transport tunnel.
2. Validates and processes the request as specified in §6.1.4.
3. Returns the OpenID4VP response through the same tunnel.

### 6.2.4 Response Delivery

The browser receives the response via the hybrid tunnel and delivers it to the verifier's JavaScript context as the resolved value of `navigator.identity.get()`. From the verifier's perspective, the response is indistinguishable from a same-device response.

### 6.2.5 Verifier Validation

The verifier validates the response identically to §6.1.6.

> **Note:** Cross-device DC API support via hybrid transport is at an early stage of browser implementation. Implementers SHOULD track [6] for current platform availability and be prepared for the hybrid path to be unavailable on some browser/OS combinations.

# 7. Normative Requirements

## 7.1 Wallet Unit Requirements

| ID | Requirement | Reference |
|----|-------------|-----------|
| WU-DC-01 | The WU MUST register itself with the platform as a digital credential provider for the `"openid4vp"` protocol. | [1] §3.1 |
| WU-DC-02 | The WU MUST accept OpenID4VP authorization requests received via the DC API. | [1], [4] |
| WU-DC-03 | The WU MUST return OpenID4VP authorization responses via the DC API response mechanism. | [1], [4] |
| WU-DC-04 | The WU MUST support the same credential formats and selective disclosure mechanisms as required by CS-002 §7.1. | [2] |
| WU-DC-05 | The WU SHOULD support both DC API and `openid4vp://` invocation to ensure backward compatibility. | [2], [4] |
| WU-DC-06 | The WU SHOULD support receiving DC API requests via CTAP2 hybrid transport to enable cross-device flows. | [1], [7] |

## 7.2 Verifier Requirements

| ID | Requirement | Reference |
|----|-------------|-----------|
| VP-DC-01 | The Verifier MUST use `navigator.identity.get()` with the `"digital-credentials"` provider type when the DC API is available. | [1] |
| VP-DC-02 | The Verifier MUST set the `protocol` field to `"openid4vp"` in the DC API request. | [1], [5] |
| VP-DC-03 | The Verifier MUST construct a valid OpenID4VP authorization request as specified in CS-002 §7.2. | [2], [4] |
| VP-DC-04 | The Verifier SHOULD implement fallback to `openid4vp://` custom URL scheme or cross-device flow when the DC API is not available. | [2] |
| VP-DC-05 | The Verifier MUST call the DC API from a [secure context](https://w3c.github.io/webappsec-secure-contexts/) and in response to a user activation event. | [1] §2.1 |
| VP-DC-06 | The Verifier SHOULD support cross-device presentation via the DC API hybrid transport path where the browser provides it. | [1], [7] |

# 8. Platform and Browser Support Considerations

The DC API is not yet universally supported across browsers and platforms. At the time of writing, presentation support is available in Chrome 141+ on Android, macOS, and desktop platforms, with other browsers at various stages of development [6]. Verifiers and wallet providers MUST plan for environments where the DC API is absent or where the `"openid4vp"` protocol is not natively supported.

Two complementary polyfill strategies exist to bridge these gaps. They address different sides of the interaction and MAY be deployed independently or together.

## 8.1 Browser Extension Polyfill (Wallet-side)

The DC API requires wallet units to register as credential providers at the OS or browser level. Web-based wallet units — running as ordinary web applications or PWAs — cannot do this because the registration path requires native platform integration (e.g. Android CredentialManager, iOS AuthenticationServices).

A **browser extension** can act as a polyfill on the wallet side by:

1. Registering itself as a credential provider proxy with the browser.
2. Intercepting DC API requests issued by verifiers via `navigator.identity.get()`.
3. Routing the OpenID4VP request to a web-based wallet that has registered with the extension.
4. Returning the wallet's OpenID4VP response back through the DC API to the verifier.

This approach is transparent to verifiers: they use the standard DC API and are unaware that the response originates from a web wallet via the extension rather than a native wallet. The trade-off is that the user must install the extension.

This pattern enables web wallets to participate in DC API flows on platforms where native wallet registration is not possible.

## 8.2 Verifier-side Polyfill

When the DC API is not available in the browser at all — because the browser does not implement it, or because no wallet (native or extension-based) supports the `"openid4vp"` protocol — the verifier must fall back to invoking OpenID4VP directly, as specified in CS-002.

A **verifier-side polyfill** (implemented as a JavaScript library or WebAssembly module) can bridge this gap by:

1. Detecting whether the DC API is available and whether a compatible wallet is registered.
2. If the DC API is available, using it as the primary invocation path (this specification).
3. If the DC API is unavailable or the request fails with a `NotSupportedError`, falling back to:
   - **Same-device flow:** Redirecting to an `openid4vp://` custom URL scheme with the authorization request (CS-002 §6.1).
   - **Cross-device flow:** Displaying a QR code encoding the OpenID4VP request URI for scanning by a mobile wallet (CS-002 §6.2).

This approach is transparent to the wallet: it receives a standard OpenID4VP request regardless of whether the verifier used the DC API or a direct invocation. The trade-off is that the verifier takes on responsibility for wallet invocation, transport, and response handling that the DC API would otherwise mediate.

Verifier-side polyfill libraries SHOULD present a unified API to the verifier application, abstracting the detection and fallback logic so that application code does not need to manage multiple invocation paths directly.

# 9. Conformance

A **Verifier** conforms to this specification if it satisfies all requirements in §7.2.

A **Wallet Unit** conforms to this specification if it satisfies all requirements in §7.1.

Conformance testing for this pre-flight specification will be defined as part of the feedback process described in the [Pre-flight CS ADR](../adr/pre-flight-CS.md). Implementers are encouraged to report their testing experience to inform the development of a full conformance specification.

# References

| # | Reference |
|---|-----------|
| [1] | W3C, "Digital Credentials API", W3C Working Draft, https://wicg.github.io/digital-credentials/ |
| [2] | WE BUILD, "Conformance Specification: Credential Presentation v1.1 (CS-002)", 2026 |
| [3] | W3C, "Credential Management Level 1", W3C Recommendation, https://www.w3.org/TR/credential-management-1/ |
| [4] | OpenID Foundation, "OpenID for Verifiable Presentations (OpenID4VP) 1.0", https://openid.net/specs/openid-4-verifiable-presentations-1_0.html |
| [5] | OpenID Foundation, "OpenID4VP over the W3C Digital Credentials API", https://openid.net/specs/openid-4-verifiable-presentations-1_0-dc-api.html |
| [6] | W3C Web Identity & Credentials Adoption CG, "Digital Credentials API Ecosystem Support", https://digitalcredentials.dev/ecosystem-support |
| [7] | FIDO Alliance, "Client to Authenticator Protocol (CTAP) 2.2 — Hybrid Transport", https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-transport |
