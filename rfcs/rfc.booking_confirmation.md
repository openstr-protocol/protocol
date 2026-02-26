# RFC: Booking Confirmation

**RFC ID:** openstr-rfc-003  
**Title:** Booking Request, Confirmation, and Cancellation  
**Status:** Draft  
**Version:** 0.1.0  
**Created:** February 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Supersedes:** None  

---

## Abstract

This RFC defines the OpenSTR booking flow — the process by which an AI agent submits a booking request on behalf of a guest, the host system validates and confirms or declines it, and the confirmed booking response is returned. It covers two confirmation paths (instant book and request-to-book), the bilateral credential validation process, the confirmed pricing and policy snapshot, the post-confirmation location disclosure, access code delivery, and a standardised cancellation endpoint. This RFC is the final transactional layer of the OpenSTR v0.1 core protocol.

---

## 1. Motivation and Problem Statement

The property listing and availability query RFCs define how an agent discovers a property and confirms its availability and pricing. The booking confirmation RFC defines what happens next — the legally and financially consequential act of creating a reservation.

This layer must handle several concerns simultaneously: validating that the guest's identity credential meets the host's requirements, authorising payment via ACP/AP2-compatible payment tokens, managing the two-step request-to-book flow where the host must actively accept before a booking is confirmed, disclosing the exact property address only at the point of confirmation, delivering access credentials, and providing a standardised mechanism for cancellation.

Getting this layer right is critical. An ambiguous or incomplete booking confirmation spec creates the conditions for disputes, double-bookings, and trust failures that would undermine the protocol's credibility.

---

## 2. Prior Art

### 2.1 Agentic Commerce Protocol (ACP)

ACP (OpenAI / Stripe) defines a standardised checkout and payment flow for AI agent commerce. OpenSTR booking requests reference ACP-compatible payment tokens for payment authorisation. ACP does not define accommodation-specific booking flows, credential validation, access code delivery, or cancellation.

### 2.2 OTA / OpenTravel Alliance

OTA defines XML-based reservation request and response schemas (`OTA_HotelResRQ` / `OTA_HotelResRS`). These are comprehensive but heavyweight enterprise B2B schemas not suitable for lightweight agent-to-host transactions.

### 2.3 W3C Verifiable Credentials

The W3C Verifiable Credentials Data Model defines how identity and reputation claims are expressed, signed, and verified. OpenSTR booking requests carry a `GuestCredential` conforming to this model. The credential is validated by the host system before the booking is confirmed.

### 2.4 Existing STR Platform Booking Flows

Airbnb, Vrbo, and Booking.com each define proprietary booking flows accessible only through their own platforms and approved API partners. None expose an open, standardised booking endpoint accessible to arbitrary AI agents. OpenSTR defines the open equivalent.

**Gap:** No open standard exists for a complete, agent-native STR booking flow covering credential validation, instant and request-to-book paths, post-confirmation location disclosure, access delivery, and cancellation. This RFC defines that standard.

---

## 3. Design Decisions

### 3.1 Quote ID as Booking Anchor

Every booking request must reference a `quote_id` issued by the availability endpoint. The host system validates the quote before processing the booking — confirming it was genuinely issued, has not expired, and exactly matches the `listing_id`, dates, guest count, and pet count in the booking request. This prevents fabricated, replayed, or price-manipulated booking requests.

### 3.2 Two Confirmation Paths

OpenSTR defines two booking confirmation paths, determined by the `instant_book` flag in the listing and availability response:

- **Instant book** — the host system validates the request and returns a confirmed booking synchronously. The agent receives a confirmed `booking_reference` in the response.
- **Request-to-book** — the host system acknowledges receipt of the request and returns a `pending` status. The host has 24 hours to accept or decline. The agent polls a status endpoint or receives an async callback. If no decision is made within 24 hours, the request expires automatically and no charge is made.

### 3.3 Bilateral Credential Validation

Before confirming a booking, the host system must validate the guest's `GuestCredential` against the `damage_guarantee.idv_level` declared in the listing. The guest agent must validate the host's `HostCredential` before submitting a booking request. Both validations must succeed for a booking to proceed. This is the bilateral trust model described in the README and `rfc.identity_trust.md`.

### 3.4 Payment Delegation

