# Changelog

All notable changes to the OpenSTR protocol are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

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
