# RFC: Documentation Layer

**RFC ID:** openstr-rfc-006 *(proposed)*
**Title:** Booking Documentation, Execution Confirmation, and Post-Booking Lifecycle Records
**Status:** Pre-Draft
**Version:** 0.0.1
**Created:** March 2026
**Updated:** March 2026
**Authors:** Daniel Bloom (openstr.org)
**Relates to:** openstr-rfc-003 (Booking Confirmation) open question 11.4; openstr-rfc-003 open question 11.7; v0.4 operational tooling RFC (planned)

---

## Purpose of this document

This pre-draft captures the design thinking developed in the OpenSTR promotion project around the documentation layer of the protocol — what it needs to do, what it must not mandate, and how it fits within the broader trust stack. It is intended as input for the development project to review, challenge, and develop into a formal RFC.

**Protocol roadmap note.** The four v0.1 core RFCs (property listing, availability query, booking confirmation, identity and trust) do not currently define a documentation layer. The booking confirmation RFC (openstr-rfc-003 v0.1.1) explicitly defers dispute resolution to v0.3 or later (open question 11.4). This document is proposed as input for that future RFC, and where relevant, for the v0.4 operational tooling RFC which covers smart lock integration and post-booking lifecycle events. Terminology and field names in this document are aligned with the v0.1 RFCs — specifically the `booking_reference` and `confirmed_at` fields from rfc.booking_confirmation.md, and the `GuestCredential` and `HostCredential` credential ID structures from rfc.identity_trust.md.

**Roadmap sequencing to confirm.** This document assumes the documentation layer lands across v0.3 (executed agreement and dispute resolution) and v0.4 (post-booking lifecycle records and operational tooling). This sequencing should be confirmed with the development project before this pre-draft is promoted to Draft status.

---

## Background: why documentation matters in an agentic flow

When a human books on Airbnb, the documentation layer is invisible. A binding agreement is created, terms are accepted, and a record exists — all within the platform's booking confirmation flow. The guest never consciously signs anything.

When a professional STR operator manages bookings outside a major OTA — through a PMS, a direct booking site, or a channel manager — the documentation layer is currently broken. Agreement, identity check, deposit request, and rental terms typically arrive as separate, disconnected communications from different systems, each requiring a manual step. The guest experiences all of them. This is the fragmented reality described in Article 2 of the OpenSTR series.

In an agentic booking flow, the documentation layer must work like Airbnb — invisible, automatic, and complete — but without platform dependency. The protocol needs to define what happens at the moment of booking confirmation so that any compliant implementation produces a trustworthy, verifiable record regardless of which system executed the booking.

---

## Core principle: mandate the standard, not the service

The OpenSTR protocol should mandate **what the documentation record must contain and how it can be verified** — not who holds it or which service produces it.

This is a deliberate design decision based on the following reasoning:

**Developer adoption.** Any requirement to push data to a mandatory external third-party service adds cost, complexity, and dependency. Developers building compliant PMS platforms or booking tools already hold the relevant data and already carry GDPR obligations for it. A large PMS who can meet the standard internally should not be required to use an external service. Mandating a specific service would be a significant barrier to protocol adoption.

**GDPR alignment.** The party holding the documentation record must have appropriate data processing obligations for the personal data it contains. This is already true of any compliant booking platform. Requiring data to move to an additional party creates additional GDPR surface without clear benefit.

**Neutrality.** The protocol is a nonprofit open standard with no commercial interest in the infrastructure it defines. Mandating a specific service would contradict that principle.

**Guest experience.** Guests do not need to know or care where the documentation record is held. The invisible flow is the goal. What matters is that the record exists, is tamper-proof, and is verifiable if a dispute arises.

---

## What the documentation record must contain

At the point of booking confirmation, the following must be recorded:

