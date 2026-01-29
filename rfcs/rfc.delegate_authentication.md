# RFC: Agentic Commerce - Delegate Authentication API

**Status:** Draft  
**Version:** 2026-01-28  
**Scope:** Delegate consumer authentication to merchant-specified providers for agent-driven checkout flows

This RFC defines a standardized REST API contract that authentication providers SHOULD implement to enable **delegated consumer authentication** with merchant-specified authentication providers.

---

## 1. Scope & Goals

- Provide a **stable, versioned** API surface (`API-Version: 2026-01-28`) that the client calls to create authentication sessions, triggers authentication, and retrieve authentication outcomes.
- Enable merchants to **delegate** authentication to a specified authentication provider.
- Allow clients to **execute consumer authentication directly** with the specified server using standardized endpoints, without routing through the merchant.

This specification covers:
- **3D Secure 2** (3DS2) authentication only
- **Browser channel** only (not app/SDK)
- **Native integration** (client handles fingerprint/challenge flows, not redirect-based)

**Out of scope:** app-based/SDK authentication, redirect-based flows, payment authorization.

### 1.1 Normative Language

The key words **MUST**, **MUST NOT**, **SHOULD**, **MAY** are to be interpreted as described in RFC 2119.

---

## 2. Protocol Phases

### 2.1 Initialization

- **Versioning:** Client **MUST** send `API-Version`. Server **MUST** validate support (e.g., `2026-01-28`).
- **Identity/Signing:** Server **SHOULD** publish acceptable signature algorithms out-of-band; client **SHOULD** sign requests (`Signature`) over canonical JSON with an accompanying `Timestamp` (RFC 3339).
- **Session-based state:** Server **MUST** return an opaque `authentication_session_id`. Client **MUST** include this ID in the URL path for all subsequent requests.

### 2.2 Authentication Lifecycle

1. **Create** - `POST /delegate_authentication` initializes a session with merchant, card, and amount. Returns fingerprint action if supported, or `not_supported` if card not enrolled.
2. **Authenticate** - `POST /delegate_authentication/{authentication_session_id}/authenticate` submits fingerprint result and triggers authentication. Returns challenge action if required, or frictionless result.
3. **Complete** - `POST /delegate_authentication/{authentication_session_id}/complete` finalizes and retrieves the final authentication result (cryptogram, ECI, etc.).

### 2.3 Client Callbacks (Separate)

Client **MUST** implement callback endpoints to receive async notifications from the issuer's ACS:
- **Fingerprint notification** - Receives 3DS Method completion.
- **Challenge notification** - Receives challenge result (CRes) after shopper verification.

---

## 3. HTTP Interface

### 3.1 Common Requirements

All endpoints **MUST** use HTTPS and return JSON.

**Request Headers (sent by client):**

- `Authorization: Bearer <token>` (**REQUIRED**)
- `Content-Type: application/json` (**REQUIRED** on requests with body)
- `Accept-Language: en-us` (OPTIONAL)
- `User-Agent: <string>` (OPTIONAL)
- `Idempotency-Key: <string>` (RECOMMENDED)
- `Request-Id: <string>` (RECOMMENDED)
- `Signature: <base64url>` (RECOMMENDED; identity verification over canonical request)
- `Timestamp: <RFC3339>` (RECOMMENDED)
- `API-Version: 2026-01-28` (**REQUIRED**)

**Response Headers:**

- `Idempotency-Key` - echo if provided
- `Request-Id` - echo if provided

**Error Shape (flat):**

```json
{
  "type": "invalid_request",
  "code": "invalid_card",
  "message": "Missing/malformed field",
  "param": "payment_method.number"
}
```

**Type & Code Guidelines**

- `type` ∈ `invalid_request`, `rate_limit_exceeded`, `processing_error`, `service_unavailable`
- `code` ∈ `invalid_card`, `duplicate_request`, `idempotency_conflict`
- `param` **SHOULD** be an RFC 9535 JSONPath (when applicable).

---

## 4. Endpoints

### 4.1 Create Session

`POST /delegate_authentication` → **201 Created**

**Request body:**

