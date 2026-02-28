# RFC: Discovery Index

**RFC ID:** openstr-rfc-005  
**Title:** OpenSTR Discovery Index  
**Status:** Draft  
**Version:** 0.1.0  
**Created:** February 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Depends on:** openstr-rfc-001 (Property Listing), openstr-rfc-004 (Identity Trust)  

---

## Abstract

This RFC defines the OpenSTR Discovery Index — a specialised, machine-readable registry of OpenSTR-compliant property listings designed to be queried by AI agents. The index solves the discovery problem that the core protocol leaves unaddressed: the `/.well-known/openstr.json` convention makes individual listings machine-readable, but provides no mechanism for an agent to find listings it does not already know about.

The index is a purpose-built intermediary layer, analogous to a Global Distribution System (GDS) for short-term rentals. It crawls known OpenSTR endpoints, validates listings against the protocol schema and host credentials, and exposes a structured query API that agents can use to search by location, dates, guest count, amenities, price, and credential status. It also maintains a portable host reputation layer — aggregating verified review events linked to host Decentralised Identifiers (DIDs) — so that an agent can assess a host's track record without relying on any single platform's review system.

This RFC defines the index registration mechanism, the crawling and validation model, the agent query API, and the host reputation schema.

---

## 1. Motivation and Problem Statement

### 1.1 The discovery gap

The four core OpenSTR RFCs define how a property is described (RFC-001), how availability is queried (RFC-002), how a booking is confirmed (RFC-003), and how trust is established (RFC-004). Together they enable an agent to interact with a property it has already found. They do not define how the agent finds the property in the first place.

Publishing a `/.well-known/openstr.json` file makes a property machine-readable. It does not make it discoverable. A general-purpose web crawler such as Google does not know to look for OpenSTR endpoints, does not understand the structured data they contain, and cannot answer structured queries about availability, pricing, or credential status. A small independent host with one property and no search engine optimisation budget is invisible to an agent performing a general web search.

The result is that the practical path to agent-native discovery leads back to the same closed platforms and proprietary APIs that OpenSTR is designed to provide an alternative to. The protocol layer is complete; the discovery layer is missing.

### 1.2 The reputation gap

Host verification, guest identity, and review history are currently siloed inside each platform. A host with a five-year track record on one platform carries none of that reputation when they list elsewhere. This is not merely a short-term rental problem — it reflects a broader structural issue in digital identity where credentials and reputation are assets held by platforms rather than by the individuals who earned them.

W3C Decentralised Identifiers (DIDs) and Verifiable Credentials provide the cryptographic foundation for portable identity. OpenSTR RFC-004 defines the HostCredential — a verifiable credential proving host identity and right-to-let. The natural extension is a reputation layer: verified review events, cryptographically linked to host and guest DIDs, aggregated by the index and queryable by agents independently of any platform.

### 1.3 The aggregation model

A realistic open standard for short-term rentals cannot expect every independent host to run and maintain their own API infrastructure. The hotel industry solved this through channel managers — software intermediaries that connect hotel inventory to multiple distribution channels. The airline industry solved this through GDS networks. In both cases, the individual operator connects to an intermediary, and the intermediary presents their inventory to agents via a standardised interface.

The OpenSTR index is the equivalent layer for short-term rentals. A host who cannot or does not want to run their own OpenSTR endpoint can be represented in the index through a compliant intermediary — a Property Management System (PMS), a channel manager, or a host management tool — that publishes and maintains the endpoint on their behalf. The index treats host-direct endpoints and intermediary-published endpoints equivalently, subject to credential validation.

---

## 2. Prior Art

### 2.1 Global Distribution Systems (GDS)

Amadeus, Sabre, and Travelport are the dominant GDS networks for flights and hotels. They are standardised intermediary registries that travel agents (human and AI) query to search and book inventory. GDS networks are not open — access requires commercial agreements and fees — but they demonstrate the model: a specialist index, queried by agents via a structured API, presenting inventory from many suppliers in a normalised format.

OpenSTR defines an open equivalent: no commercial agreements required, no fees for index access, governed by the same open protocol as the listings it indexes.

### 2.2 DNS