- **Cryptographic hash of the executed agreement** — generated from the full document content at the moment of execution, making the record tamper-proof. If the document is altered after the fact, the hash will not match.
- **Document creation timestamp** — the date and time at which the document was generated and hashed.
- **Execution status** — an enum field recording the state of execution: `pending` (document created, execution not yet confirmed), `confirmed` (execution event received and recorded), or `disputed` (execution has been formally contested). A document can be created and hashed before execution; the status transitions from `pending` to `confirmed` separately when the execution event occurs. The `disputed` state allows the record to reflect contested execution without requiring the record to be altered or deleted.
- **Execution event reference** — the independently verifiable identifier of the execution event — see Execution Confirmation section below.
- **Execution timestamp** — separate from the document creation timestamp, recording when execution was confirmed.
- **Booking reference** — the unique identifier connecting the documentation record to the booking event in the OpenSTR system.
- **Credential IDs of both parties** — references to the verified identity credentials of the host and guest, not the underlying personal data. These link the agreement to verified identities without the documentation layer needing to hold personal identity data directly.
- **Property reference** — the OpenSTR property identifier, linking the agreement to the specific listing.
- **Rental terms reference** — a pointer to the version of the rental terms that were in effect at the point of execution. This supports the single-source-of-truth principle: terms are defined once in the OpenSTR property schema and inherited by the documentation layer at booking confirmation, rather than maintained separately in two places.
- **Trust tier** — an integer indicating which execution confirmation method was used: `1` for Tier 1 (session acceptance) or `2` for Tier 2 (W3C Verifiable Presentation). Returned alongside `execution_status` by the verification endpoint — see Execution Confirmation section below.

---

## What the verification endpoint must do

A verification endpoint must exist that allows any party — host, guest, platform, regulator, or AI agent — to confirm that:

1. A documentation record exists for a given booking reference.
2. The hash stored in the record matches the document presented.
3. The record was created at the stated timestamp.
4. The `execution_status` of the record, and the timestamp at which execution was confirmed.
5. The credential IDs recorded match the parties to the transaction.

The verification endpoint must expose only non-personal metadata. It must not expose the agreement content, party names, or any personal data. This is consistent with GDPR requirements and with the design principle that personal data remains with the party responsible for processing it.

---

## Execution confirmation

A tamper-proof document that no party has confirmed they accept is not an executed agreement. The cryptographic hash proves content integrity. It does not prove acceptance. In a dispute, a guest could claim they never agreed to the terms, and without an execution record there is no counter to that.

The protocol therefore requires an execution confirmation event — a verifiable act that only an authenticated party could have triggered — to be recorded separately from document creation and included in the documentation record.

**Why payment confirmation is not used**

Payment execution is explicitly out of scope for OpenSTR at the protocol level (rfc.booking_confirmation.md Section 3.4). The protocol is payment-agnostic — implementations may use UCP, AP2, channel manager rails, or direct payment. Building execution confirmation on payment would therefore create a dependency on an implementation detail that the protocol deliberately does not define, and would introduce jurisdictional fragility in dispute scenarios where payment is reversed or disputed. These are separate concerns: financial settlement and contractual acceptance.

**Tiered execution confirmation model**

Rather than mandating a single execution confirmation method — which would either set too high a bar for adoption or provide insufficient trust quality — the protocol defines two tiers. All compliant implementations must meet Tier 1. Tier 2 is the preferred method where available and is signalled in the documentation record through the trust tier field.

**Tier 1 — Minimum viable execution confirmation**

An explicit digital acceptance event: a logged, timestamped confirmation from an authenticated user session, tied to the booking reference (`booking_reference` from rfc.booking_confirmation.md Section 5.1). This is equivalent to a guest clicking "confirm booking" on a platform — the click is recorded, timestamped, and tied to an authenticated session. It requires no special infrastructure beyond what any standard booking platform already has. Authentication may be via verified email session, OAuth login, or equivalent.

The execution event reference in this tier is the session token or acceptance event ID generated by the booking platform at the moment of confirmation. This is independently verifiable by the platform that issued it.

**Implementation note — jurisdictional variation.** The legal sufficiency of a session acceptance event as contractual execution varies by jurisdiction. In some jurisdictions, the act of payment or payment pre-authorisation is treated as the operative acceptance event; in others, a prior explicit digital acceptance is required. The protocol does not resolve these jurisdictional differences — it defines a minimum technical record that compliant implementations must produce. Implementations are responsible for ensuring that the rental terms are presented to the guest before the execution event is recorded, and that the acceptance mechanism meets applicable legal requirements in the jurisdictions in which they operate. Tier 1 is a technical floor, not a legal guarantee.

Tier 1 is immediately implementable by any developer with a standard booking flow. It does not depend on the GuestCredential infrastructure existing first.

**Tier 2 — Enhanced execution confirmation**

