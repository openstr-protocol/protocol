# Changelog

All notable changes to the OpenSTR protocol are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

---

## [0.2.0] — March 2026

### Breaking Changes

- **Discovery endpoint namespace migrated to `/.well-known/openstr/`** (RFC 8615 alignment)
  - Host manifest: `GET /.well-known/openstr` — new endpoint listing all listing IDs on a host domain
  - Listing document: `GET /.well-known/openstr/{listing_id}.json` (was `/.well-known/{listing_id}.json`)
  - Host credential: `GET /.well-known/openstr/{listing_id}/credential.json` (was `/.well-known/host-credential.json`)
  - Legacy v0.1 paths now return 404; implementations upgrading from v0.1 should serve 301 redirects for a minimum of six months
- **`listing_url` demoted from required to optional** in RFC-001 — a listing is fully valid without a human-readable URL
- **`openstr_version` value updated** from `"0.1"` to `"0.2"` in all listing documents

### Added

- **`rfc.property_listing.md` v0.2.0** — Property Listing Schema and Discovery
  - Section 4.1 rewritten: host manifest, listing document, and credential paths defined under `/.well-known/openstr/` namespace
  - Section 4.1.4: legacy path table with 301 redirect guidance
  - Section 4.2.3: `listing_id` conventions — host-assigned, opaque, URL path-safe; non-normative SHOULD recommendation for lowercase-alphanumeric-hyphen format; cross-protocol identifier role documented
  - `ucp_manifest_url` optional field added — enables UCP checkout hand-off from OpenSTR booking intent
  - Agent behaviour guideline 6.14: UCP hand-off behaviour
  - Open question 8.10: manifest caching and TTL (candidate for v0.3)
  - Section 2.5 updated: ACP replaced with UCP as the compatible commerce standard
  - RFC 8615 added to references

- **`rfc.identity_trust.md` v0.1.1** — Host and Guest Identity, Credentials, and Trust
  - Section 5.5: per-listing credential path convention documented (`/.well-known/openstr/{listing_id}/credential.json`)
  - Rationale for channel manager alignment: listing ID is the atomic unit CMs already manage
  - Multi-listing scope rule: one credential MAY cover multiple listings but MUST be reachable at each listing's per-listing path
  - Section 2.2: channel managers and PMS named explicitly as natural credential issuers at scale

- **`rfc.booking_confirmation.md` v0.1.1** — Booking Request, Confirmation, and Cancellation
  - Abstract: explicit scope boundary declaration — payment execution, post-booking lifecycle, and tourist tax remittance formally declared out of scope
  - Section 3.4 rewritten as "Payment Out of Scope": UCP, Stripe, AP2, and channel manager rails named as compatible options; `payment_token` declared optional
  - Section 3.9: post-payment calendar blocking documented as known gap with two v0.1 practical mitigations
  - Open question 11.7: three candidate v0.2 resolution approaches for post-payment calendar blocking
  - Section 2.1: ACP replaced with UCP in prior art

- **`openstr-availability-worker.js`** — availability.openstr.org
  - `handleManifest()`: new handler serving `GET /.well-known/openstr`
  - `handleListingDoc()`: renamed and updated from `handleWellKnown()`
  - `handleCredential()`: new stub handler serving `GET /.well-known/openstr/{listing_id}/credential.json`
  - All `X-OpenSTR-Version` headers updated to `0.2`

- **`openstr-demo-worker.js`** — demo.openstr.org
  - All three `/.well-known/openstr/` routes added
  - `handleManifest()` function added
  - Legacy flat routes retired

### Changed

- `README.md` — How It Works endpoint table expanded; design principles updated (ACP → UCP); Relationship to Other Standards table updated (ACP replaced with UCP; RFC 8615 added)
- `docs/channel-manager-integration.md` — Path conventions, example JSON, curl test flow, and endpoint reference table updated to v0.2 paths; protocol version header updated
- `docs/implementations.md` — Demo Worker listing and credential endpoint paths updated
- `docs/protocol-landscape.md` — UCP comparison table and coexistence paragraph updated
- `examples/listing-walled-garden-hut.json` — `openstr_version` updated to `"0.2"`; `listing_url` (Airbnb) removed; `credential_uri` updated to per-listing path
- All RFC cross-references standardised to `RFC-00N` canonical style throughout