The Domain Name System is a distributed registry mapping human-readable domain names to machine-readable IP addresses. It is queried billions of times daily, is invisible to end users, and is governed by open standards. The OpenSTR index shares the same design intent: a specialist registry, queried by machines, serving a specific structured lookup function.

### 2.3 npm / package registries

Software package registries (npm, PyPI, crates.io) index published packages and respond to structured queries from package managers. A package manager querying npm for `lodash@4.17.21` receives structured metadata it can act on directly — version, dependencies, download URL — without visiting a human-readable page. The OpenSTR index operates on the same principle for property listings.

### 2.4 Schema.org / structured data

Schema.org defines a vocabulary for embedding structured data in web pages, primarily for SEO benefit. Google and other search engines consume this data and may surface it in rich results. This provides some structured discoverability for properties that embed Schema.org markup, but it is dependent on Google's indexing and ranking decisions, returns results designed for human consumption rather than agent consumption, and cannot answer structured queries about availability or credential status.

**Gap:** No open, agent-queryable registry exists for short-term rental properties. This RFC defines that registry.

---

## 3. Design Decisions

### 3.1 Push registration, not pure crawling

The index does not crawl the open web looking for `/.well-known/openstr.json` files. Instead, hosts (or their intermediaries) register their listing endpoints explicitly with the index. The index then crawls registered endpoints periodically to refresh listing data and verify continued compliance.

This design decision reflects two realities. First, the open web is too large and noisy for an underfunded crawler to find the small number of OpenSTR endpoints within it efficiently. Second, registration creates an explicit record of intent — a host who registers with the index is signalling that they want to be discoverable by agents. Unregistered endpoints remain valid OpenSTR implementations; they simply are not indexed.

This may evolve. If OpenSTR adoption reaches sufficient scale, a purpose-built web crawler scanning for `/.well-known/openstr.json` files across all registered domains becomes feasible. That extension is deferred to a future SEP.

### 3.2 Validation at registration and on refresh

The index validates each registered listing against the OpenSTR property listing schema at the time of registration and on each subsequent crawl. Listings that fail validation are marked as `invalid` in the index and excluded from agent query results. Hosts are notified of validation failures via the contact address in their HostCredential.

### 3.3 Credential verification

The index verifies each listing's HostCredential at registration. This involves:
- Fetching the credential from the `credential_uri` declared in the listing
- Verifying the credential's cryptographic proof against the issuer's published public key
- Confirming the issuer is on the OpenSTR trusted issuer registry
- Confirming the credential's `valid_until` has not passed
- Confirming the credential's `listing_id` matches the registered listing

Listings with valid, verified credentials are marked `verified` in the index. Listings with unverified or absent credentials are marked `unverified` and may be queried but are distinguished from verified listings in all agent responses.

### 3.4 Availability is not cached

The index does not cache or store availability data. When an agent queries the index for available properties on specific dates, the index returns a list of candidate listings — properties whose listing data matches the query parameters (location, capacity, amenities, price range, credential status). The agent is then responsible for querying each candidate's availability endpoint directly to confirm actual availability.

This design keeps availability data authoritative at the source and avoids the index becoming a stale intermediary for time-sensitive booking data. It mirrors how a GDS works: the GDS returns fare options, but the airline's own reservation system confirms the booking.

### 3.5 Host reputation is stored in the index

Unlike availability, host reputation data is aggregated and stored by the index. Review events — submitted by guests with verified credentials following a completed booking — are linked to the host's DID and accumulated over time. This is the index's most valuable long-term asset: a portable, platform-independent reputation layer that hosts own and carry regardless of which channel their bookings come through.

The reputation model is defined in Section 6.

### 3.6 The index is queryable, not browsable

The index exposes a structured JSON API. It does not provide a human-facing website or search interface. Human-facing search and booking interfaces are a separate concern, to be built on top of the index API by third parties. This maintains the index's role as infrastructure rather than product.

### 3.7 Open access, rate limited

The index query API is publicly accessible without authentication for read operations. Rate limiting applies to prevent abuse. Write operations (registration, review submission) require a valid HostCredential or GuestCredential respectively.

---

## 4. Index Registration

### 4.1 Registration endpoint