A W3C Verifiable Presentation from a GuestCredential issued by an accredited OpenSTR credential provider, as defined in rfc.identity_trust.md. The guest's identity credential is presented to the host endpoint at booking confirmation, and that presentation event is itself cryptographically signed and timestamped, tying execution directly to a verified, portable identity rather than a platform session.

The credential ID in the execution event reference uses the `issuer_guest_id` opaque reference from the GuestCredential schema (rfc.identity_trust.md Section 9.1) — ensuring execution is tied to verified identity without exposing the guest's underlying personal data.

Tier 2 becomes available as the GuestCredential issuer layer matures. Implementations using Tier 2 return a higher trust tier value in the verification endpoint response, giving platforms and AI agents a machine-readable signal of the quality of execution evidence.

The architecture for both tiers is the same — only the quality of the execution event reference differs. Implementations can upgrade from Tier 1 to Tier 2 without a breaking change to the documentation record structure.

**The record format**

The protocol record is a structured JSON object — machine-readable, hashable, and queryable through the verification endpoint. The PDF agreement is a human-readable presentation layer generated from the same data. The JSON object is the record of truth; the PDF is derived from it. This is a deliberate inversion of how documentation tools typically work today, where the PDF is primary and the verification layer is secondary.

---

## Who can hold the record

Any of the following may hold the documentation record, provided they meet the requirements above:

**Internally, by the booking platform or PMS.** A developer building a compliant booking tool may implement the hashing, storage, and verification endpoint within their own infrastructure. This is the lowest-friction path for large, well-resourced operators and is explicitly permitted by the protocol.

**By a neutral third-party documentation service.** A service whose sole function is to receive executed agreement data, produce and store the cryptographic hash, and provide a verification endpoint. This is the appropriate path for smaller operators or developers who do not want to build the infrastructure themselves. To qualify as a compliant neutral third-party documentation service, an implementation must: accept an API-triggered case creation event (rather than requiring manual host initiation); implement SHA-256 hashing of document content at creation; record a Tier 1 or Tier 2 execution event reference against the case; expose a public verification endpoint returning non-personal metadata only; and store personal data in a manner consistent with applicable GDPR obligations.

**By the identity or insurance verification provider.** In the longer-term vision where a host's and guest's identity credentials are issued by accredited third-party providers, those providers may also absorb the documentation confirmation function as part of the booking handshake. This is the idealised end state — an invisible cryptographic handshake between two verified credentials at the moment of booking confirmation — but it depends on identity and insurance infrastructure that does not yet exist in connected form.

---

## The idealised long-term flow

For reference, the target agentic booking flow the documentation layer is designed to support. Field names are aligned with rfc.booking_confirmation.md and rfc.identity_trust.md.

1. An AI agent finds a property through an OpenSTR-compliant index.
2. The agent verifies the `HostCredential` against the OpenSTR trusted issuer registry (rfc.identity_trust.md Section 8.1).
3. The agent confirms availability and pricing through the availability endpoint.
4. The agent presents full pricing, policies, and safety disclosures to the user and receives explicit confirmation (rfc.booking_confirmation.md Section 9.2–9.4).
5. The user confirms the booking — this is the execution event. The booking request is submitted with the `GuestCredential` to the booking endpoint.
6. The host system validates the `GuestCredential` (rfc.identity_trust.md Section 8.2) and returns a confirmed booking response including `booking_reference` and `confirmed_at`.
7. At the moment of booking confirmation, a fully populated agreement — rental terms inherited from the property schema, credential IDs of both parties, `booking_reference`, property reference, rental dates — is submitted to the documentation layer.
8. The documentation layer cryptographically hashes the agreement content and records the hash with a creation timestamp.
9. The execution event reference — either a Tier 1 session acceptance ID or a Tier 2 W3C Verifiable Presentation reference using the `issuer_guest_id` from the GuestCredential — is recorded alongside the hash, the `execution_status` is set to `confirmed`, and the execution timestamp is recorded.
10. The documentation layer returns a confirmation reference that includes the trust tier.
11. The confirmation reference is attached to the booking record and both parties' credential records.
12. Neither party has consciously signed anything. The record exists, is tamper-proof, carries a verified execution event, and is queryable by any party with the `booking_reference`.

---

## Post-booking lifecycle records

The documentation layer does not end at booking confirmation. An AI agent that books a stay on behalf of a guest has ongoing responsibilities across the full rental lifecycle — confirming check-in, monitoring the stay, handling checkout, and acting on any damage or dispute that arises. For the agent to perform these functions autonomously, machine-readable records must exist at each stage.