- `merchant_id` (**REQUIRED**) - Merchant identifier
- `payment_method` (**REQUIRED**) - Card details (type, number, exp_month, exp_year, name)
- `amount` (**REQUIRED**) - Object with `value` (integer, minor units) and `currency` (ISO 4217)
- `channel` (**OPTIONAL**) - Browser channel data; can be deferred to authenticate
- `checkout_session_id` (**OPTIONAL**) - Checkout session identifier
- `flow_preference` (**OPTIONAL**) - Object with `type` (`challenge` | `frictionless`) and optional subtype details
- `challenge_notification_url` (**OPTIONAL**) - URL for challenge result callback
- `shopper_details` (**OPTIONAL**) - Object with `name?`, `email?`, `phone_number?`, `address?` (nested Address)

**Response body:**

- `authentication_session_id` (string) - Session ID for subsequent requests
- `status` - `action_required | pending | not_supported`
- `action` (optional) - Object with `type: fingerprint` and fingerprint details

### 4.2 Authenticate

`POST /delegate_authentication/{authentication_session_id}/authenticate` → **200 OK**

**Request body:**

- `fingerprint_completion` (**REQUIRED**) - `Y` (success), `N` (timeout), or `U` (unavailable)
- `channel` (**CONDITIONAL**) - Required if not provided in create
- `checkout_session_id` (**OPTIONAL**)
- `challenge_notification_url` (**CONDITIONAL**) - Required if not provided in create
- `shopper_details` (**OPTIONAL**) - Object with `name?`, `email?`, `phone_number?`, `address?`

**Response body:**

- `authentication_session_id` (string) - Session ID
- `status` - `authenticated | action_required | attempted | not_authenticated | rejected | unavailable`
- `action` (optional) - Object with `type: challenge` and challenge details

### 4.3 Complete

`POST /delegate_authentication/{authentication_session_id}/complete` → **200 OK**

**Response body:**

- `authentication_session_id` (string) - Session ID
- `status` - `authenticated | attempted | not_authenticated | rejected | unavailable`
- `authentication_result` - Object with 3DS data (trans_status, eci, cryptogram, ds_trans_id, three_ds_server_trans_id, message_version, etc.)

---

## 5. Data Model

- **PaymentMethod**: `type` (`card`), `number`, `exp_month`, `exp_year`, `name`
- **Amount**: `value` (int, minor units), `currency` (ISO 4217)
- **Channel**: `type` (`browser`), `browser` (object with `accept_header`, `ip_address`, `javascript_enabled`, `language`, `user_agent`, `color_depth`, `java_enabled`, `screen_height`, `screen_width`, `timezone_offset`)
- **Address**: `name`, `line_one`, `line_two?`, `city`, `state`, `country`, `postal_code`
- **ShopperDetails**: `name?`, `email?`, `phone_number?`, `address?` (nested Address object)
- **FlowPreference**: `type` (`challenge` | `frictionless`), `challenge?` (object with `type`: `mandated` | `preferred`), `frictionless?` (object). Clients **MAY** request a preference, but issuers ultimately decide the actual flow.
- **Action (fingerprint)**: `type: fingerprint`, `fingerprint` (object with `three_ds_method_url`, `three_ds_server_trans_id`)
- **Action (challenge)**: `type: challenge`, `challenge` (object with `acs_url`, `acs_trans_id`, `three_ds_server_trans_id`, `message_version`)
- **AuthenticationResult**: `trans_status`, `eci`, `cryptogram`, `ds_trans_id`, `three_ds_server_trans_id`, `message_version`, `authentication_value?`, `trans_status_reason?`, `cardholder_info?`

---

## 6. Browser Actions

### 6.1 Device Fingerprinting

When create returns `action.type: fingerprint`:

1. Client **MUST** construct payload with `threeDSServerTransID` and `threeDSMethodNotificationURL`.
2. Client **MUST** base64-encode and POST as `threeDSMethodData` to `action.fingerprint.three_ds_method_url` via hidden iframe.
3. ACS collects fingerprint and POSTs to client's notification URL.
4. Client **MUST** call `/{authentication_session_id}/authenticate` with `fingerprint_completion` after receiving notification (`Y`), or after 10 second timeout (`N`).