OpenSTR does not define a payment mechanism. Payment is handled by ACP or AP2-compatible payment handlers. The booking request carries a `payment_token` issued by the guest's payment provider. The host system submits this token to the payment handler to authorise the charge. The booking response includes a `payment_reference` from the payment handler confirming authorisation.

### 3.5 Post-Confirmation Location Disclosure

The exact property address is not included in the listing. It is disclosed only upon booking confirmation, in a `location_detail` object within the confirmed booking response. This object is machine-readable and includes full address, exact coordinates, a what3words reference, and a maps deep-link URI, enabling the guest's agent to immediately use location data for itinerary planning, transport queries, and calendar entries.

### 3.6 Access Object

The booking confirmation response includes an `access` object describing how the guest will access the property. In v0.1, two delivery methods are defined: `host_message` (the host sends the access code or instructions via a separate communication channel) and `keybox_code` (a static code is returned directly in the response). Smart lock token generation and delivery via API integration is a v0.4 operational tooling concern and is not defined in v0.1.

### 3.7 Cancellation Endpoint

A standardised cancellation endpoint is included in v0.1. The endpoint accepts a cancellation request referencing the `booking_reference`, validates the request against the booking's cancellation policy, and returns a cancellation confirmation with refund status. Refund processing is delegated to the original payment handler.

### 3.8 Request-to-Book Expiry

For request-to-book listings, the host has a maximum of 24 hours to accept or decline. This window may be shortened by the host but not extended. If no decision is recorded within 24 hours, the host system must automatically expire the request, release the dates, and notify the agent. No charge is made for expired requests.

---

## 4. Specification

### 4.1 Booking Request Endpoint

```
POST /openstr/booking
Content-Type: application/json
```

The endpoint URL is declared in the property listing as `booking_endpoint`.

### 4.2 Booking Request Object

#### 4.2.1 Required Request Fields

| Field | Type | Description |
|---|---|---|
| `listing_id` | string | The `listing_id` from the property listing |
| `quote_id` | string | The `quote_id` from the availability response. Must be valid and unexpired. |
| `check_in` | string (ISO 8601 date) | Check-in date. Must match the quoted dates exactly. |
| `check_out` | string (ISO 8601 date) | Check-out date. Must match the quoted dates exactly. |
| `guests` | integer | Guest count. Must match the quoted guest count exactly. |
| `guest_credential` | object | Verifiable guest identity credential. See Section 4.3. |
| `payment_token` | object | ACP or AP2 compatible payment token. See Section 4.4. |
| `guest_details` | object | Guest contact details. See Section 4.5. |

#### 4.2.2 Optional Request Fields

| Field | Type | Description |
|---|---|---|
| `pets` | integer | Number of pets. Must match the quoted pet count if pets were included in the availability request. |
| `message_to_host` | string | Optional message from the guest to the host. Max 500 characters. |
| `agent_id` | string (URI) | Identifier of the requesting agent. |
| `special_requests` | string | Freeform special requests. Max 300 characters. Not guaranteed to be accommodated. |

#### 4.2.3 Request Validation

The host system must return a `400 Bad Request` if:
- `quote_id` is not recognised, has expired, or does not match the booking request parameters
- `check_in` or `check_out` do not exactly match the quoted dates
- `guests` does not match the quoted guest count
- The `guest_credential` is absent or cannot be parsed
- The `payment_token` is absent or cannot be parsed

The host system must return a `403 Forbidden` if:
- The `guest_credential` fails validation
- The credential's IDV level does not meet the listing's `damage_guarantee.idv_level` requirement
- The credential has expired
- The credential issuer is not in the listing's `idv_accepted_issuers` list

### 4.3 Guest Credential Object

The `guest_credential` object carries the guest's verifiable identity and reputation claims. The full credential schema is defined in `rfc.identity_trust.md`. The fields required at the booking layer are:

| Field | Type | Description |
|---|---|---|
| `credential_uri` | string (URI) | URI of the guest's full verifiable credential document |
| `issuer` | string (URI) | URI of the credential issuer |
| `idv_level` | enum | The IDV level attested by this credential: `email_verified`, `identity_verified`, or `payment_verified` |
| `issued_at` | string (ISO 8601 datetime) | Timestamp of credential issuance |
| `expires_at` | string (ISO 8601 datetime) | Timestamp of credential expiry |
| `proof` | object | Cryptographic proof of credential validity. Format defined in `rfc.identity_trust.md`. |

