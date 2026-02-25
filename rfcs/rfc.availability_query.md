# RFC: Availability Query

**RFC ID:** openstr-rfc-002  
**Title:** Availability and Pricing Query  
**Status:** Draft  
**Version:** 0.1.1  
**Created:** February 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Supersedes:** openstr-rfc-002 v0.1.0  

---

## Abstract

This RFC defines the OpenSTR availability query — the transactional layer through which an AI agent checks whether a property is available for a requested date range, retrieves authoritative pricing for that window including all applicable fees and discounts, and learns the minimum stay rules applicable to it. The availability endpoint resolves all dynamic pricing and minimum stay rules declared in the property listing, returning a concrete, actionable response that an agent can present to a user or use to initiate a booking request.

---

## 1. Motivation and Problem Statement

The property listing defined in `rfc.property_listing.md` provides static or indicative values for pricing and minimum stay. For hosts using dynamic pricing systems such as PriceLabs, Beyond, or Wheelhouse, these values are floor estimates only — the actual nightly rate, applicable minimum stay, and gap-fill rules for any specific date window are determined at query time by the pricing engine. For static hosts managing their own rates, the listing's `pricing_rules` object provides a discount structure that must still be applied against the authoritative nightly rate for the requested window.

An agent cannot responsibly present a price or minimum stay to a user, or initiate a booking, without first querying the availability endpoint for the specific dates requested. This RFC defines that query and its response.

The availability query also determines whether a property is available at all for the requested dates — accounting for existing reservations, owner blocks, and platform holds — before proceeding to a booking request.

---

## 2. Prior Art

### 2.1 iCal / CalDAV

iCalendar (RFC 5545) is the de facto standard for calendar availability in the STR industry. However, iCal addresses only calendar state (available / blocked) and carries no pricing, fee, discount, or minimum stay information. It is a synchronisation format, not a query protocol.

### 2.2 OTA / OpenTravel Alliance

OTA defines XML-based availability request and response schemas. These are comprehensive but heavyweight, designed for enterprise B2B integrations and not suitable for lightweight agent-to-host queries.

### 2.3 Dynamic pricing provider APIs

PriceLabs, Beyond, and Wheelhouse each expose proprietary APIs for rate and minimum stay retrieval. These are platform-specific and not accessible to third-party agents without integration agreements. The OpenSTR availability endpoint acts as a normalisation layer — the host's system queries the pricing provider internally and returns a standardised response.

**Gap:** No open, lightweight standard exists for querying STR property availability, authoritative pricing including all fees and discounts, and dynamic minimum stay rules in a single request. This RFC defines that standard.

---

## 3. Design Decisions

### 3.1 Single endpoint, single date range

The availability query takes a single date range per request. Agents wishing to compare multiple date windows must make multiple requests. This keeps the request and response schema simple and the endpoint easy to implement. Batch availability queries are deferred to a future SEP.

### 3.2 Pricing authority

The availability response is the authoritative source of pricing for a given date window. Any `nightly_rate_indicative` value from the property listing must be disregarded once an availability response has been received for the same window.

### 3.3 Minimum stay authority

The availability response is the authoritative source of minimum stay for a given date window. For `dynamic` listings the response resolves gap-fill and seasonal rules as applied by the pricing system. For `static` listings it confirms the applicable minimum based on the listing's `min_stay` declaration.

### 3.4 Gap-fill rule surfacing

When a date window falls between two existing bookings and the pricing system applies a reduced minimum stay, the response explicitly identifies this via a `gap_fill` flag so the agent understands why the minimum differs from the listing default.

### 3.5 Complete pricing breakdown

The response returns a full pricing breakdown covering nightly rates, cleaning fee, pet fee, extra guest fee, all applicable discounts, and total. Fees that depend on request parameters — such as pet fee and extra guest fee — are conditionally required: if the conditions that trigger them are present in the request and the listing declares the fee, it must appear in the breakdown. The `total` must always reflect all applicable fees and discounts. This ensures agents can present fully transparent pricing to users.

### 3.6 Discount resolution

The response resolves and applies all applicable discounts in the following priority order: promotional discount (if active for the window) → length-of-stay discounts (trip-length and weekly/monthly) → last-minute discount. Only one discount type is applied per booking — the highest applicable value. The response declares which discount was applied and its amount.

### 3.7 Quote ID

The response issues a `quote_id` that must be referenced in the booking request. The host system validates the quote when processing the booking, confirming it was genuinely issued, has not expired, and matches the request parameters. This prevents fabricated or replayed quotes.

### 3.8 Rate limiting and caching

Availability data for dynamic listings can change frequently. The response includes `quote_expires_at` to guide agents. Agents must not cache availability responses beyond the indicated expiry.

---

## 4. Specification

### 4.1 Endpoint