**Protocol roadmap note.** Smart lock token generation and API-based access delivery are scoped for v0.4 (rfc.booking_confirmation.md Section 3.6 and open question 11.3). The lifecycle records defined below are proposed as input for the v0.4 operational tooling RFC and the v0.3 dispute resolution RFC (rfc.booking_confirmation.md open question 11.4). They are not part of the v0.1 core protocol.

The protocol defines three post-booking lifecycle events, each producing a structured record queryable by the agent:

**1. Check-in confirmation**

Confirms that the guest took possession of the property at the agreed time and that access was granted. This record establishes the baseline for any subsequent damage or condition dispute.

Check-in confirmation must not depend on voluntary guest action. Experience with existing platform features — including Airbnb's own arrival confirmation — shows that guests reliably ignore prompts at check-in: they are focused on arrival and have no motivation to complete a checklist. Any protocol design that depends on guest-initiated confirmation at check-in will have near-zero adoption.

Instead, the protocol defines check-in confirmation as a booking lifecycle event that can be confirmed by any of the following authorised signals, in descending order of trust quality:

- **Smart lock access event** — a timestamped access log from a connected smart lock tied to the booking reference. This is the strongest signal: it requires no guest action, is independently verifiable, and is already produced by many PMS-connected properties.
- **Agent-initiated confirmation** — the agent, which holds the booking record and knows the check-in time, sends a single low-friction prompt to the guest at the confirmed check-in time. Not a checklist — a single binary confirmation tap. This is the fallback where smart access is unavailable.
- **PMS trigger** — a check-in event fired by the property management system at the confirmed check-in time, independent of guest action.

The check-in confirmation record contains: booking reference, property reference, confirmation timestamp, confirmation signal type, and optionally a condition baseline reference if a pre-arrival condition record exists.

The condition baseline is not mandatory. Its presence or absence is itself a machine-readable signal — an agent can query whether a baseline exists and factor that into any dispute assessment.

**2. Pre-arrival condition record (optional)**

A machine-readable record of property condition immediately before guest arrival, typically produced by the host's cleaning or management workflow. This is the most reliable damage dispute baseline because it is produced by the host or their staff at the point of handover readiness, not by a guest completing a checklist.

Tools such as Breezeway and Properly already generate cleaning completion and condition records integrated with PMS platforms. The protocol does not require operators to use these tools, but defines a standard output format that any compliant tool can produce — a JSON record containing property reference, booking reference, timestamp, condition status, and optionally a reference to attached photographic evidence.

This record is produced by the host side, not the guest side. It does not require any guest action and does not interrupt the guest experience.

**3. Checkout confirmation**

Confirms that the guest vacated the property and records whether any damage or condition deviation was noted. The protocol applies the same principle as check-in: checkout confirmation should not depend on voluntary guest action.

Reliable checkout signals include:

- **Smart lock final access event** — the last access event before the next booking's check-in window or cleaning trigger, indicating the guest has departed.
- **Agent-initiated confirmation** — the agent knows the checkout time from the booking record and sends a prompt at that moment. Single binary confirmation, not a checklist.
- **Cleaning trigger event** — many PMS platforms fire a cleaning job at checkout time. The initiation of that cleaning job is a reliable departure proxy.

The checkout confirmation record contains: booking reference, property reference, confirmation timestamp, confirmation signal type, and a deviation flag indicating whether any damage or condition issue was recorded. The deviation flag is binary — present or absent — and is the only field the agent needs to query to determine whether a damage record exists. The damage record itself is held separately and may contain personal data; the agent queries the flag, not the record.

**Why the agent needs these records**

Without lifecycle records, an agent acting on a guest's behalf after booking has no machine-readable information about whether the stay proceeded as expected. It cannot confirm the property was accessible at check-in. It cannot assess a damage claim against a verified baseline. It cannot confirm checkout occurred. It is operating blind from the moment the booking is confirmed.

With lifecycle records, the agent can confirm each stage of the rental autonomously, flag anomalies to the guest without waiting for a human to notice, and act on dispute or damage claims with reference to a verified evidential record — not a platform's internal process.

**Implementation is optional, adoption is incentivised**

The protocol does not mandate lifecycle records for all implementations. The mandatory documentation requirement is the executed agreement at booking confirmation. Lifecycle records are defined as optional extensions that implementations can choose to support.