```
POST https://index.openstr.org/v1/register
Content-Type: application/json
```

### 4.2 Registration request

| Field | Type | Required | Description |
|---|---|---|---|
| `listing_endpoint` | string (URI) | Yes | The full URI of the `/.well-known/openstr.json` endpoint |
| `availability_endpoint` | string (URI) | Yes | The full URI of the availability endpoint |
| `booking_endpoint` | string (URI) | Yes | The full URI of the booking endpoint |
| `host_credential_uri` | string (URI) | Yes | The URI of the host's HostCredential |
| `contact_email` | string | Yes | Contact address for validation failure notifications |
| `intermediary` | object | No | Present if the listing is published by a PMS or channel manager on the host's behalf |

#### Intermediary object

| Field | Type | Required | Description |
|---|---|---|---|
| `intermediary_id` | string | Yes | The intermediary's registered OpenSTR identifier |
| `intermediary_name` | string | Yes | Human-readable name of the intermediary |
| `intermediary_credential_uri` | string (URI) | Yes | The intermediary's own HostCredential URI |

### 4.3 Registration response

A successful registration returns HTTP 201 with:

```json
{
  "index_id": "idx-chi-001-abc123",
  "listing_id": "host-demo-chicago-001",
  "status": "pending_validation",
  "credential_status": "pending_verification",
  "registered_at": "2026-03-01T10:00:00Z",
  "first_crawl_scheduled": "2026-03-01T10:05:00Z",
  "message": "Registration received. Validation crawl scheduled within 5 minutes."
}
```

### 4.4 Registration lifecycle

| Status | Meaning |
|---|---|
| `pending_validation` | Registered, awaiting first crawl |
| `active_verified` | Crawled, schema valid, credential verified |
| `active_unverified` | Crawled, schema valid, credential absent or unverifiable |
| `invalid` | Crawled, schema validation failed |
| `unreachable` | Endpoint not responding after 3 consecutive crawl attempts |
| `suspended` | Removed from query results pending review |
| `withdrawn` | Host has requested removal from index |

---

## 5. Crawling and Validation

### 5.1 Crawl schedule

The index crawls each registered endpoint on the following schedule:

| Listing status | Crawl frequency |
|---|---|
| `active_verified` | Every 24 hours |
| `active_unverified` | Every 6 hours |
| `invalid` | Every 1 hour for 24 hours, then every 24 hours |
| `unreachable` | Every 1 hour for 3 attempts, then suspended |

### 5.2 Crawl behaviour

Each crawl:
1. Fetches `/.well-known/openstr.json` via HTTP GET
2. Validates the response against the OpenSTR property listing schema (RFC-001)
3. Re-verifies the HostCredential if it has changed or is approaching expiry
4. Updates the index record with the refreshed listing data
5. Updates the listing status accordingly

The index crawler identifies itself with the User-Agent string:
```
OpenSTR-Index-Crawler/0.1 (+https://index.openstr.org/crawler)
```

### 5.3 Schema validation rules

A listing passes schema validation if:
- The response is valid JSON
- `openstr_version` is present and a supported version
- `listing_id` is present and matches the registered value
- `property_classification` contains valid `category`, `sub_type`, and `occupancy_type` values
- `capacity.guests` is a positive integer
- `location.country` is a valid ISO 3166-1 alpha-2 code
- `location.timezone` is a valid IANA timezone identifier
- `pricing.currency` is a valid ISO 4217 currency code
- `availability_endpoint` and `booking_endpoint` are valid URIs
- `host_credential.credential_uri` is a valid URI

All other fields are validated as present if declared required in RFC-001. Optional fields are validated for type correctness if present.

---

## 6. Agent Query API

### 6.1 Search endpoint

```
POST https://index.openstr.org/v1/search
Content-Type: application/json
```

### 6.2 Search request