```
POST /openstr/availability
Content-Type: application/json
```

### 4.2 Request Object

#### 4.2.1 Required Request Fields

| Field | Type | Description |
|---|---|---|
| `listing_id` | string | The `listing_id` from the property listing |
| `check_in` | string (ISO 8601 date) | Requested check-in date in `YYYY-MM-DD` format |
| `check_out` | string (ISO 8601 date) | Requested check-out date in `YYYY-MM-DD` format |
| `guests` | integer | Number of guests |

#### 4.2.2 Optional Request Fields

| Field | Type | Description |
|---|---|---|
| `pets` | integer | Number of pets, if applicable |
| `agent_id` | string (URI) | Identifier of the requesting agent, for rate limiting and analytics |
| `currency` | string | ISO 4217 currency code. If omitted, response uses listing's declared currency. |

#### 4.2.3 Request Validation

The host system must return a `400 Bad Request` if:
- `check_out` is not after `check_in`
- `check_in` is in the past
- `guests` exceeds `guests_max`
- The date range is less than 1 night

### 4.3 Response Object

#### 4.3.1 Available Response — HTTP 200

| Field | Type | Description |
|---|---|---|
| `listing_id` | string | Echo of requested `listing_id` |
| `available` | boolean | Always `true` in this response |
| `check_in` | string (ISO 8601 date) | Confirmed check-in date |
| `check_out` | string (ISO 8601 date) | Confirmed check-out date |
| `nights` | integer | Number of nights |
| `guests` | integer | Echo of requested guest count |
| `min_stay_applicable` | integer | Minimum stay applicable to this specific window as resolved by the pricing system |
| `gap_fill` | boolean | `true` if a gap-fill rule reduced the minimum stay for this window |
| `pricing` | object | Full pricing breakdown. See Section 4.4. |
| `policies_snapshot` | object | Snapshot of key policies at query time. See Section 4.5. |
| `quote_expires_at` | string (ISO 8601 datetime) | Timestamp after which this quote should be re-queried |
| `quote_id` | string | Unique identifier for this quote, required in the booking request |

#### 4.3.2 Unavailable Response — HTTP 200

A 200 status is used rather than 4xx because unavailability is an expected, non-error response.

| Field | Type | Description |
|---|---|---|
| `listing_id` | string | Echo of requested `listing_id` |
| `available` | boolean | Always `false` in this response |
| `check_in` | string (ISO 8601 date) | Echo of requested check-in |
| `check_out` | string (ISO 8601 date) | Echo of requested check-out |
| `reason` | enum | Reason for unavailability. See Section 4.3.3. |
| `next_available` | object | Optional. Nearest available window. See Section 4.6. |

#### 4.3.3 Unavailability Reason Vocabulary

| Value | Description |
|---|---|
| `dates_blocked` | Dates blocked by an existing reservation or owner block |
| `min_stay_not_met` | Requested range is shorter than the applicable minimum stay |
| `max_stay_exceeded` | Requested range exceeds the maximum stay |
| `guests_exceed_capacity` | Requested guest count exceeds property maximum |
| `advance_notice_required` | Check-in date is too soon — host requires more notice |
| `preparation_time` | Dates blocked due to turnaround period following an adjacent booking |

### 4.4 Pricing Breakdown Object

All monetary values are in the listing's declared currency unless the request specified a `currency` override.

#### Required pricing fields

| Field | Type | Description |
|---|---|---|
| `currency` | string | ISO 4217 currency code |
| `nightly_rate_average` | number | Average nightly rate across the stay, for display purposes |
| `nights` | integer | Number of nights |
| `subtotal_nights` | number | Total cost of nights before fees and discounts |
| `cleaning_fee` | number | Cleaning fee for this stay |
| `total` | number | Grand total including all fees and discounts. Must reflect all conditionally required fees below. |
| `pricing_mode` | enum | `static` or `dynamic` — echoes the listing's pricing mode for agent transparency |

#### Conditionally required pricing fields

These fields are required when the triggering condition is true.

| Field | Condition | Description |
|---|---|---|
| `pet_fee` | Request included `pets` > 0 and listing declares a `pet_fee` | Total pet fee for this stay |
| `extra_guest_fee` | `guests` exceeds the listing's `extra_guest_fee.above` threshold | Total extra guest fee for this stay |

#### Optional pricing fields

| Field | Type | Description |
|---|---|---|
| `nightly_breakdown` | array[object] | Per-night pricing for dynamic listings: `[{ "date": "YYYY-MM-DD", "rate": number }]` |
| `discount_applied` | object | Discount applied, if any. See Section 4.4.1. |
| `taxes` | object | `{ "rate_pct": number, "amount": number, "description": string }` |

#### 4.4.1 Discount Applied Object