### Infrastructure

- Smoke tested live 2026-03-31: manifest (200), listing document (200), credential stub (200), old path (404), availability (200 — unaffected)

---

## [0.1.4] — March 2026

### Added

- `rfc.property_listing.md` v0.1.4
  - Sub-type vocabulary expanded to 37 values
  - `check_in_time` split into `check_in_time_from` / `check_in_time_to` window
  - `policies.age_restrictions` array with freeform reason field
  - `policies.advance_notice` object
  - `policies.good_track_record_required` boolean
  - `policies.ev_charging_permitted` boolean
  - `policies.availability_window_months` field
  - `policies.house_rules` promoted to structured array
  - `pets.pet_restrictions` array
  - `pets.host_pets_on_property` boolean
  - Amenity vocabulary expanded (kitchen, bedroom, bathroom, outdoor items)
  - Environment tags expanded (`rural`, `countryside`, `historic`, `estate`, `farm`, `nature_reserve`, `village`)
  - Cancellation policy vocabulary corrected to `flexible`, `moderate`, `firm`, `strict`
  - Agent behaviour guidelines 6.9–6.13 added

---

## [0.1.3] — March 2026

### Added

- Security remediation Tier 2 (sessions C9–C11)
  - S-2.2: weak PRNG in `generateId()` — replaced with `crypto.getRandomValues()`
  - S-3.4: `listing_id` validation — permissive sanity check added to availability Worker
  - S-3.5: unawaited D1 write — `await` added
  - S-1.4 / S-3.1: SSRF on iCal fetch and index crawler — allowlist-based URL validation
  - S-2.1: LIKE wildcard bypass in index search — `escapeLike()` helper added
  - S-2.3: review fabrication — `verificationLevel` downgraded to `host_attested` for all v0.1 reviews

- `docs/protocol-landscape.md` — new document covering MCP, A2A, UCP, and AP2; `org.openstr.*` vendor namespace declaration; A2A binding roadmap
- `README.md` — Protocol Landscape section added; ACP references replaced with UCP/AP2

---

## [0.1.0] — February 2026

### Added

- `rfc.property_listing.md` v0.1.3 — Property Listing Schema and Discovery
  - Three-tier `property_classification` object replacing flat `property_type` field
  - `pricing_rules` object for static host discount and promotional structure
  - `safety_disclosures` object as required top-level field
  - `environment_tags` vocabulary with 14 standard values
  - `year_built` and `property_size` optional fields
  - Property sub-type vocabulary with 17 values
  - Section 2.2 — Property Classification Conventions documenting independent derivation of vocabulary

- `rfc.availability_query.md` v0.1.1 — Availability and Pricing Query
  - Pet fee and extra guest fee made conditionally required in pricing breakdown
  - Discount types expanded: `weekly`, `monthly`, `trip_length`, `last_minute`, `promotional`
  - Discount priority order defined
  - Static host example added (Section 6.4)

- `rfc.booking_confirmation.md` v0.1.0 — Booking Request, Confirmation, and Cancellation
  - Full booking request and confirmed response schemas
  - Instant book and request-to-book flows
  - Status polling endpoint for pending bookings
  - `location_detail` object — post-confirmation exact location disclosure
  - `access` object — check-in method and access code delivery
  - Cancellation endpoint with policy-based refund calculation
  - 24-hour request-to-book expiry

- `rfc.identity_trust.md` v0.1.0 — Host and Guest Identity, Credentials, and Trust
  - `GuestCredential` schema with three IDV tiers
  - `HostCredential` schema with property-level right-to-let verification
  - W3C Verifiable Credentials 2.0 envelope and Ed25519Signature2020 proof type
  - Bitstring Status List revocation mechanism
  - Trusted issuer registry governance model
  - HostCredential accreditation requirements
  - Bootstrap issuer rationale and two-entity model documented

- `README.md` — Protocol overview, design principles, scope, machine-to-machine vision, trust and certification model, compatibility table
- `docs/governance.md` v0.1.0 — Roles, RFC process, SEP process, versioning, contribution guidelines, trusted issuer registry governance
- `CONTRIBUTING.md` — Contribution guidelines and style guide
- `LICENSE` — Apache 2.0
