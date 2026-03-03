# OpenSTR Channel Manager Integration Guide

**Document version:** 0.1.0  
**Protocol version:** OpenSTR v0.1 (RFC-001 v0.1.4, RFC-002 v0.1.1, RFC-003 v0.1.0, RFC-004 v0.1.0)  
**Audience:** Property Management Systems (PMS) and channel managers  
**Status:** Draft — for feedback and implementation

---

## Overview

OpenSTR is an open protocol that makes short-term rental listings discoverable and bookable by AI agents — without a platform intermediary. This guide explains how a channel manager or PMS can expose their host inventory to AI agent traffic through the OpenSTR index.

A single channel manager integration makes every listing in your inventory agent-queryable overnight. This is the fastest path to scale for the protocol and the fastest path to AI agent traffic for your hosts.

---

## 1. The Architecture in Two Minutes

OpenSTR is a three-layer stack:

| Layer | What it is |
|---|---|
| **Protocol** | Open standard defined by RFCs at openstr.org |
| **Index** | Searchable registry at index.openstr.org — where AI agents query for available properties |
| **Endpoints** | Per-listing availability and booking endpoints that agents call after finding a property in the index |

Your channel manager sits between the index and your hosts' listings. You publish structured listing data to the index and expose availability and booking endpoints that agents can call in real time.

Agents do not browse the web to find listings. They query the index by location, dates, guest count, and amenity requirements. The index returns matching listings with their endpoint URLs. The agent then calls the availability endpoint to confirm pricing and availability before initiating a booking.

---

## 2. What You Need to Implement

Three things are required for a complete integration:

1. **RFC-001 listing JSON** — a structured, machine-readable description of each property
2. **An availability endpoint** — responds to RFC-002 availability queries with live pricing and a quote
3. **A booking stub endpoint** — accepts RFC-003 booking requests (full booking flow optional at this stage)

Additionally, each listing must be registered with the index (`POST /v1/register` at `index.openstr.org`).

---

## 3. RFC-001: The Listing Document

### 3.1 What it is

The listing document is a JSON file describing a single property. It is served at a stable HTTPS URL — conventionally `/.well-known/openstr.json` on your host's domain, or at a URL you control for listings you manage on behalf of hosts.