Only one discount is applied per booking — the highest applicable value. Priority order: promotional → trip-length / weekly / monthly → last-minute.

| Field | Type | Description |
|---|---|---|
| `type` | enum | `weekly`, `monthly`, `trip_length`, `last_minute`, or `promotional` |
| `pct` | number | Percentage discount applied |
| `amount` | number | Monetary value of discount |
| `promotion_id` | string | Present only if `type` is `promotional` — references the listing's `promotion_id` |

### 4.5 Policies Snapshot Object

A snapshot of key policies at query time. Agents must use these values when presenting booking terms, not cached listing values.

| Field | Type | Description |
|---|---|---|
| `cancellation_policy` | enum | `flexible`, `moderate`, `strict`, `non_refundable` |
| `check_in_time` | string | `HH:MM` 24-hour format |
| `check_out_time` | string | `HH:MM` 24-hour format |
| `instant_book` | boolean | Whether this booking will be confirmed instantly |
| `damage_guarantee` | object | Current damage guarantee requirement |

### 4.6 Next Available Window Object

Optionally returned in an unavailable response to help agents suggest alternatives.

| Field | Type | Description |
|---|---|---|
| `check_in` | string (ISO 8601 date) | Next available check-in date |
| `check_out` | string (ISO 8601 date) | Earliest check-out given applicable minimum stay |
| `min_stay_nights` | integer | Minimum stay applicable to this window |

---

## 5. Error Responses

| HTTP Status | Meaning |
|---|---|
| `400 Bad Request` | Request validation failed. Response body includes `error` string. |
| `401 Unauthorized` | Authentication required but not provided. |
| `429 Too Many Requests` | Rate limit exceeded. Response includes `Retry-After` header. |
| `500 Internal Server Error` | Host system error. Agents should retry with exponential backoff. |

---

## 6. Examples

### 6.1 Request — Dynamic Host, 7-Night Stay with Pet

```json
{
  "listing_id": "host-abc123-prop-001",
  "check_in": "2026-07-04",
  "check_out": "2026-07-11",
  "guests": 2,
  "pets": 1,
  "currency": "USD"
}
```

### 6.2 Available Response

The host's system has queried PriceLabs, resolved a 3-night minimum, confirmed availability, and returned a per-night breakdown reflecting dynamic rates over the Fourth of July period. A weekly discount and pet fee are applied.

```json
{
  "listing_id": "host-abc123-prop-001",
  "available": true,
  "check_in": "2026-07-04",
  "check_out": "2026-07-11",
  "nights": 7,
  "guests": 2,
  "min_stay_applicable": 3,
  "gap_fill": false,
  "pricing": {
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
    "total": 1353.50,
    "nightly_breakdown": [
      { "date": "2026-07-04", "rate": 240 },
      { "date": "2026-07-05", "rate": 240 },
      { "date": "2026-07-06", "rate": 175 },
      { "date": "2026-07-07", "rate": 175 },
      { "date": "2026-07-08", "rate": 175 },
      { "date": "2026-07-09", "rate": 175 },
      { "date": "2026-07-10", "rate": 185 }
    ],
    "pricing_mode": "dynamic"
  },
  "policies_snapshot": {
    "cancellation_policy": "moderate",
    "check_in_time": "15:00",
    "check_out_time": "11:00",
    "instant_book": false,
    "damage_guarantee": {
      "mode": "idv_guarantee",
      "idv_level": "identity_verified"
    }
  },
  "quote_expires_at": "2026-02-24T18:00:00Z",
  "quote_id": "quote-20260224-abc123-0704"
}
```

### 6.3 Unavailable Response — Gap-Fill Scenario

A 1-night request falling in a gap between two bookings where the gap-fill minimum is 2 nights.

```json
{
  "listing_id": "host-abc123-prop-001",
  "available": false,
  "check_in": "2026-06-15",
  "check_out": "2026-06-16",
  "reason": "min_stay_not_met",
  "next_available": {
    "check_in": "2026-06-15",
    "check_out": "2026-06-17",
    "min_stay_nights": 2
  }
}
```

The agent can immediately re-query with the 2-night window returned in `next_available`, or present the guest with the corrected minimum stay without an additional round trip.

### 6.4 Request — Static Host with Last-Minute Discount

A static host with their own nightly rates and a last-minute discount declared in `pricing_rules`. The host system applies the discount directly without querying a third-party pricing provider.

```json
{
  "listing_id": "host-xyz789-prop-002",
  "check_in": "2026-02-26",
  "check_out": "2026-02-28",
  "guests": 2,
  "currency": "USD"
}
```