However, the protocol defines them precisely enough that any compliant implementation produces interoperable records. A guest whose agent can query lifecycle records has a materially better protected stay than one whose agent cannot. That asymmetry creates commercial incentive for hosts and operators to support the standard — it becomes a visible quality signal, queryable by agents comparing properties at the discovery stage.

---

## Single source of truth for rental terms

A specific design requirement that emerged from this analysis:

Rental terms — house rules, cancellation policy, pet policy, damage deposit requirements — must be defined once in the OpenSTR property schema and inherited by the documentation layer at booking confirmation. They must not be maintained separately in the documentation system.

If a host maintains terms in two places — once in their property listing and once in a documentation tool — those terms can diverge. A guest sees one set of terms at discovery, a different set in the executed agreement. This is a consistency risk and a potential dispute trigger.

The protocol should define a terms versioning mechanism within the property schema so that the documentation record can reference a specific version of the terms that were in effect at the point of booking, and that version is immutable once referenced by an executed agreement.

---

## Third-party documentation service requirements

This section defines the minimum requirements a neutral third-party documentation service must meet to be considered a compliant implementation under this RFC. These requirements are provided to enable evaluation of candidate services and to give developers building documentation infrastructure a clear target.

**Mandatory requirements**

- **API-triggered case creation.** Cases must be creatable via an API call at booking confirmation time, not solely through manual host initiation. This is necessary for the agentic flow — a human initiating documentation after the fact is architecturally incompatible with invisible, automatic execution.
- **SHA-256 content hashing at creation.** The document content must be hashed at the moment the record is created. The hash must be stored and returned by the verification endpoint. Any modification to the document after hashing must produce a detectable mismatch.
- **Execution event recording.** The service must accept and record a Tier 1 execution event reference (session acceptance ID) or Tier 2 execution event reference (W3C Verifiable Presentation reference) against the case, and must record the `execution_status` and execution timestamp separately from the document creation timestamp.
- **Public verification endpoint.** A publicly accessible endpoint must exist that accepts a booking reference or document ID and returns non-personal verification metadata — at minimum: document hash, `execution_status`, execution timestamp, trust tier, `booking_reference`, property reference, and terms version reference. The endpoint must not return personal data.
- **GDPR-compliant data handling.** Personal data must be held by the party with appropriate processing obligations. A third-party service holding personal data on behalf of a host or platform must have a valid data processing agreement in place.

**Optional but protocol-aligned**

- Modular document type support (agreement, custom terms, handover checklist, damage invoice) connected under a shared case identifier.
- Local or operator-controlled storage of the full document, with only verification metadata held centrally — a GDPR-appropriate architecture for services that process personal data on behalf of operators.
- Lifecycle record support for check-in confirmation, pre-arrival condition records, and checkout confirmation.

Services meeting the mandatory requirements above may be listed in the OpenSTR ecosystem directory. Listing is not endorsement — it indicates that the service has declared compliance with this RFC's requirements. The protocol does not accredit or certify third-party documentation services.

---

## Open questions for the development project

1. **Hash standard.** SHA-256 is the mandated hash function for this RFC. This is consistent with current practice in compliant implementations reviewed during the drafting of this pre-draft, and aligns with the broader W3C Verifiable Credentials ecosystem used by the Tier 2 execution model. No further decision is required on this point; the open question is whether to state this normatively in the RFC body rather than only in the open questions list. **Proposed resolution:** promote to normative requirement in Section "What the documentation record must contain" when this pre-draft is advanced to Draft status.

2. **Terms versioning.** How should the property schema implement terms versioning so that executed agreements can reference an immutable snapshot of the terms in effect at booking? The `policies_confirmed` snapshot in the booking confirmation response (rfc.booking_confirmation.md Section 5.1) covers pricing and cancellation policy but not the full rental terms — a terms version reference field may be needed in the property listing schema. **RFC-001 dependency:** this requirement implies a new field (e.g. `terms_version`) in the property listing schema, which would be a breaking change candidate for RFC-001 v0.3. This pre-draft should be read as formally flagging that dependency. The terms versioning mechanism cannot be fully resolved within this RFC alone — it requires a coordinated change to rfc.property_listing.md.