| Field | Type | Required | Description |
|---|---|---|---|
| `location` | object | Yes | Geographic search parameters |
| `check_in` | string (ISO 8601 date) | No | Requested check-in date. If provided, `check_out` is also required |
| `check_out` | string (ISO 8601 date) | No | Requested check-out date |
| `guests` | integer | No | Number of guests. Filters by `capacity.guests` |
| `pets` | integer | No | Number of pets. Filters to listings with `pets_allowed: true` |
| `amenities` | array of strings | No | Required amenities. Returns only listings declaring all specified amenities |
| `price_max` | number | No | Maximum nightly rate from listing indicative pricing |
| `property_category` | string | No | Filter by property classification category |
| `credential_status` | string | No | `verified`, `unverified`, or `any` (default: `any`) |
| `min_idv_level` | string | No | Minimum IDV level declared in HostCredential |
| `page` | integer | No | Page number for paginated results (default: 1) |
| `page_size` | integer | No | Results per page, max 50 (default: 20) |

#### Location object

| Field | Type | Required | Description |
|---|---|---|---|
| `country` | string | Yes | ISO 3166-1 alpha-2 country code |
| `region` | string | No | State, county, or region name |
| `place` | string | No | City, town, or locality name |
| `lat` | number | No | Centre latitude for radius search |
| `lng` | number | No | Centre longitude for radius search |
| `radius_km` | number | No | Search radius in kilometres (default: 25 if lat/lng provided) |

### 6.3 Search response

```json
{
  "query_id": "qry-abc123",
  "total_results": 14,
  "page": 1,
  "page_size": 20,
  "results": [
    {
      "index_id": "idx-nfk-001-xyz789",
      "listing_id": "host-norfolk-beachcottage-001",
      "listing_endpoint": "https://norfolkcottage.example.com/.well-known/openstr.json",
      "availability_endpoint": "https://norfolkcottage.example.com/openstr/availability",
      "booking_endpoint": "https://norfolkcottage.example.com/openstr/booking",
      "credential_status": "verified",
      "last_crawled": "2026-03-01T08:00:00Z",
      "summary": {
        "title": "Beach Cottage — 2BR, North Norfolk",
        "property_category": "cottage",
        "occupancy_type": "entire_place",
        "location": {
          "place": "Burnham Market",
          "region": "Norfolk",
          "country": "GB",
          "approximate_lat": 52.8974,
          "approximate_lng": 0.7380
        },
        "capacity": {
          "guests": 4,
          "bedrooms": 2
        },
        "amenities_highlight": ["wifi", "garden", "pets_allowed", "parking"],
        "nightly_rate_indicative": 185.00,
        "currency": "GBP",
        "pets_allowed": true
      },
      "reputation": {
        "review_count": 12,
        "average_rating": 4.8,
        "verified_review_count": 12,
        "host_since": "2023-06-01"
      }
    }
  ],
  "note": "Results show listings matching query parameters. Availability for specific dates must be confirmed by querying each listing's availability_endpoint directly."
}
```

### 6.4 Listing detail endpoint

```
GET https://index.openstr.org/v1/listings/{index_id}
```

Returns the full cached listing data from the last crawl, plus the listing's full reputation record. Does not query the live endpoint — agents requiring live data should fetch the `listing_endpoint` directly.

### 6.5 Batch availability hint

An optional convenience endpoint for agents who want to pre-filter candidate listings by availability before querying each endpoint individually:

```
POST https://index.openstr.org/v1/availability-hint
```

The index queries each candidate listing's availability endpoint in parallel and returns a filtered list. This is a best-effort service — the index does not guarantee availability data is current, and the authoritative source remains the listing's own availability endpoint. Agents must confirm with the listing directly before proceeding to a booking.

---

## 7. Host Reputation

### 7.1 Design principles

The host reputation model is built on three principles:

**Portability.** Reputation is linked to the host's DID, not to any platform or channel. A host who has received verified reviews through OpenSTR-compliant bookings carries that reputation to any new listing or platform that queries the OpenSTR index.

**Verifiability.** Every review event must be signed by a guest with a valid, verified GuestCredential. Anonymous or unverifiable reviews are not accepted. This prevents fake reviews and links each review to a real, verified identity.

**Immutability.** Submitted review events are append-only. Neither the host nor the index operator can delete a submitted review. A host can submit a public response to any review, which is stored alongside it. This maintains integrity and trust in the reputation record.

### 7.2 Review event submission

Reviews are submitted by guests following a completed booking.

