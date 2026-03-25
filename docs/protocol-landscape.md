# Protocol Landscape

This document explains how OpenSTR relates to the emerging ecosystem of agentic commerce protocols. It is intended for developers evaluating OpenSTR, agent builders deciding how to integrate, and channel managers assessing protocol compatibility.

---

## The emerging stack for agent-native commerce

Several open protocols have emerged in 2025–2026 that collectively define how AI agents discover, negotiate with, and transact with commercial services. They operate at different layers and are largely complementary rather than competing. OpenSTR is a domain-specific protocol built within and on top of this stack.

The four protocols most relevant to OpenSTR are:

| Protocol | Governed by | Primary purpose |
|---|---|---|
| [MCP — Model Context Protocol](https://modelcontextprotocol.io) | Anthropic | Standardises how AI models connect to external tools and data sources |
| [A2A — Agent2Agent Protocol](https://a2a-protocol.org) | Google | Standardises communication between AI agents |
| [UCP — Universal Commerce Protocol](https://ucp.dev) | Google / UCP Authors | Standardises agent-native checkout and payment for retail commerce |
| [AP2 — Agent Payments Protocol](https://ap2-protocol.org) | Open / Google-aligned | Standardises autonomous payment mandates for AI agents |

OpenSTR sits at the intersection of these: it is a **domain-specific discovery and booking protocol for short-term rental**, designed to be transport-compatible with MCP and A2A, and payment-compatible with AP2 and UCP.

---

## MCP — Model Context Protocol

### What it is

MCP is Anthropic's open standard for connecting AI models to external tools, APIs, and data sources. It defines a common interface so that an LLM can call a tool — a web search, a calendar lookup, a booking API — using a consistent, model-agnostic protocol. MCP servers expose a set of named tools; MCP clients (models and agents) discover and invoke them.

### How OpenSTR relates

OpenSTR endpoints are natural MCP tools. A host's availability endpoint (`POST /openstr/availability`) and booking endpoint (`POST /openstr/booking`) can be wrapped in an MCP server and exposed to any MCP-compatible agent without modification to the underlying OpenSTR implementation.

An OpenSTR MCP server would expose tools along the lines of:

```
check_availability(listing_id, check_in, check_out, guests) → PricingResponse
create_booking(listing_id, check_in, check_out, guests, guest_credential) → BookingConfirmation
get_listing(listing_id) → PropertyListing
```

The index layer (`index.openstr.org`) can similarly be exposed as an MCP tool for discovery: `search_listings(location, dates, guests, filters) → [ListingRef]`.

**Summary:** MCP is a transport and tooling layer. OpenSTR is payload and domain semantics. They are fully complementary. OpenSTR endpoints should, and eventually will, be exposed as MCP tools as part of the reference implementation.

---

## A2A — Agent2Agent Protocol

### What it is

A2A is Google's open standard for structured communication between AI agents. Where MCP defines how a model calls a tool, A2A defines how one agent communicates with another agent acting on behalf of a different party. It introduces the concept of an **Agent Card** (published at `/.well-known/agent.json`) that describes an agent's capabilities, supported protocols, and endpoints.

### How OpenSTR relates

In an OpenSTR transaction, there are two agents: the **guest agent** (acting on behalf of the traveller) and the **host agent or endpoint** (acting on behalf of the host or property management system). This is precisely the multi-agent scenario A2A is designed for.

A host implementing OpenSTR could publish an A2A Agent Card that declares support for OpenSTR booking flows. A guest agent that speaks A2A could discover the host endpoint via the Agent Card and negotiate the transaction using structured A2A messaging, with OpenSTR schemas as the shared data vocabulary.

UCP already defines an A2A transport binding for its checkout capability. OpenSTR's booking confirmation flow (`RFC-003`) is the natural parallel for the STR vertical.

**Summary:** A2A is the inter-agent communication layer; OpenSTR provides the STR-specific data vocabulary and transaction semantics that travel over it. Future versions of OpenSTR should define an A2A binding for the booking flow.

---

## UCP — Universal Commerce Protocol

### What it is

UCP is Google's open standard for agent-native retail commerce. It defines how a business publishes a machine-readable profile (at `/.well-known/ucp`), how an agent discovers and negotiates capabilities with that business, and how checkout and payment flow through a decoupled three-party trust model (business / platform / payment credential provider). It is Apache 2.0 licensed and transport-agnostic: it supports REST, MCP, A2A, and an embedded protocol binding.

UCP's current capability set covers retail checkout, order lifecycle management, and identity linking. Its roadmap includes multi-item carts, loyalty, cross-sell, and global market rollout.

### How OpenSTR relates

The structural parallels between OpenSTR and UCP are deliberate. Both use `/.well-known/` for self-declaration. Both are designed for AI agent consumption. Both are Apache 2.0. Both keep the merchant (or host) as the merchant of record, avoiding platform intermediation.

The critical difference is **domain scope**. UCP is a horizontal retail checkout protocol — it handles line items, payment tokenisation, PCI-DSS scope, and cart semantics. It has no concept of temporal availability, date-range pricing, occupancy rules, iCal synchronisation, or the bilateral trust model required for an overnight stay where a stranger enters your property.

OpenSTR is the STR vertical built within the same architectural paradigm as UCP:

| Concern | UCP | OpenSTR |
|---|---|---|
| Self-declaration endpoint | `/.well-known/ucp` | `/.well-known/:listing_id.json` |
| Discovery | Google AI Mode / Gemini | `index.openstr.org` + commercial indices |
| Capability negotiation | Intersection algorithm on profile capabilities | RFC-001 schema version |
| Availability | Not applicable | RFC-002 availability query |
| Checkout / booking | `dev.ucp.shopping.checkout` capability | RFC-003 booking confirmation |
| Payment | UCP payment handlers + AP2 mandates | AP2-compatible (RFC-003, planned) |
| Trust / identity | Platform-managed | RFC-004 bilateral credentials |
| Transport | REST, MCP, A2A, embedded | REST (MCP and A2A bindings planned) |

**Vendor namespace.** UCP's governance model uses reverse-domain capability naming. OpenSTR capabilities can be declared under the `org.openstr.*` vendor namespace — for example, `org.openstr.rental.availability` and `org.openstr.rental.booking` — positioning them as UCP-compatible extensions for the STR vertical. This requires no UCP approval; the governance model explicitly delegates authority to domain holders.

**What this means practically:** a host publishing an OpenSTR listing can, without conflict, also publish a `/.well-known/ucp` profile that declares `org.openstr.rental.*` capabilities. A UCP-aware agent encountering a property endpoint would then be able to negotiate STR-specific capabilities using the same profile and intersection algorithm it uses for retail checkout.

**Summary:** UCP is the horizontal retail layer. OpenSTR is the STR vertical. The two are architecturally aligned and should be explicitly compatible. OpenSTR's RFC development roadmap should include a formal UCP compatibility binding.

---

## AP2 — Agent Payments Protocol

### What it is

AP2 is an open protocol for autonomous agent payments. It defines how an AI agent can acquire a **cryptographic payment mandate** — a verifiable, non-repudiable authorisation from a user to a specific merchant for a specific transaction amount — and present it to a merchant for settlement. The mandate is signed by the user's key on a non-agentic surface (e.g., a wallet app), preventing the agent from authorising payments beyond what the user explicitly approved.

UCP's AP2 Mandates Extension (`dev.ucp.shopping.ap2_mandate`) builds on this for the retail context. AP2 is also the payment layer referenced in ACP (Agentic Commerce Protocol, OpenAI/Stripe-aligned).

### How OpenSTR relates

OpenSTR's booking confirmation flow (RFC-003) currently captures booking intent and confirms terms but defers payment execution to a payment handler. AP2 is the intended payment layer for that handler in fully autonomous booking scenarios.

In a complete AP2-backed OpenSTR booking:

1. The guest agent presents a summary of the booking terms (property, dates, price, cancellation policy) to the guest on a non-agentic surface.
2. The guest cryptographically signs a payment mandate for the exact amount.
3. The agent includes the mandate in the booking request (`POST /openstr/booking`).
4. The host endpoint verifies the mandate and, on confirmation, charges the payment credential provider directly.

This provides the host with **non-repudiable proof** that the guest authorised the specific transaction — essential for dispute resolution in a direct, platform-free booking. It also respects PCI-DSS scope in the same way UCP's payment architecture does: raw payment credentials never pass through OpenSTR infrastructure.

RFC-003's payment model should be aligned with AP2 mandate semantics rather than defining a parallel credential model. The `payment_token` field in the booking request should accept AP2 mandate tokens as the primary format, with OpenSTR specifying how mandate scope maps to booking terms (amount, dates, cancellation thresholds).

**Summary:** AP2 is the payment mandate layer. OpenSTR is the booking layer. RFC-003's payment architecture should be explicitly AP2-compatible, borrowing from UCP's AP2 Mandates Extension as a reference model.

---

## How the layers fit together

```
┌─────────────────────────────────────────────────────────────┐
│                     Guest Agent                             │
│           (MCP client / A2A participant)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │  A2A  /  MCP
┌──────────────────────────▼──────────────────────────────────┐
│                  OpenSTR Protocol Layer                     │
│  RFC-001 listing  │  RFC-002 availability  │  RFC-003 booking│
│                   └──────────────────────────────────────── │
│                              RFC-004 identity / trust        │
└────────────────────────┬────────────────────────────────────┘
                         │  AP2 payment mandate
┌────────────────────────▼────────────────────────────────────┐
│              Payment Credential Provider                    │
│               (AP2 / UCP payment handler)                   │
└─────────────────────────────────────────────────────────────┘
```

OpenSTR is the middle layer. It does not duplicate MCP, A2A, UCP, or AP2 — it uses them:

- **MCP** as the tool-binding layer for agent integrations
- **A2A** as the inter-agent communication layer for guest-host agent dialogue
- **UCP** as the architectural reference model for capability declaration and checkout; OpenSTR capabilities are declared under `org.openstr.*` in the UCP governance namespace
- **AP2** as the payment mandate layer for RFC-003 booking confirmation

---

## OpenSTR's unique contribution

None of the four protocols above has a concept of:

- A short-term rental property with temporal availability and date-range pricing
- Occupancy and minimum stay rules with exception overrides
- Bilateral trust between a host and a guest, where both parties are strangers to each other and both require identity and reputation verification before transacting
- iCal synchronisation with external channel managers
- The damage guarantee model tied to identity verification tier
- Post-booking location disclosure contingent on payment confirmation
- The channel manager integration layer needed to reach scale in the STR market

These are OpenSTR's domain contributions. Everything else — transport, payment, agent communication, capability negotiation — is deliberately delegated to the appropriate layer of the emerging agentic commerce stack.

---

## Roadmap: protocol alignment work

The following items are on the OpenSTR roadmap to formalise protocol compatibility:

1. **MCP tool definitions** — publish an MCP server schema exposing OpenSTR endpoints as named tools, as part of the reference implementation.

2. **UCP vendor namespace declaration** — formally declare `org.openstr.rental.*` capabilities under the UCP governance model, enabling UCP-aware agents to negotiate STR capabilities using the standard UCP profile and intersection algorithm.

3. **A2A binding for RFC-003** — define an A2A transport binding for the booking confirmation flow, enabling guest and host agents to negotiate bookings using structured A2A messaging with OpenSTR schemas as the shared vocabulary.

4. **AP2-compatible payment model for RFC-003** — align the `payment_token` field in the booking request with AP2 mandate token semantics, borrowing from UCP's AP2 Mandates Extension as a reference model, so that RFC-003 supports non-repudiable, cryptographically verified payment authorisation out of the box.

---

*Last updated: March 2026. This document will be updated as the UCP, AP2, A2A, and MCP specifications evolve.*