The host system must fetch and independently verify the credential at `credential_uri` rather than trusting the inline fields alone. The inline fields serve as a pre-validation hint only.

### 4.4 Payment Token Object

OpenSTR delegates payment execution to ACP or AP2-compatible payment handlers. The `payment_token` object carries the token issued by the guest's payment provider.

| Field | Type | Description |
|---|---|---|
| `handler` | enum | `acp` or `ap2` |
| `token` | string | Payment token issued by the payment handler |
| `amount` | number | Amount to be charged. Must match the `total` from the availability response. |
| `currency` | string | ISO 4217 currency code. Must match the quoted currency. |

The host system submits this token to the declared payment handler to authorise the charge. The token must be validated by the payment handler before the booking is confirmed. If payment authorisation fails, the host system must return a `402 Payment Required` response.

### 4.5 Guest Details Object

| Field | Type | Required | Description |
|---|---|---|---|
| `first_name` | string | Yes | Guest's first name |
| `last_name` | string | Yes | Guest's last name |
| `email` | string | Yes | Guest's email address for booking correspondence |
| `phone` | string | No | Guest's phone number in E.164 format (e.g. `+12025550123`) |
| `country_of_residence` | string | No | ISO 3166-1 alpha-2 country code |

---

## 5. Booking Response

### 5.1 Instant Book — Confirmed Response (HTTP 200)

Returned synchronously when `instant_book` is `true` and all validations pass.

| Field | Type | Description |
|---|---|---|
| `booking_reference` | string | Unique booking reference. Stable identifier for this reservation. |
| `status` | enum | Always `confirmed` in this response |
| `listing_id` | string | Echo of `listing_id` |
| `check_in` | string (ISO 8601 date) | Confirmed check-in date |
| `check_out` | string (ISO 8601 date) | Confirmed check-out date |
| `nights` | integer | Number of nights |
| `guests` | integer | Confirmed guest count |
| `pricing_confirmed` | object | Final confirmed pricing. Echo of availability response pricing breakdown. |
| `payment_reference` | string | Reference returned by the payment handler confirming charge authorisation |
| `policies_confirmed` | object | Final confirmed policies snapshot. See Section 4.5 of rfc.availability_query.md. |
| `location_detail` | object | Full property location, disclosed post-confirmation. See Section 5.3. |
| `access` | object | Access information for check-in. See Section 5.4. |
| `host_contact` | object | Host contact details for this booking. See Section 5.5. |
| `confirmed_at` | string (ISO 8601 datetime) | Timestamp of booking confirmation |
| `cancellation_endpoint` | string (URI) | URL of the cancellation endpoint for this booking |

### 5.2 Request-to-Book — Pending Response (HTTP 202)

Returned when `instant_book` is `false`. The booking request has been received and is awaiting host decision.

| Field | Type | Description |
|---|---|---|
| `booking_reference` | string | Unique booking reference. Used for status polling and cancellation. |
| `status` | enum | Always `pending` in this response |
| `listing_id` | string | Echo of `listing_id` |
| `check_in` | string (ISO 8601 date) | Requested check-in date |
| `check_out` | string (ISO 8601 date) | Requested check-out date |
| `guests` | integer | Requested guest count |
| `pricing_quoted` | object | Pricing from the availability response, held pending confirmation |
| `payment_status` | enum | `authorised` — payment is authorised but not yet captured |
| `expires_at` | string (ISO 8601 datetime) | Timestamp after which the request expires if no host decision is made. Maximum 24 hours from submission. |
| `status_endpoint` | string (URI) | URL for the agent to poll for booking status updates |
| `cancellation_endpoint` | string (URI) | URL to cancel the pending request |
| `submitted_at` | string (ISO 8601 datetime) | Timestamp of request submission |

### 5.2.1 Status Polling Endpoint

```
GET /openstr/booking/{booking_reference}/status
```

Returns the current status of a pending booking request.

| Field | Type | Description |
|---|---|---|
| `booking_reference` | string | Booking reference |
| `status` | enum | `pending`, `confirmed`, `declined`, or `expired` |
| `updated_at` | string (ISO 8601 datetime) | Timestamp of last status change |