```
POST https://index.openstr.org/v1/reviews
Content-Type: application/json
```

| Field | Type | Required | Description |
|---|---|---|---|
| `booking_reference` | string | Yes | The OpenSTR booking reference |
| `listing_id` | string | Yes | The listing ID |
| `host_did` | string | Yes | The host's DID from the HostCredential |
| `guest_credential` | object | Yes | The guest's GuestCredential for identity verification |
| `check_in` | string | Yes | Actual check-in date |
| `check_out` | string | Yes | Actual check-out date |
| `ratings` | object | Yes | Structured ratings (see below) |
| `review_text` | string | No | Free-text review, max 2000 characters |
| `private_feedback` | string | No | Private message to host, not published |

#### Ratings object

| Field | Type | Required | Range | Description |
|---|---|---|---|---|
| `overall` | number | Yes | 1.0–5.0 | Overall experience |
| `accuracy` | number | Yes | 1.0–5.0 | Listing accuracy |
| `cleanliness` | number | Yes | 1.0–5.0 | Property cleanliness |
| `communication` | number | Yes | 1.0–5.0 | Host communication |
| `location` | number | No | 1.0–5.0 | Location satisfaction |
| `value` | number | Yes | 1.0–5.0 | Value for money |

Ratings are to one decimal place. Half-star increments (0.5) are the minimum granularity.

### 7.3 Review validation

The index validates each submitted review:
1. Verifies the `guest_credential` is valid and issued by a trusted credential provider
2. Confirms the `booking_reference` corresponds to a booking event logged by the host's booking endpoint (the index requests confirmation from the host's booking endpoint via a signed callback)
3. Confirms the check-out date has passed before accepting the review
4. Applies a one-review-per-booking rule — a guest may not submit more than one review per booking reference

Reviews that fail validation are rejected with a structured error response.

### 7.4 Reputation record structure

The index maintains a reputation record for each registered host DID:

```json
{
  "host_did": "did:example:host-norfolk-001",
  "host_since": "2023-06-01",
  "total_reviews": 24,
  "verified_reviews": 24,
  "average_ratings": {
    "overall": 4.8,
    "accuracy": 4.9,
    "cleanliness": 4.7,
    "communication": 5.0,
    "location": 4.6,
    "value": 4.8
  },
  "review_history": [
    {
      "review_id": "rev-abc123",
      "listing_id": "host-norfolk-beachcottage-001",
      "check_in": "2026-02-10",
      "check_out": "2026-02-14",
      "ratings": {
        "overall": 5.0,
        "accuracy": 5.0,
        "cleanliness": 5.0,
        "communication": 5.0,
        "value": 5.0
      },
      "review_text": "Wonderful cottage. Exactly as described.",
      "submitted_at": "2026-02-15T09:00:00Z",
      "guest_credential_issuer": "verified-identity.example.com",
      "host_response": null
    }
  ]
}
```

### 7.5 Host response

A host may submit a public response to any review within 30 days of submission:

```
POST https://index.openstr.org/v1/reviews/{review_id}/response
```

The response is stored in the review record and returned to agents querying the reputation record. Responses are limited to 500 characters.

### 7.6 Cross-platform reputation

Hosts are encouraged to submit review events from bookings made through other channels — Airbnb, direct booking websites, and similar — provided those reviews are verifiable. Cross-platform review submission requires:
- A booking confirmation document (PDF or structured data) from the originating platform
- A signed attestation from the host that the booking occurred as described
- The guest's consent and GuestCredential if available

Cross-platform reviews submitted without a guest GuestCredential are stored but marked `host_attested` rather than `guest_verified`, and are distinguished in agent responses. This allows hosts to build a reputation record from existing bookings while maintaining transparency about verification level.

---

## 8. Index Operator Responsibilities

An entity operating an OpenSTR-compliant index must:

- Make the registration, search, listing detail, and review endpoints publicly accessible
- Publish their crawler User-Agent string at the path referenced in the User-Agent header
- Process registration requests within 10 minutes during normal operation
- Complete validation crawls within 5 minutes of registration
- Refresh all active listings within a 24-hour window
- Publish an uptime SLA and status page
- Provide a mechanism for hosts to withdraw their listing from the index on request, processing withdrawals within 1 hour
- Publish their data retention policy
- Not sell or share individual listing or reputation data with third parties without explicit consent
- Comply with applicable data protection law in the jurisdictions where they operate

