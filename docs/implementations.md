# Registered Implementations

This document lists known implementations of the OpenSTR protocol.

## How to register

Open a pull request adding your implementation below. See CONTRIBUTING.md for details.

## Format

| Implementer | Type | RFC coverage | Status | Link |
|---|---|---|---|---|
| OpenSTR Demo (openstr.org) | Host-side reference implementation | RFC-001, RFC-002, RFC-003, RFC-004 | Live | https://demo.openstr.org |
| OpenSTR Discovery Index (openstr.org) | Discovery index | RFC-005 | Live | https://index.openstr.org |

## Implementation notes

### OpenSTR Demo

A reference implementation of the host-side OpenSTR protocol endpoints, deployed as a Cloudflare Worker. Implements a fictional property (2BR Lincoln Park garden flat, Chicago) with full availability querying, booking confirmation, cancellation, and status retrieval. Includes an interactive API explorer at `demo.openstr.org/test`.

- **Listing endpoint:** `https://demo.openstr.org/.well-known/openstr/host-demo-chicago-001.json`
- **Availability endpoint:** `https://demo.openstr.org/openstr/availability`
- **Booking endpoint:** `https://demo.openstr.org/openstr/booking`
- **Host credential:** `https://demo.openstr.org/.well-known/openstr/host-demo-chicago-001/credential.json` *(demo stub — not issued by accredited issuer)*
- **RFC-001 coverage:** Full property listing schema including pricing, amenities, policies, and damage guarantee
- **RFC-002 coverage:** Full availability query including dynamic pricing resolution, quote issuance, and gap-fill rules
- **RFC-003 coverage:** Full booking confirmation, status retrieval, and cancellation with refund calculation
- **RFC-004 coverage:** Stub HostCredential — structure compliant, not cryptographically verified

### OpenSTR Discovery Index

The first implementation of the OpenSTR Discovery Index as defined in RFC-005. Deployed as a Cloudflare Worker with a D1 database. Accepts host registrations, validates listing schema on crawl, stores denormalised listing data for search, and exposes a structured agent query API.

- **Status endpoint:** `https://index.openstr.org/v1/status`
- **Registration endpoint:** `https://index.openstr.org/v1/register`
- **Search endpoint:** `https://index.openstr.org/v1/search`
- **Listing detail:** `https://index.openstr.org/v1/listings/:index_id`
- **RFC-005 coverage:** Registration, crawling, agent search, listing detail, host reputation (review submission and aggregation)
- **Registered listings:** 1 (OpenSTR Demo property)
