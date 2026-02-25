# OpenSTR Protocol

[![Status](https://img.shields.io/badge/Status-Draft-yellow.svg)]()
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-0.1.0--draft-orange.svg)]()

**An open standard for machine-readable short-term rental property discovery and booking, designed for AI agent workflows.**

---

## What is OpenSTR?

OpenSTR is an open protocol that defines a common language for how AI agents discover, query, and book short-term rental properties — without requiring a centralised platform intermediary.

As AI agents become the primary interface through which people search, plan, and transact, the short-term rental industry faces a structural shift. Today, property discovery and booking are controlled by a small number of closed platforms. OpenSTR proposes an open alternative: any host can publish a machine-readable listing, and any AI agent can discover and book it, using a shared, standardised vocabulary.

OpenSTR is **not** a platform. It is a specification — a set of agreed data formats and API conventions that anyone can implement.

---

## Design Principles

**Open by default.** The specification is public, free to implement, and governed transparently. No licence fees, no approval process.

**Built on existing standards.** OpenSTR is designed to be payment-compatible with the [Agentic Commerce Protocol (ACP)](https://agenticcommerce.dev) and [Google's Agent Payments Protocol (AP2)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol), and uses [W3C Verifiable Credentials](https://www.w3.org/TR/vc-data-model-2.0/) for the bilateral trust layer. We build on what already exists rather than reinventing it.

**Separation of concerns.** OpenSTR handles property discovery, availability querying, and booking confirmation. Payment execution delegates to ACP/AP2-compliant payment handlers. Identity verification is handled by a pluggable credential layer.

**Agent-first, not human-first.** Schemas and API responses are optimised for machine parsing and agent reasoning, not human reading. Human-facing interfaces are an implementation concern, not a protocol concern.

**Bilateral trust.** Trust in a short-term rental transaction flows in both directions. OpenSTR defines verifiable credential interfaces for both guests and hosts. A guest's agent can verify a host's identity and right-to-let before booking; a host's system can verify a guest's identity and track record before accepting. Neither party is trusted by default.

**Progressive trust.** The protocol defines what identity and reputation claims are required, but is agnostic about how those claims are produced. Both hosts and guests decide which credential issuers they accept.

**Standards body, not platform.** openstr.org governs the specification and maintains the trusted issuer registry. It does not operate as a booking platform, payment processor, or credential issuer. The commercial opportunity created by the protocol is deliberately left open for third parties to build on.

---

## Scope

### v0.1 — Core Discovery and Booking (current)

All four core RFCs are drafted and available in the `rfcs/` directory.

- **Property Listing** (`rfc.property_listing.md`) — a structured, machine-readable description of a short-term rental property, including a three-tier property classification, amenities, environment tags, safety disclosures, static and dynamic pricing rules, minimum stay declaration, and the damage guarantee model based on identity verification tiers
- **Availability Query** (`rfc.availability_query.md`) — a standardised request/response for checking property availability, resolving dynamic pricing and minimum stay rules, and returning a complete pricing breakdown including all fees and applicable discounts
- **Booking Confirmation** (`rfc.booking_confirmation.md`) — a standardised flow for submitting a booking request with bilateral credential validation, two confirmation paths (instant book and request-to-book), post-confirmation location disclosure, access code delivery, and a cancellation endpoint
- **Identity and Trust** (`rfc.identity_trust.md`) — the bilateral credential system defining the `GuestCredential` and `HostCredential` schemas, three-tier guest IDV vocabulary, W3C Verifiable Credentials proof mechanism, credential lifecycle including expiry and Bitstring Status List revocation, and the trusted issuer registry and accreditation model

### v0.2 — Review and Reputation Issuance *(planned)*

Defines how verified post-stay reviews are created, cryptographically signed, and attached to both guest and host credentials as portable, platform-independent reputation records. Review data will be anchored to confirmed OpenSTR booking references, preventing fake reviews. Reputation fields are already defined in the v0.1 `HostCredential` schema for forward compatibility.

### v0.3 — Guest-Host Communication *(planned)*

A standardised, platform-independent messaging layer enabling pre-booking enquiries, post-booking coordination, and check-in communications between a guest's agent and a host's endpoint — without requiring a centralised messaging platform.

### v0.4 — Operational Tooling Integration *(planned)*

Defines how host-side property management systems integrate with the OpenSTR protocol. Covers automated task management, smart lock token generation and delivery, channel management, and operational event notifications. A short-term rental automation platform serves as the reference implementation for v0.4.

---

## How It Works

A host implementing OpenSTR exposes three endpoints on their own domain or infrastructure:

```
GET  /.well-known/openstr.json         → Property listing metadata
POST /openstr/availability             → Availability and pricing query
POST /openstr/booking                  → Booking request and confirmation
```

An AI agent acting on behalf of a guest:

1. Discovers a property via web crawl, directory, or direct URL
2. Fetches the property listing from `/.well-known/openstr.json`
3. Verifies the `HostCredential` against the OpenSTR trusted issuer registry
4. Queries the availability endpoint for the guest's requested dates
5. Presents the guest with confirmed pricing including all fees and applicable discounts
6. Presents safety disclosures, cancellation policy, and damage guarantee terms
7. Submits a booking request with a verifiable `GuestCredential` and ACP/AP2 payment token
8. Receives a confirmed booking response including exact location detail and access instructions

No centralised platform is involved at any step. The transaction is directly between the guest's agent and the host's endpoint.

---

## The Machine-to-Machine Vision

OpenSTR is designed for a world where AI agents are the primary interface for travel planning and booking. In this model, a guest instructs their agent — "find me a two-bedroom apartment in Chicago, dog-friendly, four nights in July, around $200 a night" — and the agent handles discovery, availability checking, pricing, credential validation, and booking confirmation without the guest visiting a single webpage.

In this future state, a host's "listing" is an endpoint, not a website. A URL that returns structured JSON. The agent handles all guest-side presentation using the data in the listing — property details, images, amenities, policies — without the host needing a separate human-readable page.

**The transition period**

We are not there yet. Most guests will want to verify what they are booking before fully trusting an agent's recommendation, and most hosts will want a visual interface to manage their listing. The `listing_url` field in the OpenSTR listing provides a human-readable fallback for guests who want it, and the `images` array gives agents the raw material to generate their own presentation without requiring a separate webpage.

As agent trust develops, the human-readable layer will be used less. OpenSTR is designed to support both states — a full website today, a pure JSON endpoint tomorrow — without requiring either.

**What hosts actually need**

If the agent handles guest-side presentation, the host-facing side needs to solve a different problem: publishing and managing a valid OpenSTR listing, not showcasing a property. That breaks into three practical components:

A **listing management interface** — where the host enters and maintains their property data and generates a valid `openstr.json`. A **hosting layer** — the three OpenSTR endpoints need to run somewhere; for non-technical hosts this is a hosted service rather than self-managed infrastructure. A **booking dashboard** — an internal management view for incoming requests, confirmed reservations, and revenue reporting. Entirely host-facing; guests never see it.

These are implementation concerns rather than protocol concerns. OpenSTR defines what the endpoints must return; the tooling that helps hosts publish and manage them is a separate layer that the ecosystem will build.

---

## Trust and Certification

### Guest credentials

Guest identity credentials are issued by recognised third-party IDV providers — Stripe Identity, Onfido, Yoti, and others — that meet OpenSTR's requirements. openstr.org maintains a curated registry of recognised guest credential issuers at `openstr.org/trusted-issuers/guest`.

### Host credentials

Host credentials must attest to both identity and property-level right-to-let — a combination no existing third-party registry covers today. OpenSTR defines the `HostCredential` schema and an accreditation framework for third-party registries that wish to issue compliant credentials. openstr.org maintains a registry of accredited HostCredential issuers at `openstr.org/trusted-issuers/host`.

### Bootstrap issuer

At the time of writing, no accredited HostCredential issuers exist. A separate certification company — legally and operationally independent of openstr.org — is planned to serve as the first accredited issuer, solving the bootstrap problem while preserving the independence of the standards body. This model is deliberately structured to avoid conflicts of interest: openstr.org sets the accreditation bar; an independent company does the verification work commercially. As the ecosystem matures, additional issuers are expected to enter the space under the same accreditation scheme.

This two-entity model mirrors the PCI DSS structure, where the PCI Security Standards Council defines the standard and independent Qualified Security Assessors perform the compliance work.

---

## Repository Structure

```
rfcs/
├── rfc.property_listing.md        ← Property schema and discovery (v0.1.3)
├── rfc.availability_query.md      ← Availability and pricing query (v0.1.1)
├── rfc.booking_confirmation.md    ← Booking request and confirmation flow (v0.1.0)
└── rfc.identity_trust.md          ← Guest and host credential interfaces (v0.1.0)

spec/
└── 0.1.0-draft/
    ├── openapi.yaml               ← Machine-readable API definition
    └── json-schema/               ← JSON Schema definitions

examples/
├── property_listing.json
├── availability_query.json
├── availability_response.json
├── booking_request.json
└── booking_response.json

docs/
├── governance.md
├── principles.md
├── compatibility.md               ← ACP / AP2 / MCP compatibility notes
└── trusted-issuers.md             ← Trusted issuer registry and accreditation requirements
```

---

## Compatibility

OpenSTR is designed to complement, not compete with, existing agentic commerce infrastructure.

| Layer | Handled by |
|---|---|
| Property discovery and description | OpenSTR |
| Availability and booking confirmation | OpenSTR |
| Payment execution | ACP / AP2 |
| Guest identity verification | OpenSTR `GuestCredential` + recognised third-party IDV issuers |
| Host identity and right-to-let verification | OpenSTR `HostCredential` + accredited HostCredential issuers |
| Guest and host reputation | OpenSTR `GuestCredential` / `HostCredential` reputation fields (v0.2) |
| Agent-to-tool integration | MCP (OpenSTR endpoints can be exposed as MCP tools) |

---

## Current Status

OpenSTR is in **early draft**. All four v0.1 core RFCs are written and available in the `rfcs/` directory. The specification is actively seeking feedback from the developer, property technology, and AI agent communities.

This is not yet a stable specification. Breaking changes to the schema and API design should be expected before v1.0.

---

## Contributing

We welcome contributions, critiques, and implementation reports. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a pull request or opening an issue.

To propose a significant change to the protocol, open a **Spec Enhancement Proposal (SEP)** by creating an issue with the `SEP` label and following the template in [docs/governance.md](docs/governance.md).

---

## Relationship to Other Standards

| Standard | Relationship |
|---|---|
| [Schema.org LodgingBusiness](https://schema.org/LodgingBusiness) | OpenSTR defines its own native schema; optional `schema_org` compatibility field provided |
| [ACP (OpenAI / Stripe)](https://agenticcommerce.dev) | OpenSTR delegates payment execution to ACP-compliant handlers |
| [AP2 (Google)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol) | AP2 is a supported payment execution target |
| [W3C Verifiable Credentials 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Required credential format for both GuestCredential and HostCredential |
| [W3C Bitstring Status List](https://www.w3.org/TR/vc-bitstring-status-list/) | Recommended revocation mechanism; mandatory for HostCredential issuers |
| [MCP (Anthropic)](https://modelcontextprotocol.io) | OpenSTR endpoints can be exposed as MCP tools for agent integration |
| [OTA / OpenTravel Alliance](https://opentravel.org) | Prior art; OpenSTR classification vocabulary independently derived |
| [RFC 5545 — iCalendar](https://datatracker.ietf.org/doc/html/rfc5545) | Prior art for availability; OpenSTR extends to cover pricing and booking |

---

## Licence

Licensed under the [Apache 2.0 Licence](LICENSE).

---

## Origin

OpenSTR was initiated in February 2026 by Daniel Bloom as an independent open standard, hosted at [openstr.org](https://openstr.org). The protocol emerged from a practical observation: as AI agents become the primary interface for travel planning and booking, the absence of an open, machine-readable standard for short-term rental properties creates both a gap and an opportunity.

Contributions and co-maintainers are welcome.
