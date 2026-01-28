# RFC: Agentic Commerce — Delegate Authentication API

**Status:** Draft  
**Version:** 2026-01-28  
**Scope:** Delegate consumer authentication to merchant-specified providers for agent-driven checkout flows

This RFC defines HTTP endpoints that enable **delegated consumer authentication** with merchant-specified authentication providers. The Agent executes authentication directly with the provider while maintaining parallel execution with payment tokenization for optimal latency.

---

## 1. Scope & Goals

- Provide a **stable, versioned** surface (API-Version = `2026-01-28`).
- Enable merchants to **delegate** authentication to a specified authentication provider, mirroring the existing `payment_provider` pattern.
- Allow Agents to **execute consumer authentication directly** with the specified provider using standardized endpoints, without routing through the merchant.

This specification covers:
- **3D Secure 2** (3DS2) authentication only
- **Browser channel** only (not app/SDK)
- **Native integration** (agent handles fingerprint/challenge flows, not redirect-based)

**Out of scope:** 3D Secure 1, app-based/SDK authentication, redirect-based flows, payment authorization.

### 1.1 Terminology & Normative Language

The key words **MUST**, **MUST NOT**, **SHOULD**, **MAY** are to be interpreted as described in RFC 2119.

---

## 2. Protocol Phases

### 2.1 Initialization

- **Version compatibility:** Agent **MUST** send `API-Version`. Provider **MUST** validate support (e.g., `2026-01-28`).
- **State management:** Provider **MUST** return an opaque `authentication_token` in every response. Agent **MUST** include the most recent token in all subsequent requests.

### 2.2 Session Initialization (`/init`)

- Agent **MUST** issue `POST /delegate_authentication/init` with payment method details, amount, browser info, and callback URL.
- Provider **MUST** return `authentication_token` and `status`.
- If `status` is `action_required`, provider **MUST** include an `action` object with `type: fingerprint`.
- If `status` is `pending`, no browser action is needed; agent proceeds directly to `/authenticate`.
- If `status` is `not_supported`, authentication is not available for this card (e.g., card not enrolled, issuer not participating in 3DS); agent **SHOULD** proceed directly to authorization without 3DS data.

### 2.3 Device Fingerprinting (Browser Action)

When `/init` returns `action.type: fingerprint`:

1. Agent **MUST** construct the 3DS Method payload containing `threeDSServerTransID` and `threeDSMethodNotificationURL`.
2. Agent **MUST** base64-encode the payload and POST it as `threeDSMethodData` to `action.fingerprint.three_ds_method_url` via a hidden iframe.
3. ACS collects device fingerprint and POSTs to the agent's notification URL.
4. Agent **MUST** call `/authenticate` within 10 seconds with `fingerprint_completion: Y` (success) or `N` (timeout).
5. If no `action` was returned from `/init`, agent **MUST** call `/authenticate` with `fingerprint_completion: U`.

### 2.4 Authentication Request (`/authenticate`)

- Agent **MUST** issue `POST /delegate_authentication/authenticate` with `authentication_token` and `fingerprint_completion` result.
- Provider initiates authentication.
- Provider **MUST** return updated `authentication_token` and `status`.

**Possible status values:**
- `authenticated`: Frictionless success.
- `action_required`: Challenge required.
- `attempted`: Proof of authentication attempt provided.
- `not_authenticated`: Authentication failed.
- `rejected`: Issuer rejected.
- `unavailable`: Technical error.

If `status` is `action_required`, provider **MUST** include an `action` object with `type: challenge` and the agent proceeds to the challenge flow.
If no `action` is returned, agent proceeds to completion `/complete`.

### 2.5 Challenge Flow (Browser Action)

When `/authenticate` returns `action.type: challenge`:

1. Agent **MUST** construct the CReq payload containing `threeDSServerTransID`, `acsTransID`, `messageVersion`, `messageType: CReq`, and `challengeWindowSize`.
2. Agent **MUST** base64-encode (no padding) and POST it as `creq` to `action.challenge.acs_url` via an iframe/modal.
3. Shopper completes challenge in the rendered UI.
4. ACS POSTs `CRes` to agent's `challenge_notification_url`.
5. Agent **MUST** decode CRes and extract `transStatus`.
6. Agent **MUST** call `/complete` with `challenge_result` set to the extracted `transStatus`.

### 2.6 Completion (`/complete`)

- Agent **MUST** issue `POST /delegate_authentication/complete` with `authentication_token`.
- After a challenge, agent **MUST** include `challenge_result` (Y/N/U from CRes).
- Provider **MUST** return final `status` and `authentication_result` containing available 3DS authentication data (trans_status, ECI, cryptogram, transaction IDs, message_version, etc.).
- The `authentication_result` object **MUST** be returned for all terminal statuses except `unavailable`. Fields are populated based on availability from the authentication response.

**Possible status values:**
- `authenticated`: Authentication successful.
- `attempted`: Proof of authentication attempt provided.
- `not_authenticated`: Authentication failed.
- `rejected`: Issuer rejected.
- `unavailable`: Technical error.

### 2.7 Error Handling

- Errors **MUST** be returned as **flat JSON objects** with fields: `type`, `code`, `message`, optional `param`.
- Servers **SHOULD** use appropriate HTTP status codes:
  - `400/422`: Invalid request or semantic validation error
  - `401`: Unauthorized
  - `409`: Idempotency conflict
  - `429`: Rate limit exceeded
  - `500/503`: Processing or service unavailable

---

## 3. HTTP Interface

_TODO: Define `/delegate_authentication/init`, `/delegate_authentication/authenticate`, `/delegate_authentication/complete` endpoints_

---

## 4. Responses

_TODO: Define success and error response formats_

---

## 5. Agent Callback Endpoints

_TODO: Define `notification_urls` for fingerprint and challenge completion callbacks_

---

## 6. Security Considerations

_TODO: Define authentication, transport security, and data handling requirements_

---

## 7. Validation Rules

_TODO: Define request validation requirements_

---

## 8. Examples

_TODO: Provide frictionless and challenge flow examples_

---

## 9. Conformance Checklist

_TODO: Define implementation requirements checklist_

---

## 10. Change Log

- **2026-01-28**: Initial draft. Scope and goals defined.