### 6.2 Challenge Flow

When authenticate returns `action.type: challenge`:

1. Client **MUST** construct CReq with `threeDSServerTransID`, `acsTransID`, `messageVersion`, `messageType: CReq`, `challengeWindowSize`.
2. Client **MUST** base64-encode (no padding) and POST as `creq` to `action.challenge.acs_url` via iframe/modal.
3. Shopper completes challenge.
4. ACS POSTs `CRes` to client's `challenge_notification_url`.
5. Client **MUST** decode CRes, extract `transStatus`, and call `/{authentication_session_id}/complete`.

---

## 7. Idempotency & Retries

- Clients **SHOULD** provide `Idempotency-Key` on **create** requests for safe retries.
- Replays with the **same** key and **identical** parameters **MUST** return the original result.
- If the same key is replayed with **different** parameters, server **MUST** return `409` with:
  ```json
  {
    "type": "invalid_request",
    "code": "idempotency_conflict",
    "message": "Same Idempotency-Key used with different parameters"
  }
  ```
- Servers **SHOULD** be tolerant of network timeouts and implement at-least-once processing with idempotency.

---

## 8. Security Considerations

- **Authentication:** `Authorization: Bearer <token>` **REQUIRED**.
- **Integrity:** `Signature` over canonical JSON **SHOULD** be verified (algorithm policy advertised out-of-band).
- **Freshness:** `Timestamp` **SHOULD** be required and checked within an acceptable clock-skew window.
- **PII/PCI:** Card data handling **MUST** follow applicable PCI DSS requirements; logs **MUST NOT** contain full PAN or CVC.
- **Transport:** All requests **MUST** use HTTPS/TLS 1.2+.
- **Callback verification:** Client **SHOULD** validate callback origin where possible.

---

## 9. Validation Rules (non-exhaustive)

**Payment Method:**
- `payment_method.type` **MUST** be `card`.
- `payment_method.number` present (string).
- When present:
  - `exp_month` length ≤ 2 and value `"01"`-`"12"`.
  - `exp_year` length ≤ 4 and four digits.

**Amount:**
- `amount.value` **MUST** be a positive integer (minor units).
- `amount.currency` **MUST** be ISO 4217 (e.g., `EUR`, `USD`).

**Channel:**
- `channel.type` **MUST** be `browser` when present.
- `channel.browser` required fields: `accept_header`, `ip_address`, `javascript_enabled`, `language`, `user_agent`.
- When `javascript_enabled` is `true`: `color_depth`, `java_enabled`, `screen_height`, `screen_width`, `timezone_offset` **MUST** be present.

**Flow Preference:**
- `flow_preference.type` ∈ `challenge | frictionless` when present.
- `flow_preference.challenge.type` ∈ `mandated | preferred` when present.

**Shopper Details:**
- `shopper_details.address.country` **MUST** be ISO 3166-1 alpha-2 when present.

**Status Values:**
- `fingerprint_completion` ∈ `Y | N | U`.
- Create `status` ∈ `action_required | pending | not_supported`.
- Authenticate `status` ∈ `authenticated | action_required | attempted | not_authenticated | rejected | unavailable`.
- Complete `status` ∈ `authenticated | attempted | not_authenticated | rejected | unavailable`.

---

## 10. Examples

### 10.1 Create - Request (minimal)

```json
{
  "merchant_id": "merchant_abc123",
  "payment_method": {
    "type": "card",
    "number": "4917610000000000",
    "exp_month": "03",
    "exp_year": "2030",
    "name": "Jane Doe"
  },
  "amount": {
    "value": 1000,
    "currency": "EUR"
  }
}
```

### 10.2 Create - Request (with channel)