3. **Verification endpoint specification.** What is the exact JSON response structure the verification endpoint must return? This needs to be defined in the protocol schema so that any implementation — internal or third-party — is interoperable. The response must include at minimum: document hash, `execution_status`, execution timestamp, trust tier, `booking_reference`, property reference, terms version reference, and credential IDs. It must not include personal data.

   **Proposed non-normative schema (for discussion):**
   ```json
   {
     "booking_reference": "string — OpenSTR booking reference",
     "property_ref": "string — OpenSTR listing_id",
     "document_hash": "string — SHA-256 hex digest of the agreement content at creation",
     "created_at": "string — ISO 8601 timestamp of document creation",
     "execution_status": "string — enum: pending | confirmed | disputed",
     "execution_timestamp": "string | null — ISO 8601 timestamp when execution was confirmed",
     "trust_tier": "integer — 1 (session acceptance) or 2 (W3C Verifiable Presentation)",
     "execution_event_ref": "string | null — session token ID (Tier 1) or VP reference (Tier 2)",
     "terms_version_ref": "string — version identifier of the rental terms in effect at execution",
     "host_credential_uri": "string — /.well-known/openstr/{listing_id}/credential.json",
     "guest_credential_id": "string | null — issuer_guest_id opaque reference, present for Tier 2 only"
   }
   ```
   This schema is proposed for discussion only. Field names should be confirmed against the v0.1 RFC vocabulary before promotion to normative status.

4. **Tier 1 session acceptance standard.** What exactly constitutes a valid Tier 1 execution event reference? The protocol needs to define the minimum requirements for an authenticated session acceptance event — what the token must contain, how it must be generated, and how it can be independently verified by a third party querying the verification endpoint.

5. **Tier 2 credential provider accreditation.** Which organisations are accredited to issue GuestCredentials for Tier 2 execution confirmation? This connects to the trusted issuer registry defined in rfc.identity_trust.md and the accreditation framework at `openstr.org/trusted-issuers/guest`.

6. **Credential ID format.** ~~How are host and guest credential IDs formatted in the documentation record?~~ **Resolved by rfc.identity_trust.md.** The guest credential ID uses the `issuer_guest_id` opaque reference from the GuestCredential schema (Section 9.1), ensuring execution is tied to verified identity without exposing personal data. The host credential ID uses the `credential_uri` per-listing path convention (`/.well-known/openstr/{listing_id}/credential.json`) defined in Section 2.6. *Note: verify section numbers against current rfc.identity_trust.md before promotion to Draft.*

7. **JSON record schema.** What is the full JSON schema for the documentation record? The record is the primary source of truth; the PDF is derived from it. The schema needs to be defined precisely enough that any compliant implementation produces an interoperable record. Field names should align with the v0.1 RFC field vocabulary — specifically `booking_reference` and `confirmed_at` from rfc.booking_confirmation.md.

8. **Minimum viable implementation.** What is the minimum a developer must implement to be considered Tier 1 documentation-compliant? This determines the barrier to adoption and should be as low as possible while preserving the tamper-proof, integrity-verified, and execution-confirmed requirements.

9. **Check-in confirmation signal hierarchy.** The protocol defines smart lock access, agent-initiated confirmation, and PMS trigger as valid check-in signals. Smart lock token delivery is scoped for v0.4 (rfc.booking_confirmation.md Section 3.6) — the check-in confirmation signal specification should be coordinated with that RFC. What is the minimum technical specification for each signal type?

10. **Condition baseline record schema.** What is the JSON schema for a pre-arrival condition record? This needs to be defined precisely enough that tools like Breezeway and Properly can produce a compliant output without rebuilding their core product.

11. **Deviation flag specification.** The checkout confirmation record contains a binary deviation flag. What exactly triggers the flag — any condition note, only formally recorded damage, or something else? This needs a precise definition to avoid ambiguity in agent queries.

12. **Lifecycle record optionality.** How should the protocol signal which lifecycle records a given property or operator supports? An agent querying a property at the discovery stage needs a machine-readable capability declaration — does this operator support check-in confirmation, pre-arrival condition records, checkout confirmation? This likely connects to the property listing schema (rfc.property_listing.md).

---

*This document was prepared in the OpenSTR Protocol Promotion Project and should be read alongside the OpenSTR article series, particularly Article 1 (The Invisible Wall) and Article 2 (The New Infrastructure Layer), which provide the broader context for the trust and infrastructure layer of which documentation is one component.*

*openstr.org · github.com/openstr-protocol/protocol*