If `status` is `confirmed`, the full confirmed booking response (Section 5.1) is included in the status response body. If `status` is `declined` or `expired`, a `reason` string is included.

### 5.3 Location Detail Object

Returned only in a confirmed booking response. Provides the full property address and location data for agent use in itinerary planning, transport queries, and calendar entries.

| Field | Type | Required | Description |
|---|---|---|---|
| `address_line_1` | string | Yes | Street address line 1 |
| `address_line_2` | string | No | Street address line 2 |
| `city` | string | Yes | City or town |
| `region` | string | Yes | State, county, or region |
| `postcode` | string | Yes | Full postcode or zip code |
| `country` | string | Yes | ISO 3166-1 alpha-2 country code |
| `lat` | number | Yes | Exact latitude coordinate |
| `lng` | number | Yes | Exact longitude coordinate |
| `what3words` | string | No | what3words address (e.g. `filled.count.soap`) |
| `maps_uri` | string (URI) | No | Deep-link URI to property location in a maps application (e.g. Google Maps, Apple Maps) |
| `directions_notes` | string | No | Freeform directions or arrival notes from the host. Max 500 characters. |

### 5.4 Access Object

Describes how the guest will access the property at check-in.

| Field | Type | Required | Description |
|---|---|---|---|
| `method` | enum | Yes | `in_person`, `keybox`, or `smartlock`. Must match listing's `check_in_method`. |
| `delivery` | enum | Yes | How the access credential is delivered: `host_message` or `inline` |
| `delivery_timing` | string | Yes | Human-readable description of when access details will be provided (e.g. `"Sent to guest email by 08:00 on day of check-in"`) |
| `keybox_code` | string | If `method` is `keybox` and `delivery` is `inline` | The static keybox code. Only included if the host chooses to deliver it inline rather than via message. |
| `access_notes` | string | No | Freeform access instructions. Max 500 characters. |

**Smart lock tokens:** For properties with `method: smartlock`, v0.1 uses `delivery: host_message` — the host delivers the smart lock code or app instructions via their normal communication channel at the time they choose (e.g. morning of check-in). Automated smart lock token generation and time-limited API delivery is a v0.4 operational tooling concern.

**Delivery timing examples:**

| Scenario | `delivery` | `delivery_timing` |
|---|---|---|
| Host sends code on morning of check-in | `host_message` | `"Sent to guest email by 08:00 on day of check-in"` |
| Static keybox code returned at booking | `inline` | `"Code available immediately in this confirmation"` |
| Smart lock — host sends app invite | `host_message` | `"Smart lock access sent via email within 2 hours of booking confirmation"` |

### 5.5 Host Contact Object

Minimum host contact information provided at booking confirmation for guest communication during the stay.

| Field | Type | Required | Description |
|---|---|---|---|
| `display_name` | string | Yes | Host's display name or property management name |
| `email` | string | No | Host contact email for this booking |
| `phone` | string | No | Host phone number in E.164 format |
| `response_time` | string | No | Host's typical response time (e.g. `"Within 1 hour"`) |

---

## 6. Cancellation Endpoint

### 6.1 Cancellation Request

```
POST /openstr/booking/{booking_reference}/cancel
Content-Type: application/json
```

| Field | Type | Required | Description |
|---|---|---|---|
| `booking_reference` | string | Yes | The booking reference to cancel |
| `reason` | enum | Yes | `guest_cancellation`, `host_cancellation`, or `force_majeure` |
| `reason_notes` | string | No | Freeform explanation. Max 300 characters. |
| `requested_by` | enum | Yes | `guest` or `host` |

### 6.2 Cancellation Response (HTTP 200)

| Field | Type | Description |
|---|---|---|
| `booking_reference` | string | Echo of booking reference |
| `status` | enum | Always `cancelled` in this response |
| `cancelled_at` | string (ISO 8601 datetime) | Timestamp of cancellation |
| `cancellation_policy_applied` | enum | The cancellation policy that was applied: `flexible`, `moderate`, `strict`, or `non_refundable` |
| `refund` | object | Refund details. See Section 6.3. |
| `payment_reference` | string | Reference for the refund transaction with the payment handler |

### 6.3 Refund Object