```json
{
  "listing_id": "host-xyz789-prop-002",
  "available": true,
  "check_in": "2026-02-26",
  "check_out": "2026-02-28",
  "nights": 2,
  "guests": 2,
  "min_stay_applicable": 2,
  "gap_fill": false,
  "pricing": {
    "currency": "USD",
    "nightly_rate_average": 120,
    "nights": 2,
    "subtotal_nights": 240,
    "cleaning_fee": 50,
    "discount_applied": {
      "type": "last_minute",
      "pct": 15,
      "amount": 36
    },
    "total": 254,
    "pricing_mode": "static"
  },
  "policies_snapshot": {
    "cancellation_policy": "flexible",
    "check_in_time": "14:00",
    "check_out_time": "10:00",
    "instant_book": true,
    "damage_guarantee": {
      "mode": "none"
    }
  },
  "quote_expires_at": "2026-02-24T20:00:00Z",
  "quote_id": "quote-20260224-xyz789-0226"
}
```

---

## 7. Agent Behaviour Guidelines

**7.1 Always query before booking.** Agents must not submit a booking request without first receiving a valid availability response with `available: true`. The `quote_id` is required in the booking request.

**7.2 Respect quote expiry.** Agents must check `quote_expires_at` before submitting a booking request. If expired, a fresh availability query must be made.

**7.3 Use response pricing, not listing pricing.** Once an availability response has been received, the listing's `nightly_rate_indicative` must be disregarded. The `total` from the pricing breakdown is authoritative.

**7.4 Use response minimum stay.** Once an availability response has been received, `min_stay_applicable` is authoritative for that window. Agents should note the `gap_fill` flag when presenting terms.

**7.5 Present all fees transparently.** Agents must present the full pricing breakdown to the user — including cleaning fee, pet fee, extra guest fee, and any discount — not just the total. Users must be able to see what they are paying for.

**7.6 Use policies_snapshot for booking terms.** Agents must present `policies_snapshot` values when seeking booking confirmation, not cached listing values.

**7.7 Handle next_available gracefully.** When an unavailable response includes `next_available`, agents should offer this as an alternative before moving to another listing.

**7.8 Respect rate limits.** Agents must honour `429` responses and `Retry-After` headers. A single query per user-initiated date selection is appropriate.

---

## 8. Security Considerations

**8.1 Quote ID integrity.** The host system must validate the `quote_id` when received in a booking request — confirming it was genuinely issued, has not expired, and matches the listing, dates, and guest count.

**8.2 Price consistency.** The host system must honour the `total` from a valid, unexpired availability response when referenced by `quote_id` in a booking request.

**8.3 Rate limiting.** Hosts are encouraged to rate limit the availability endpoint. Limits should permit at least 60 requests per hour per agent for legitimate use.

**8.4 HTTPS requirement.** The availability endpoint must be served over HTTPS.

---

## 9. Open Questions

**9.1 Batch availability queries.** A batch request format for multiple date windows in a single request is deferred to a future SEP.

**9.2 Multi-property queries.** Cross-property availability queries via a directory endpoint are deferred, related to rfc.property_listing.md open question 8.1.

**9.3 Seasonal pricing declaration in listing.** Should the listing optionally declare simplified seasonal rate bands to assist agents in pre-filtering before an availability query? Deferred to a future SEP.

**9.4 Tax handling.** Tax calculation rules vary significantly by jurisdiction. A tax extension defining jurisdiction-specific rules is deferred to a future SEP.

**9.5 Currency conversion.** The conversion mechanism and rate source for currency overrides are not standardised in v0.1 and are left to the host system's implementation.

**9.6 Discount stacking.** The current spec applies only the highest applicable discount. Whether discount stacking (e.g. weekly discount plus promotional discount) should be supported is an open question for community input.

---

## 10. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |
| 0.1.1-draft | February 2026 | Pet fee and extra guest fee made conditionally required in pricing breakdown; discount types expanded to include `last_minute`, `trip_length`, and `promotional`; static host example added; discount priority order defined; pricing breakdown `total` clarified to always reflect all applicable fees |

---

## 11. References

- OpenSTR `rfc.property_listing.md` — Property listing schema and discovery
- OpenSTR `rfc.booking_confirmation.md` — Booking request and confirmation flow
- OpenSTR `rfc.identity_trust.md` — Host and Guest Credential interfaces
- [RFC 5545 — iCalendar](https://datatracker.ietf.org/doc/html/rfc5545)
- [ISO 8601 — Date and time format](https://www.iso.org/iso-8601-date-and-time-format.html)
- [ISO 4217 — Currency codes](https://www.iso.org/iso-4217-currency-codes.html)
- [Agentic Commerce Protocol (ACP)](https://agenticcommerce.dev)
- [PriceLabs](https://pricelabs.co) — Example dynamic pricing provider
- [Beyond](https://beyondpricing.com) — Example dynamic pricing provider
- [Wheelhouse](https://usewheelhouse.com) — Example dynamic pricing provider