```json
{
  "merchant_id": "merchant_abc123",
  "payment_method": {
    "type": "card",
    "number": "4917610000000000",
    "exp_month": "03",
    "exp_year": "2030",
    "name": "Jane Doe"
  },
  "amount": {
    "value": 1000,
    "currency": "EUR"
  },
  "checkout_session_id": "checkout_session_123",
  "channel": {
    "type": "browser",
    "browser": {
      "accept_header": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
      "ip_address": "192.168.1.1",
      "javascript_enabled": true,
      "language": "en-US",
      "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)...",
      "color_depth": 24,
      "java_enabled": false,
      "screen_height": 1080,
      "screen_width": 1920,
      "timezone_offset": 0
    }
  },
  "flow_preference": {
    "type": "challenge",
    "challenge": {
      "type": "preferred"
    }
  },
  "challenge_notification_url": "https://agent.example.com/3ds/challenge-callback",
  "shopper_details": {
    "name": "Jane Doe",
    "email": "shopper@example.com",
    "phone_number": "15551234567",
    "address": {
      "name": "Jane Doe",
      "line_one": "123 Main Street",
      "line_two": "Apt 4B",
      "city": "Amsterdam",
      "state": "NH",
      "country": "NL",
      "postal_code": "1012 AB"
    }
  }
}
```

### 10.3 Create - Response (201, fingerprint required)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "action_required",
  "action": {
    "type": "fingerprint",
    "fingerprint": {
      "three_ds_method_url": "https://acs.issuer.com/3dsmethod",
      "three_ds_server_trans_id": "abc-123-def"
    }
  }
}
```

### 10.4 Create - Response (201, not supported)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "not_supported"
}
```

### 10.5 Authenticate - Request

```json
{
  "fingerprint_completion": "Y"
}
```

### 10.6 Authenticate - Response (200, frictionless)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "authenticated"
}
```

### 10.7 Authenticate - Response (200, challenge required)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "action_required",
  "action": {
    "type": "challenge",
    "challenge": {
      "acs_url": "https://acs.issuer.com/challenge",
      "acs_trans_id": "xyz-789",
      "three_ds_server_trans_id": "abc-123-def",
      "message_version": "2.2.0"
    }
  }
}
```

### 10.8 Complete - Response (200, authenticated)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "authenticated",
  "authentication_result": {
    "trans_status": "Y",
    "eci": "05",
    "cryptogram": "AQIDBAUGBwgJCgsMDQ4PEBESExQ=",
    "ds_trans_id": "c4e59ceb-a382-4d6a-bc87-385d591fa09d",
    "three_ds_server_trans_id": "6edcc246-23ee-4e94-ac5d-8ae620bea7d9",
    "message_version": "2.2.0",
    "authentication_value": "CAVV_VALUE"
  }
}
```

### 10.9 Complete - Response (200, not authenticated)

```json
{
  "authentication_session_id": "auth_session_abc123",
  "status": "not_authenticated",
  "authentication_result": {
    "trans_status": "N",
    "trans_status_reason": "01",
    "eci": "07",
    "ds_trans_id": "3022b509-391e-4c25-a225-f37e062843a5",
    "three_ds_server_trans_id": "87a357a1-faee-4cd1-b3cf-ea2c704bf073",
    "message_version": "2.2.0",
    "cardholder_info": "Please contact your bank's customer support."
  }
}
```

### 10.10 Error - 400 (invalid card)

```json
{
  "type": "invalid_request",
  "code": "invalid_card",
  "message": "Invalid expiry date (exp_month or exp_year)",
  "param": "payment_method.exp_month"
}
```

### 10.11 Error - 409 (idempotency conflict)

**409 Conflict**

```json
{
  "type": "invalid_request",
  "code": "idempotency_conflict",
  "message": "Same Idempotency-Key used with different parameters"
}
```

---

## 11. Conformance Checklist

- [ ] Accepts `API-Version` and validates `2026-01-28`
- [ ] Verifies `Authorization` (Bearer)
- [ ] Verifies auth; signs/verifies requests where applicable
- [ ] Implements create, authenticate, complete endpoints
- [ ] Returns appropriate `status` values per endpoint
- [ ] Returns `action` object when browser action required
- [ ] Returns `authentication_result` on complete
- [ ] Emits flat error objects with `type/code/message/param?`
- [ ] Supports `channel` in either create or authenticate
- [ ] Honors `Idempotency-Key`; returns `409` on conflict

---

## 12. Change Log

- **2026-01-28**: Initial draft. Scope, goals, and session-based API defined.