| Field | Type | Description |
|---|---|---|
| `refund_amount` | number | Amount to be refunded to the guest |
| `currency` | string | ISO 4217 currency code |
| `refund_pct` | number | Percentage of total booking cost refunded |
| `refund_basis` | string | Human-readable explanation of refund calculation (e.g. `"Cancelled 8 days before check-in. Moderate policy: 50% refund."`) |
| `refund_status` | enum | `pending`, `processing`, or `completed` |
| `estimated_refund_date` | string (ISO 8601 date) | Estimated date refund will reach the guest |

### 6.4 Cancellation Policy Reference

The refund amount is calculated by the host system based on the `cancellation_policy` declared in the listing and confirmed in the booking response. The following indicative refund structure is defined in v0.1 as a reference. Hosts may declare custom terms in the listing's `house_rules` field, which take precedence.

| Policy | Cancelled 7+ days before check-in | Cancelled 2–6 days before | Cancelled within 24 hours or after check-in |
|---|---|---|---|
| `flexible` | 100% refund | 100% refund | 50% refund |
| `moderate` | 100% refund | 50% refund | No refund |
| `strict` | 50% refund | No refund | No refund |
| `non_refundable` | No refund | No refund | No refund |

**Note:** Cleaning fee is refunded in full if the cancellation occurs before check-in, regardless of policy. The cancellation policy applies to the accommodation cost only.

### 6.5 Host-Initiated Cancellation

If `requested_by` is `host`, the guest is entitled to a full refund regardless of the declared cancellation policy. Host-initiated cancellations should be recorded as such in the host's credential reputation data (v0.2).

---

## 7. Error Responses

| HTTP Status | Meaning |
|---|---|
| `400 Bad Request` | Request validation failed — missing fields, mismatched quote parameters, or malformed request |
| `402 Payment Required` | Payment token authorisation failed |
| `403 Forbidden` | Guest credential validation failed — invalid, expired, insufficient IDV level, or unaccepted issuer |
| `404 Not Found` | `booking_reference` not found (for status, cancellation endpoints) |
| `409 Conflict` | Dates are no longer available — another booking was confirmed between availability query and booking request |
| `410 Gone` | Quote has expired — a fresh availability query is required |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Host system error — agents should retry with exponential backoff |

---

## 8. Full Booking Flow Example

### 8.1 Instant Book Flow

**Step 1 — Booking Request**

```json
{
  "listing_id": "host-abc123-prop-001",
  "quote_id": "quote-20260224-abc123-0704",
  "check_in": "2026-07-04",
  "check_out": "2026-07-11",
  "guests": 2,
  "pets": 1,
  "guest_credential": {
    "credential_uri": "https://identity.example-issuer.com/credentials/guest-7x9k2",
    "issuer": "https://identity.example-issuer.com",
    "idv_level": "identity_verified",
    "issued_at": "2026-01-10T00:00:00Z",
    "expires_at": "2027-01-10T00:00:00Z",
    "proof": {
      "type": "Ed25519Signature2020",
      "created": "2026-01-10T00:00:00Z",
      "verificationMethod": "https://identity.example-issuer.com/keys/1",
      "proofValue": "z3MvGebUd..."
    }
  },
  "payment_token": {
    "handler": "acp",
    "token": "acp_tok_1AbCdEfGhIjKlMnOpQrStUvWx",
    "amount": 1353.50,
    "currency": "USD"
  },
  "guest_details": {
    "first_name": "Alex",
    "last_name": "Morgan",
    "email": "alex.morgan@example.com",
    "phone": "+12025550187",
    "country_of_residence": "US"
  },
  "message_to_host": "Looking forward to our stay. We will have a small dog with us."
}
```

**Step 2 — Confirmed Response**