Multiple independent index operators may exist. Agents may query multiple indexes and merge results. Index operators are encouraged but not required to share crawl data with each other. Federation between index operators is deferred to a future SEP.

---

## 9. Privacy Considerations

### 9.1 Location precision

The index stores and returns only the `approximate_lat` and `approximate_lng` values from the property listing. Precise addresses are not stored in the index and are not returned in search responses. Precise location is disclosed by the host's own booking endpoint after booking confirmation, as defined in RFC-003.

### 9.2 Guest data

The index does not store guest personal data beyond what is necessary to validate review submissions. Guest DID references in review records are one-way hashed before storage so that a guest's review history cannot be reconstructed from the index alone.

### 9.3 Host DID

The host's DID is a public identifier by design — it is the mechanism by which reputation is portable. Hosts accept that their DID and associated reputation record are publicly queryable when registering with the index.

---

## 10. Security Considerations

### 10.1 Credential forgery

The index verifies HostCredentials cryptographically against the issuer's published public key. A forged or self-issued credential without a valid proof from a trusted issuer will fail verification and the listing will be marked `unverified`.

### 10.2 Review manipulation

Review manipulation is mitigated by:
- Requiring a verified GuestCredential for every review submission
- The one-review-per-booking rule
- Booking confirmation via signed callback to the host's booking endpoint
- Immutability of submitted reviews

### 10.3 Endpoint hijacking

If a host's domain is compromised and the OpenSTR endpoint replaced with a malicious one, the index will detect schema validation failures on the next crawl and mark the listing `invalid`. The HostCredential is tied to the host's DID, not the domain, so a domain compromise does not allow an attacker to impersonate the host's credential.

### 10.4 Index abuse

The registration endpoint is rate-limited per IP address and per HostCredential to prevent bulk registration of invalid listings. Listings are validated before being made queryable.

---

## 11. Open Questions

**11.1 Federation.** Should multiple index operators federate — sharing crawl data and presenting a unified query surface to agents? A federated model improves resilience and avoids single points of failure. The complexity of federation governance is deferred to a future SEP.

**11.2 Automated crawling.** Should the index additionally crawl all registered domains for `/.well-known/openstr.json` files, independent of explicit registration? This would catch listings published by intermediaries who have not registered individually. Deferred to a future SEP once adoption reaches sufficient scale.

**11.3 Guest reputation.** This RFC defines host reputation. An equivalent guest reputation system — allowing hosts to verify that a prospective guest is trustworthy — is a natural complement. Deferred to a future SEP given the additional privacy complexity.

**11.4 Review dispute resolution.** The immutability principle prevents review deletion but does not address the case where a review contains false factual claims. A dispute resolution process for contested reviews is deferred to a future SEP.

**11.5 Cross-platform review verification.** The mechanism for verifying reviews from non-OpenSTR bookings (Section 7.6) relies on host attestation for cases where no GuestCredential is available. A more robust cross-platform verification mechanism — perhaps involving signed booking confirmations from originating platforms — is deferred to a future SEP.

**11.6 Index economics.** This RFC defines index operator responsibilities but does not define a revenue or sustainability model for index operators. A sustainable index requires funding — whether through registration fees, query fees, or other mechanisms — without creating the gatekeeping dynamics that OpenSTR is designed to avoid. This is an open governance question deferred to a future SEP.

---

## 12. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |

---

## 13. References

- openstr-rfc-001: Property Listing
- openstr-rfc-002: Availability Query
- openstr-rfc-003: Booking Confirmation
- openstr-rfc-004: Identity Trust
- W3C Decentralised Identifiers (DIDs) v1.0 — https://www.w3.org/TR/did-core/
- W3C Verifiable Credentials Data Model — https://www.w3.org/TR/vc-data-model/
- RFC 5545 — iCalendar
- IANA Timezone Database — https://www.iana.org/time-zones
- ISO 3166-1 — Country Codes
- ISO 4217 — Currency Codes