The full schema is defined in [RFC-001 (rfc.property_listing.md)](https://github.com/openstr-protocol/protocol). The minimum required fields are described below.

### 3.2 Minimum required fields

```json
{
  "listing_id": "your-listing-identifier",
  "listing_status": "active",
  "listing_name": "The Walled Garden Hut",
  "listing_description": "...",
  "property_classification": {
    "category": "unique_space",
    "sub_type": "shepherds_hut",
    "style": "rural_retreat"
  },
  "location": {
    "country": "GB",
    "region": "Norfolk",
    "place": "Snettisham",
    "latitude": 52.8573,
    "longitude": 0.4937,
    "approximate_location": true
  },
  "capacity": {
    "guests_max": 2,
    "bedrooms": 1,
    "beds": 1,
    "bathrooms": 1
  },
  "pricing": {
    "currency": "GBP",
    "nightly_rate": 150,
    "cleaning_fee": 35
  },
  "policies": {
    "check_in_time_from": "15:00",
    "check_in_time_to": "20:00",
    "check_out_time": "11:00",
    "min_nights": 2,
    "cancellation_policy": "moderate"
  },
  "safety_disclosures": {
    "has_co_detector": true,
    "has_smoke_detector": true
  },
  "host_did": "did:openstr:your-host-id"
}
```

### 3.3 Property classification vocabulary

The `property_classification` object uses a three-field taxonomy:

- **`category`** — top-level type: `entire_place`, `private_room`, `shared_room`, `unique_space`
- **`sub_type`** — more specific descriptor (see full vocabulary in RFC-001; 37 values including `cottage`, `apartment`, `villa`, `cabin`, `treehouse`, `shepherds_hut`, `yurt`, `boat`, `windmill`, and more)
- **`style`** — character descriptor: `beachfront`, `mountain_retreat`, `city_centre`, `rural_retreat`, `lakeside`, `garden_setting`, etc.

### 3.4 Mapping from your PMS data model

Most PMS data models map cleanly to RFC-001. The main translation points are:

| Your field | RFC-001 field |
|---|---|
| Property type | `property_classification.category` + `sub_type` |
| Max guests | `capacity.guests_max` |
| Nightly rate | `pricing.nightly_rate` |
| Check-in time | `policies.check_in_time_from` / `check_in_time_to` |
| Cancellation policy | `policies.cancellation_policy` (`flexible`, `moderate`, `firm`, `strict`) |
| Amenities | `amenities[]` — standard vocabulary in RFC-001 Section 6 |
| Location | `location.country`, `region`, `place`, `latitude`, `longitude` |

### 3.5 Generating listing_id values

The `listing_id` must be a stable, globally unique string. Recommended format:

```
{your-namespace}-{your-internal-id}
```

Examples:
- `guesty-prop-abc123`
- `hostaway-uk-listing-7842`
- `stayflow-property-00441`

Using your own namespace as a prefix avoids collisions and makes the origin transparent.

### 3.6 Serving the listing document

Each listing document must be served at a stable HTTPS URL. You have two options:

**Option A — Per-listing URL on your domain:**
```
https://listings.yourplatform.com/{listing_id}.json
```

**Option B — Well-known URL on the host's domain (if you manage their domain):**
```
https://{host-domain}/.well-known/openstr.json
```

Option A is recommended for channel managers managing many listings centrally.

---

## 4. Host Registration

### 4.1 Register a host account

Before registering listings, each host must have an account on the OpenSTR availability infrastructure. If you are managing listings on behalf of hosts, you can register on their behalf or register a single channel manager account and use the `intermediary` object in each listing registration.

```bash
curl -X POST https://availability.openstr.org/hosts/register \
  -H "Content-Type: application/json" \
  -d @register-host.json
```

**`register-host.json`:**
```json
{
  "display_name": "Acme Properties Ltd",
  "email": "openstr@acmeproperties.com",
  "country": "GB",
  "terms_accepted": true,
  "privacy_accepted": true
}
```

**Successful response (HTTP 201):**
```json
{
  "host_id": "host-abc123xyz",
  "host_did": "did:openstr:host-abc123xyz",
  "api_key": "ostr_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "message": "Host registered successfully. Store your API key securely — it will not be shown again."
}
```

> **Important:** Store the `api_key` value immediately and securely. It is shown only once and cannot be recovered.

---

## 5. Registering Listings with the Index

### 5.1 Registration endpoint

```
POST https://index.openstr.org/v1/register
Content-Type: application/json
```

### 5.2 Registration payload

```json
{
  "listing_id": "your-listing-identifier",
  "listing_endpoint": "https://listings.yourplatform.com/your-listing-identifier.json",
  "availability_endpoint": "https://availability.openstr.org/availability/your-listing-identifier",
  "booking_endpoint": "https://availability.openstr.org/booking/your-listing-identifier",
  "host_credential_uri": "",
  "contact_email": "openstr@yourplatform.com",
  "intermediary": {
    "intermediary_id": "your-channel-manager-id",
    "intermediary_name": "Your Channel Manager Name",
    "intermediary_credential_uri": ""
  }
}
```

> During the bootstrap period, `host_credential_uri` and `intermediary_credential_uri` may be left empty. Listings will register as `active_unverified`. See Section 8 on credentials.

**Successful response (HTTP 201):**
```json
{
  "index_id": "idx-abc123xyz",
  "listing_id": "your-listing-identifier",
  "status": "pending_validation",
  "registered_at": "2026-03-03T10:00:00Z",
  "first_crawl_scheduled": "2026-03-03T10:05:00Z",
  "message": "Registration received. Validation crawl scheduled within 5 minutes."
}
```

The index will crawl the `listing_endpoint` URL within a few minutes of registration and validate the listing JSON against the RFC-001 schema. If validation passes, the listing moves to `active_unverified` (or `active_verified` if a valid HostCredential is present).

### 5.3 Bulk registration

For large inventories, iterate over your listings and submit one registration request per listing. There is no batch registration endpoint in v0.1 — this is a candidate for a future SEP. Rate limiting applies; implement a modest delay between requests (recommended: 200ms).

Example shell loop:
```bash
for file in listings/*.json; do
  listing_id=$(jq -r '.listing_id' "$file")
  # Build registration payload and POST
  curl -X POST https://index.openstr.org/v1/register \
    -H "Content-Type: application/json" \
    -d @"registration-${listing_id}.json"
  sleep 0.2
done
```

---

## 6. Configuring Availability

### 6.1 The config endpoint

The availability config tells the OpenSTR availability infrastructure how to respond to RFC-002 queries for each listing. It stores pricing parameters and iCal feed URLs for real-time availability checking.

```
POST https://availability.openstr.org/config/{listing_id}
Authorization: Bearer ostr_YOUR_API_KEY
Content-Type: application/json
```

### 6.2 Config payload

```json
{
  "ical_feeds": [
    {
      "url": "https://www.airbnb.com/calendar/ical/YOUR_AIRBNB_ICAL_URL.ics",
      "source": "airbnb",
      "label": "Airbnb"
    },
    {
      "url": "https://www.vrbo.com/icalendar/YOUR_VRBO_ICAL_URL.ics",
      "source": "vrbo",
      "label": "VRBO"
    }
  ],
  "currency": "GBP",
  "nightly_rate": 150,
  "cleaning_fee": 35,
  "pet_fee": null,
  "extra_guest_fee": null,
  "extra_guest_above": null,
  "service_fee_pct": 0,
  "weekly_discount_pct": 10,
  "monthly_discount_pct": 20,
  "min_nights": 2,
  "cancellation_policy": "moderate",
  "checkin_time": "15:00",
  "checkout_time": "11:00",
  "listing_json": { ... }
}
```

> The `listing_json` field should contain the full RFC-001 listing object. This is the source used by the index crawler and served at the `/.well-known/` endpoint. It must exactly match the listing document you publish at `listing_endpoint`.

### 6.3 iCal feed merging

Multiple iCal feeds may be provided (Airbnb, VRBO, Booking.com, your own PMS, etc.). The availability infrastructure merges all feeds: any date blocked in any feed is treated as unavailable. This means your hosts' existing channel calendars are respected automatically — no double-booking risk.

**Supported `source` values:** `airbnb`, `vrbo`, `booking_com`, `tripadvisor`, `google_calendar`, `other`

### 6.4 Dynamic pricing

v0.1 supports static pricing (a fixed nightly rate with percentage discounts). Dynamic pricing — where the nightly rate varies by date — is a candidate for RFC-002 v0.2. If your PMS manages dynamic pricing, you have two options in the interim:

- Update the config nightly rate via the config endpoint when rates change (acceptable for simple rate management)
- Implement your own availability endpoint that returns RFC-002 compliant responses with your full dynamic pricing logic (recommended for sophisticated pricing)

---

## 7. The Availability Endpoint

### 7.1 Using the hosted availability infrastructure

If you use `availability.openstr.org` as your availability endpoint, the infrastructure handles RFC-002 query responses automatically based on your config. No additional implementation is required.

### 7.2 Self-hosting an availability endpoint

If you want full control over availability and pricing logic — recommended for channel managers with their own PMS — you can implement your own RFC-002 compliant endpoint and point the index registration at it.

**RFC-002 availability request:**
```
POST https://your-endpoint.com/availability/{listing_id}
Content-Type: application/json
```

```json
{
  "check_in": "2026-07-10",
  "check_out": "2026-07-17",
  "guests": 2,
  "pets": 1
}
```

**RFC-002 availability response (available):**
```json
{
  "listing_id": "your-listing-identifier",
  "available": true,
  "check_in": "2026-07-10",
  "check_out": "2026-07-17",
  "nights": 7,
  "guests": 2,
  "pets": 1,
  "pricing": {
    "mode": "static",
    "breakdown": {
      "nightly_rate": 150,
      "nights": 7,
      "nightly_total": 1050,
      "pet_fee": 30,
      "cleaning_fee": 35,
      "discount": {
        "type": "weekly",
        "percentage": 10,
        "amount": 105
      },
      "total": 1010,
      "currency": "GBP"
    },
    "total": 1010,
    "currency": "GBP"
  },
  "policies": {
    "cancellation_policy": "moderate",
    "checkin_time": "15:00",
    "checkout_time": "11:00",
    "min_nights": 2
  },
  "quote_id": "quote-abc123",
  "quote_expires_at": "2026-03-03T11:00:00Z"
}
```

**RFC-002 availability response (unavailable):**
```json
{
  "listing_id": "your-listing-identifier",
  "available": false,
  "check_in": "2026-07-10",
  "check_out": "2026-07-17",
  "reason": "dates_unavailable"
}
```

The `quote_id` is a reference token that should be included in any subsequent booking request. Quotes should expire after a reasonable period (30–60 minutes is typical).

---

## 8. Credentials and Trust Status

### 8.1 Bootstrap period

During the bootstrap period, listings register as `active_unverified`. This means they appear in index search results but are distinguished from `active_verified` listings. Agent implementations should surface credential status to users rather than treating it as a blocking condition.

> **Bootstrap note:** During the bootstrap period, listings register as `active_unverified`. Channel managers seeking to issue verifiable credentials to their host base can apply for trusted issuer accreditation — see Section 8.3.

### 8.2 What verification adds

A verified listing carries a `HostCredential` — a W3C Verifiable Credential that attests to:
- The host's identity (verified against government-issued documents or company registration)
- The host's legal right to let the specific property
- Property address verification against an authoritative source
- Local short-term let registration number where required by regulation

Verified listings display as `active_verified` in the index and will receive preferential ranking as agent implementations mature.

### 8.3 Becoming a trusted issuer

Channel managers and PMS providers are well-positioned to become accredited HostCredential issuers. You already hold KYC data and property records for your host base. Credential issuance is a natural extension of your existing onboarding process — and a revenue opportunity: hosts will pay for a portable, cryptographically verifiable credential that works across any OpenSTR-compatible platform.

**Accreditation requirements for HostCredential issuers (RFC-004 §7.2):**

*Identity verification:*
- Must verify individual host identity against a government-issued document, or business host identity against company registration documents
- Must implement liveness detection for individual identity checks

*Property verification — right-to-let:*
- Must verify the host's legal right to let each property (acceptable methods: title deeds, lease agreement with subletting clause, landlord consent letter, or property management agreement)
- Verification method must be recorded and declared in the credential
- Verification must be repeated if ownership or tenancy changes

*Property verification — address:*
- Must verify the property address against an authoritative source (national address register, land registry, or equivalent)

*Local registration:*
- Where required by local regulation (Edinburgh, Barcelona, Amsterdam, Paris, New York, etc.), must verify the short-term let registration or licence number before including it in the credential

*Technical requirements:*
- Must support W3C Verifiable Credentials with `Ed25519Signature2020` proof
- Must issue credentials conforming to the OpenSTR `HostCredential` schema
- Must implement Bitstring Status List revocation (mandatory for HostCredential issuers)
- Must refresh revocation status within 24 hours of a revocation event

*Governance:*
- Must accept OpenSTR's accreditation terms and conditions
- Must notify openstr.org within 48 hours of any security incident affecting credential integrity
- Must cooperate with OpenSTR dispute resolution processes

**To apply for trusted issuer accreditation:** Contact openstr.org via the GitHub repository at [github.com/openstr-protocol/protocol](https://github.com/openstr-protocol/protocol). Accredited issuers are listed at `https://openstr.org/trusted-issuers/host`.

---

## 9. Booking Stub (RFC-003)

### 9.1 Current state

RFC-003 defines the full booking confirmation flow. In v0.1, the booking endpoint acts as a stub that acknowledges the request and returns a `pending` status. Full booking processing (payment, confirmation, guest messaging) will be standardised in subsequent RFC iterations.

If you use `availability.openstr.org` as your booking endpoint, the stub is already live. It accepts RFC-003 compliant requests and returns a `pending_review` response.

### 9.2 Full booking endpoint implementation

To handle bookings end-to-end, implement:

```
POST https://your-endpoint.com/booking/{listing_id}
Content-Type: application/json
```

**Booking request:**
```json
{
  "quote_id": "quote-abc123",
  "check_in": "2026-07-10",
  "check_out": "2026-07-17",
  "guests": 2,
  "pets": 1,
  "guest_credential_uri": "https://identity.example.com/credentials/guest-7x9k2",
  "total_price": 1010,
  "currency": "GBP",
  "payment_reference": "pay_stripe_abc123",
  "guest_message": "We are looking forward to our stay."
}
```

**Booking response (instant book):**
```json
{
  "booking_id": "bkg-abc123xyz",
  "listing_id": "your-listing-identifier",
  "status": "confirmed",
  "check_in": "2026-07-10",
  "check_out": "2026-07-17",
  "confirmed_at": "2026-03-03T10:00:00Z",
  "location_detail": {
    "full_address": "The Walled Garden Hut, Garden Lane, Snettisham, PE31 7NQ",
    "what3words": "///example.words.here"
  },
  "access": {
    "method": "key_safe",
    "instructions": "Key safe is on the left of the front gate. Code will be sent 24 hours before check-in."
  }
}
```

---

## 10. Keeping Listings Fresh

### 10.1 The index crawler

The OpenSTR index crawler re-validates each registered listing periodically (approximately every 24 hours). It fetches the listing document from `listing_endpoint` and validates it against the RFC-001 schema. Listings that fail validation are marked `invalid` and removed from agent query results. Hosts are notified via their `contact_email`.

**Implication for channel managers:** Ensure your listing endpoint always returns valid RFC-001 JSON. Schema validation errors will suppress listings from agent results.

### 10.2 Updating listing data

When a host updates their listing (rates, description, amenities, capacity), update both:
1. The listing document at `listing_endpoint` (the crawler will pick this up on the next crawl)
2. The `listing_json` field in the availability config (via `POST /config/{listing_id}`) — this keeps the availability infrastructure in sync immediately

The config update should be triggered on any change to pricing, policies, or listing content.

### 10.3 Removing a listing

To remove a listing from the index, set `listing_status` to `inactive` in the listing document. The crawler will update the index status on the next crawl. There is no explicit deregister endpoint in v0.1.

---

## 11. Testing Your Integration

### 11.1 Validate your listing JSON

A public validator is planned at `https://openstr.org/validate`. Until it is available, validate against the RFC-001 JSON schema in the [protocol repository](https://github.com/openstr-protocol/protocol).

### 11.2 End-to-end test flow

**Step 1 — Serve your listing document:**
```bash
curl https://your-endpoint.com/.well-known/your-listing-id.json
```
Expected: valid RFC-001 JSON with all required fields.

**Step 2 — Submit a test availability query:**
```bash
curl -X POST https://availability.openstr.org/availability/your-listing-id \
  -H "Content-Type: application/json" \
  -d '{"check_in": "2026-07-10", "check_out": "2026-07-14", "guests": 2}'
```
Expected: RFC-002 response with `available: true` or `available: false` and a pricing breakdown.

**Step 3 — Verify the listing appears in the index:**
```bash
curl -X POST https://index.openstr.org/v1/search \
  -H "Content-Type: application/json" \
  -d '{"location": {"country": "GB", "region": "Norfolk"}, "guests": 2}'
```
Expected: your listing in the results array.

**Step 4 — Fetch listing detail from the index:**
```bash
curl https://index.openstr.org/v1/listings/{index_id}
```
Expected: full listing detail including status (`active_unverified` or `active_verified`).

---

## 12. Reference: Key Endpoints

| Operation | Method | URL |
|---|---|---|
| Register host | POST | `https://availability.openstr.org/hosts/register` |
| Update host profile | PUT | `https://availability.openstr.org/hosts/{host_id}` |
| Set availability config | POST | `https://availability.openstr.org/config/{listing_id}` |
| View availability config | GET | `https://availability.openstr.org/config/{listing_id}` |
| Register listing with index | POST | `https://index.openstr.org/v1/register` |
| Search index | POST | `https://index.openstr.org/v1/search` |
| Get listing detail | GET | `https://index.openstr.org/v1/listings/{index_id}` |
| Availability query | POST | `https://availability.openstr.org/availability/{listing_id}` |
| Listing document | GET | `https://availability.openstr.org/.well-known/{listing_id}.json` |
| Booking stub | POST | `https://availability.openstr.org/booking/{listing_id}` |
| Health check | GET | `https://availability.openstr.org/health` |

---

## 13. Getting Help

- **Protocol specification:** [github.com/openstr-protocol/protocol](https://github.com/openstr-protocol/protocol)
- **Reference index:** [index.openstr.org](https://index.openstr.org)
- **Demo environment:** [demo.openstr.org](https://demo.openstr.org)
- **Trusted issuer registry:** [openstr.org/trusted-issuers](https://openstr.org/trusted-issuers)
- **Issues and questions:** GitHub issues at the protocol repository

---

*OpenSTR is an open protocol published under the Apache 2.0 licence. This document is provided for implementers and is subject to change as the protocol matures.*