```json
{
  "booking_reference": "OPENSTR-2026-ABC123-070411",
  "status": "confirmed",
  "listing_id": "host-abc123-prop-001",
  "check_in": "2026-07-04",
  "check_out": "2026-07-11",
  "nights": 7,
  "guests": 2,
  "pricing_confirmed": {
    "currency": "USD",
    "nightly_rate_average": 195,
    "nights": 7,
    "subtotal_nights": 1365,
    "cleaning_fee": 75,
    "pet_fee": 50,
    "discount_applied": {
      "type": "weekly",
      "pct": 10,
      "amount": 136.50
    },
    "total": 1353.50
  },
  "payment_reference": "acp_pay_9ZxYwVuTsRqPoNmLkJiHgFe",
  "policies_confirmed": {
    "cancellation_policy": "moderate",
    "check_in_time": "15:00",
    "check_out_time": "11:00",
    "instant_book": true,
    "damage_guarantee": {
      "mode": "idv_guarantee",
      "idv_level": "identity_verified"
    }
  },
  "location_detail": {
    "address_line_1": "742 W Belden Ave",
    "city": "Chicago",
    "region": "Illinois",
    "postcode": "60614",
    "country": "US",
    "lat": 41.9234,
    "lng": -87.6421,
    "what3words": "filled.count.soap",
    "maps_uri": "https://maps.google.com/?q=41.9234,-87.6421",
    "directions_notes": "Ring the lower bell on arrival. Garden gate code is the same as the front door code."
  },
  "access": {
    "method": "smartlock",
    "delivery": "host_message",
    "delivery_timing": "Sent to guest email by 08:00 on day of check-in",
    "access_notes": "The smart lock is on the left side of the front door. Enter the code and turn the handle within 5 seconds."
  },
  "host_contact": {
    "display_name": "Lincoln Park Stays",
    "email": "host@lincolnparkstays.com",
    "phone": "+13125550142",
    "response_time": "Within 1 hour"
  },
  "confirmed_at": "2026-02-24T14:32:00Z",
  "cancellation_endpoint": "https://yourdomain.com/openstr/booking/OPENSTR-2026-ABC123-070411/cancel"
}
```

### 8.2 Cancellation Example

**Request**

```json
{
  "booking_reference": "OPENSTR-2026-ABC123-070411",
  "reason": "guest_cancellation",
  "reason_notes": "Change of travel plans.",
  "requested_by": "guest"
}
```

**Response** — cancelled 40 days before check-in, moderate policy applies.

```json
{
  "booking_reference": "OPENSTR-2026-ABC123-070411",
  "status": "cancelled",
  "cancelled_at": "2026-05-25T10:15:00Z",
  "cancellation_policy_applied": "moderate",
  "refund": {
    "refund_amount": 1278.50,
    "currency": "USD",
    "refund_pct": 100,
    "refund_basis": "Cancelled 40 days before check-in. Moderate policy: full refund. Cleaning fee refunded in full.",
    "refund_status": "processing",
    "estimated_refund_date": "2026-05-28"
  },
  "payment_reference": "acp_refund_2Kl9MnPqRsTuVwXyZa"
}
```

---

## 9. Agent Behaviour Guidelines

**9.1 Validate host credential before submitting.** The agent must verify the host's `HostCredential` before submitting a booking request. A booking must not be submitted to an endpoint whose host credential is invalid, expired, or from an unrecognised issuer.

**9.2 Present full pricing before requesting confirmation.** The agent must present the complete `pricing_confirmed` breakdown — including all fees and discounts — to the user and receive explicit confirmation before submitting a booking request. The user must know exactly what they are committing to.

**9.3 Present policies before requesting confirmation.** The agent must present the cancellation policy, check-in and check-out times, and damage guarantee terms from `policies_confirmed` to the user before submitting the booking request.

**9.4 Present safety disclosures before requesting confirmation.** The agent must surface the property's `safety_disclosures` — particularly any `not_suitable_infants`, `not_suitable_children`, `security_cameras_exterior`, `security_cameras_interior`, and `weapons_on_premises` flags — to the user before booking confirmation.

**9.5 Poll status for request-to-book.** For pending bookings, agents should poll the `status_endpoint` at reasonable intervals — no more frequently than once every 10 minutes — until a final status is received. Agents should notify the user when a pending request is confirmed, declined, or expired.

**9.6 Handle 409 Conflict gracefully.** A `409 Conflict` response means the dates became unavailable between the availability query and the booking request. The agent should re-query availability and present the user with updated options rather than treating this as a terminal error.

**9.7 Store booking reference securely.** The `booking_reference` is the guest's proof of reservation. Agents must store it securely and make it available to the user for the duration of the stay and a reasonable period afterwards.

**9.8 Use location_detail for itinerary.** Upon receiving a confirmed booking, the agent should immediately use `location_detail` to create calendar entries, suggest transport options, and update any trip itinerary — without requiring the user to copy and paste the address.

**9.9 Surface access delivery timing clearly.** The agent must communicate `access.delivery_timing` to the user so they understand when and how they will receive their access credentials. This is particularly important for smart lock properties where the code is not included inline.

---

## 10. Security Considerations

**10.1 Quote ID validation.** The host system must validate the `quote_id` against its own records. It must reject any booking request where the quote is unrecognised, expired, or where the request parameters do not exactly match the quoted values.

**10.2 Double-booking prevention.** Between availability query and booking confirmation, another booking may be accepted for the same dates. The host system must apply a lock on dates when a booking request is received and processing. If a conflict is detected, a `409 Conflict` must be returned immediately.

**10.3 Payment amount validation.** The host system must verify that the `payment_token.amount` matches the `total` from the original availability response. Booking requests with mismatched payment amounts must be rejected with `400 Bad Request`.

**10.4 Credential fetch and verify.** The host system must independently fetch the credential at `credential_uri` and verify its cryptographic proof. It must not trust the inline `idv_level` field alone. The credential must not be expired and must be issued by a recognised OpenSTR issuer.

**10.5 Location detail protection.** The `location_detail` object must only be returned in a confirmed booking response. It must never be included in pending, declined, or expired booking responses.

**10.6 Keybox code protection.** Where `keybox_code` is returned inline, the booking response must be transmitted over HTTPS only. The host system should consider whether inline delivery is appropriate for their security posture, or whether `host_message` delivery is preferable.

**10.7 Cancellation authentication.** The cancellation endpoint must verify that the cancellation request originates from either the booking guest agent (verified by `guest_credential`) or the host system. Unauthenticated cancellation requests must be rejected.

---

## 11. Open Questions

**11.1 Asynchronous callbacks.** Instead of polling the `status_endpoint`, should the spec define a webhook or callback mechanism for request-to-book status updates? A callback extension is deferred to a future SEP.

**11.2 Booking modification.** This RFC defines creation and cancellation of bookings but not modification. In v0.1, all changes to a confirmed booking require cancellation and rebooking. This is a known limitation — cancel-and-rebook incurs two full payment transactions with associated provider fees on both, risks the desired dates becoming unavailable in the interim, and may trigger cancellation policy penalties unnecessarily.

A dedicated `POST /openstr/booking/:reference/amend` endpoint is planned for v0.2. Three amendment types have been identified: zero-cost amendments (guest count changes within the same pricing tier, guest detail corrections), top-up amendments (pet additions, guest count changes crossing a fee threshold), and repricing amendments (date changes). Each requires different payment settlement behaviour. See `docs/DISCUSSION.md` for full design context, open questions, and a proposed endpoint sketch.

**11.3 Smart lock token format.** Automated smart lock token generation, time-limited access, and API-based delivery are v0.4 concerns. The token format and delivery mechanism will be defined in the operational tooling RFC.

**11.4 Dispute resolution.** This RFC does not define a dispute resolution process for cases where the property does not match the listing, or where damage guarantee claims are made. A dispute resolution extension is proposed for v0.3 or later.

**11.5 Multi-guest credential submission.** The booking request carries a single `GuestCredential` for the lead guest. Whether co-guests should also present credentials is an open question, particularly for large groups. Deferred to a future SEP.

**11.6 Receipt and invoice format.** A standardised machine-readable receipt or invoice format for tax and expense purposes is not defined in v0.1. Deferred to a future SEP.

---

## 12. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |

---

## 13. References

- OpenSTR `rfc.property_listing.md` — Property listing schema and discovery
- OpenSTR `rfc.availability_query.md` — Availability and pricing query
- OpenSTR `rfc.identity_trust.md` — Host and Guest Credential interfaces
- [Agentic Commerce Protocol (ACP)](https://agenticcommerce.dev)
- [Google Agent Payments Protocol (AP2)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)
- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
- [what3words](https://what3words.com/developers/documentation)
- [ISO 8601 — Date and time format](https://www.iso.org/iso-8601-date-and-time-format.html)
- [ISO 3166-1 alpha-2 country codes](https://www.iso.org/iso-3166-country-codes.html)
- [ISO 4217 — Currency codes](https://www.iso.org/iso-4217-currency-codes.html)
- [E.164 telephone number format](https://www.itu.int/rec/T-REC-E.164)
