# RFC: Identity and Trust

**RFC ID:** openstr-rfc-004  
**Title:** Host and Guest Identity, Credentials, and Trust  
**Status:** Draft  
**Version:** 0.1.0  
**Created:** February 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Supersedes:** None  

---

## Abstract

This RFC defines the OpenSTR identity and trust model — the bilateral credential system through which hosts and guests establish verifiable identity before a booking is confirmed. It specifies the `HostCredential` schema, the `GuestCredential` schema, the three-tier guest IDV level vocabulary, the cryptographic proof mechanism based on W3C Verifiable Credentials, the credential lifecycle including expiry and revocation, the trusted issuer registry governance model, and the accreditation requirements for HostCredential issuers. This RFC is the trust foundation on which the property listing, availability query, and booking confirmation RFCs depend.

---

## 1. Motivation and Problem Statement

Short-term rental platforms provide trust infrastructure as a core part of their value proposition. Airbnb, Vrbo, and Booking.com verify host and guest identities, maintain reputation histories, and act as a centralised arbitration layer when disputes arise. This trust infrastructure is one of the primary reasons hosts and guests accept the platform's commission costs and terms.

For an open, platform-independent protocol like OpenSTR to be credible, it must provide equivalent trust guarantees without requiring a central platform intermediary. A guest agent must be able to verify that a host is who they claim to be and has the legal right to let the property they are listing. A host system must be able to verify that a guest meets the identity requirements for their property. Both verifications must be possible cryptographically, without relying on a single trusted authority, and in a way that is portable across any OpenSTR-compliant implementation.

OpenSTR addresses this through a bilateral credential model: both hosts and guests hold verifiable credentials issued by recognised third-party authorities. These credentials are the machine-readable equivalent of a platform's verification badge — portable, cryptographically provable, and independent of any single platform.

---

## 2. Design Decisions

### 2.1 W3C Verifiable Credentials as Foundation

OpenSTR credentials conform to the W3C Verifiable Credentials Data Model 2.0. This is the established open standard for machine-readable, cryptographically verifiable identity claims. It defines the envelope (how a credential is structured and signed) without prescribing the contents (what claims are made). OpenSTR defines the required and optional claims within this envelope for both host and guest credentials.

Choosing W3C VC as the foundation means OpenSTR credentials are interoperable with any system that supports the VC standard, including existing digital identity wallets, IDV providers, and emerging agent identity frameworks.

### 2.2 Asymmetric Trust Architecture

Guest-side and host-side credentials have different trust requirements and different issuer ecosystems.

**Guest credentials** attest to personal identity. Established IDV providers — Stripe Identity, Onfido, Yoti, Apple, Google Identity, and others — already operate at scale in this space with robust KYC pipelines. OpenSTR defines the required claim structure and maintains a curated list of recognised guest credential issuers at `openstr.org/trusted-issuers`. Any recognised issuer may issue compliant guest credentials.

**Host credentials** must attest to both personal or business identity and property-specific right-to-let — a combination that no existing third-party registry covers today. OpenSTR defines the HostCredential schema and an accreditation framework for third-party registries that wish to issue compliant host credentials. openstr.org maintains a public registry of accredited HostCredential issuers. openstr.org does not operate as a HostCredential issuer itself — this preserves decentralisation and creates a commercial opportunity for third parties to build OpenSTR-compatible host verification services.

### 2.3 Three-Tier Guest IDV Vocabulary

Guest credentials carry one of three IDV levels, each representing a different depth of identity verification:

- `email_verified` — email address ownership confirmed by the issuer
- `identity_verified` — identity verified against a government-issued document (passport, national ID, or driving licence)
- `payment_verified` — payment method verified with a financial guarantee in place

Hosts declare the minimum IDV level required in the listing's `damage_guarantee.idv_level` field. Agents must confirm the guest credential meets or exceeds this level before initiating a booking request.

### 2.4 Expiry Plus Optional Revocation

All OpenSTR credentials carry an `expires_at` timestamp. Relying parties must treat expired credentials as invalid regardless of other fields.

Revocation before expiry is supported via the W3C Bitstring Status List specification. Issuers that support revocation include a `status_list` object in the credential pointing to the relevant entry in their published status bitfield. Relying parties that receive a credential with a `status_list` entry must check revocation status before accepting the credential. Issuers that do not support revocation may omit `status_list` — in this case relying parties rely on expiry only. Issuers offering `identity_verified` or `payment_verified` guest credentials, or any HostCredential, are strongly recommended to implement revocation.

### 2.5 Reputation Fields Reserved for v0.2

The HostCredential schema defines reserved fields for host reputation data — aggregate rating, response rate, cancellation rate, and verified review count — but these fields are not required in v0.1 and will only be populated once the review and reputation system is defined in RFC-005. Defining the schema fields now ensures forward compatibility and allows early issuers to build the data model correctly from the start.

---

## 3. Credential Structure

Both `HostCredential` and `GuestCredential` conform to the W3C Verifiable Credentials Data Model 2.0 envelope. The OpenSTR-specific claims are carried in the `credentialSubject` object.

### 3.1 W3C VC Envelope

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://openstr.org/contexts/v1"
  ],
  "type": ["VerifiableCredential", "OpenSTRHostCredential"],
  "id": "https://issuer.example.com/credentials/host-abc123",
  "issuer": "https://issuer.example.com",
  "validFrom": "2026-01-15T00:00:00Z",
  "validUntil": "2027-01-15T00:00:00Z",
  "credentialSubject": { },
  "credentialStatus": { },
  "proof": { }
}
```

The `type` array must include `VerifiableCredential` and either `OpenSTRHostCredential` or `OpenSTRGuestCredential`. The `@context` array must include both the W3C VC context and the OpenSTR context URI.

### 3.2 Proof Type

OpenSTR v0.1 requires support for `Ed25519Signature2020` as the proof type. This is a widely supported, compact, and efficient digital signature scheme based on the Ed25519 elliptic curve. Issuers may additionally support other proof types (e.g. `JsonWebSignature2020`) but must include an `Ed25519Signature2020` proof for OpenSTR compliance.

```json
"proof": {
  "type": "Ed25519Signature2020",
  "created": "2026-01-15T00:00:00Z",
  "verificationMethod": "https://issuer.example.com/keys/1",
  "proofPurpose": "assertionMethod",
  "proofValue": "z3MvGebUd..."
}
```

The `verificationMethod` URI must resolve to a public key document accessible over HTTPS. Relying parties must fetch and cache this document to verify the proof signature.

---

## 4. GuestCredential

### 4.1 credentialSubject Fields

#### Required fields

| Field | Type | Description |
|---|---|---|
| `id` | string (URI) | A stable, privacy-preserving identifier for this guest. Must not be a real-world identifier such as an email address or national ID number. A DID or opaque URI issued by the issuer is appropriate. |
| `idv_level` | enum | The IDV level attested by this credential: `email_verified`, `identity_verified`, or `payment_verified` |
| `idv_method` | string | Human-readable description of the verification method used by the issuer (e.g. `"Passport verified via NFC chip read"`, `"Government ID document check plus liveness detection"`) |
| `issuer_guest_id` | string | The issuer's internal reference for this guest, opaque to the host. Enables the issuer to track revocation and reputation without exposing personal data. |

#### Optional fields

| Field | Type | Description |
|---|---|---|
| `country_of_residence` | string | ISO 3166-1 alpha-2 country code. Included only with guest consent. |
| `age_range` | enum | `18_24`, `25_34`, `35_44`, `45_54`, `55_64`, `65_plus`. Included only with guest consent. Allows age-restricted properties to filter without disclosing exact date of birth. |
| `phone_verified` | boolean | Whether the guest's phone number has been verified by the issuer |
| `payment_method_type` | enum | `credit_card`, `debit_card`, `bank_account`. Present only for `payment_verified` credentials. Does not include card numbers or account details. |

### 4.2 IDV Level Definitions

#### `email_verified`

The issuer has confirmed that the guest controls the email address associated with this credential, via a confirmation link or OTP. No identity document check is performed. Appropriate for low-risk properties with no damage guarantee or `mode: none`.

#### `identity_verified`

The issuer has verified the guest's identity against a government-issued document — passport, national identity card, or driving licence — using a recognised document verification method. The method must include liveness detection to prevent spoofing with a photograph. The issuer must retain a record of the document check sufficient to support a legal dispute. This level is equivalent to the identity verification required by major STR platforms. Appropriate for properties with `damage_guarantee.mode: idv_guarantee` and `idv_level: identity_verified`.

#### `payment_verified`

The issuer has both verified the guest's identity to `identity_verified` standard and confirmed that a valid payment method is associated with the credential, with a financial guarantee in place. The financial guarantee means the issuer accepts liability up to a declared limit in the event of guest-caused damage where the guest cannot be recovered against. This is the highest trust tier and appropriate for high-value properties or those requiring maximum assurance. `payment_verified` implies `identity_verified` — a host requiring `identity_verified` must accept a `payment_verified` credential.

### 4.3 IDV Level Hierarchy

`payment_verified` ≥ `identity_verified` > `email_verified`

A host declaring `idv_level: identity_verified` must accept credentials at `identity_verified` or `payment_verified`. A host declaring `idv_level: email_verified` must accept all three levels.

### 4.4 GuestCredential Example

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://openstr.org/contexts/v1"
  ],
  "type": ["VerifiableCredential", "OpenSTRGuestCredential"],
  "id": "https://identity.example-issuer.com/credentials/guest-7x9k2",
  "issuer": "https://identity.example-issuer.com",
  "validFrom": "2026-01-10T00:00:00Z",
  "validUntil": "2027-01-10T00:00:00Z",
  "credentialSubject": {
    "id": "did:example:guest-7x9k2",
    "idv_level": "identity_verified",
    "idv_method": "Passport verified via NFC chip read with liveness detection",
    "issuer_guest_id": "igi-8k2p9x",
    "country_of_residence": "US",
    "age_range": "35_44",
    "phone_verified": true
  },
  "credentialStatus": {
    "id": "https://identity.example-issuer.com/status/2026-01#94372",
    "type": "BitstringStatusListEntry",
    "statusPurpose": "revocation",
    "statusListIndex": "94372",
    "statusListCredential": "https://identity.example-issuer.com/status/2026-01"
  },
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "2026-01-10T00:00:00Z",
    "verificationMethod": "https://identity.example-issuer.com/keys/1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z3MvGebUd4QrNpX7kLsT9wJm..."
  }
}
```

---

## 5. HostCredential

### 5.1 credentialSubject Fields

#### Required fields

| Field | Type | Description |
|---|---|---|
| `id` | string (URI) | Stable identifier for this host. A DID or opaque URI issued by the issuer. |
| `host_type` | enum | `individual` or `business` |
| `identity_verified` | boolean | Whether the host's identity has been verified by the issuer against a government-issued document (individual) or company registration documents (business) |
| `identity_verified_at` | string (ISO 8601 datetime) | Timestamp of identity verification |
| `properties` | array[object] | One or more verified property objects. See Section 5.2. At least one property must be present. |
| `issuer_host_id` | string | The issuer's internal reference for this host, opaque to agents. |

#### Optional fields

| Field | Type | Description |
|---|---|---|
| `business_name` | string | Registered business name, for `host_type: business` |
| `business_registration_number` | string | Company registration number, for `host_type: business` |
| `business_country` | string | ISO 3166-1 alpha-2 country code of business registration |
| `phone_verified` | boolean | Whether the host's phone number has been verified |
| `reputation` | object | Host reputation summary. Schema defined here; populated in v0.2. See Section 5.3. |

### 5.2 Property Verification Object

Each entry in the `properties` array represents a single property for which the host has been verified as having the right to let.

#### Required property fields

| Field | Type | Description |
|---|---|---|
| `listing_id` | string | The OpenSTR `listing_id` this property record corresponds to |
| `property_ref` | string | The issuer's internal reference for this property |
| `right_to_let_verified` | boolean | Whether the host's right to let this property has been verified by the issuer. Must be `true` for a compliant HostCredential. |
| `right_to_let_verified_at` | string (ISO 8601 datetime) | Timestamp of right-to-let verification |
| `right_to_let_method` | string | Human-readable description of how right-to-let was verified (e.g. `"Title deeds reviewed"`, `"Landlord consent letter verified"`, `"Leasehold agreement with subletting clause confirmed"`) |
| `address_verified` | boolean | Whether the property address has been verified against an authoritative source |
| `country` | string | ISO 3166-1 alpha-2 country code of property location |

#### Optional property fields

| Field | Type | Description |
|---|---|---|
| `local_registration_number` | string | Short-term let registration or licence number where required by local regulations (e.g. Edinburgh, Barcelona, Amsterdam, Paris) |
| `local_registration_verified` | boolean | Whether the local registration number has been verified as current and valid by the issuer |
| `planning_permission_reference` | string | Planning permission or change-of-use reference where applicable |
| `property_reputation` | object | Per-property reputation summary. Schema defined here; populated in v0.2. See Section 5.3. |

### 5.3 Reputation Object (Schema Reserved — Populated in v0.2)

The reputation object schema is defined here for forward compatibility. Issuers building HostCredential infrastructure should include this object in their data model from the start. In v0.1 this object may be omitted or present with null values. Reputation data will only be meaningful once the review and reputation RFC (RFC-005, v0.2) is implemented.

#### Host-level reputation fields

| Field | Type | Description |
|---|---|---|
| `aggregate_rating` | number | Average host rating across all properties and stays. Scale 1.0–5.0. |
| `rating_count` | integer | Number of verified ratings contributing to aggregate_rating |
| `response_rate_pct` | number | Percentage of booking enquiries responded to within 24 hours |
| `cancellation_rate_pct` | number | Percentage of confirmed bookings cancelled by the host |
| `reputation_updated_at` | string (ISO 8601 datetime) | Timestamp of last reputation update |

#### Property-level reputation fields

| Field | Type | Description |
|---|---|---|
| `property_rating` | number | Average rating for this specific property. Scale 1.0–5.0. |
| `property_rating_count` | integer | Number of verified ratings for this property |
| `verified_stays` | integer | Total number of confirmed and completed OpenSTR bookings at this property |

### 5.4 HostCredential Example

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://openstr.org/contexts/v1"
  ],
  "type": ["VerifiableCredential", "OpenSTRHostCredential"],
  "id": "https://hostregistry.example.com/credentials/host-abc123",
  "issuer": "https://hostregistry.example.com",
  "validFrom": "2026-01-15T00:00:00Z",
  "validUntil": "2027-01-15T00:00:00Z",
  "credentialSubject": {
    "id": "did:example:host-abc123",
    "host_type": "individual",
    "identity_verified": true,
    "identity_verified_at": "2026-01-14T10:30:00Z",
    "issuer_host_id": "ihi-3m7q1r",
    "properties": [
      {
        "listing_id": "host-abc123-prop-001",
        "property_ref": "prop-chi-lp-001",
        "right_to_let_verified": true,
        "right_to_let_verified_at": "2026-01-14T11:00:00Z",
        "right_to_let_method": "Freehold title deeds reviewed and address confirmed against land registry",
        "address_verified": true,
        "country": "US",
        "local_registration_number": "CHI-STR-2026-004821",
        "local_registration_verified": true
      }
    ],
    "reputation": null
  },
  "credentialStatus": {
    "id": "https://hostregistry.example.com/status/2026-01#12847",
    "type": "BitstringStatusListEntry",
    "statusPurpose": "revocation",
    "statusListIndex": "12847",
    "statusListCredential": "https://hostregistry.example.com/status/2026-01"
  },
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "2026-01-15T00:00:00Z",
    "verificationMethod": "https://hostregistry.example.com/keys/1",
    "proofPurpose": "assertionMethod",
    "proofValue": "zQ7rNpX4kMsT2wJm9vGebUd8..."
  }
}
```

---

## 6. Credential Lifecycle

### 6.1 Issuance

Credentials are issued by a recognised issuer following successful verification. The issuer signs the credential with their Ed25519 private key and publishes it at the `id` URI declared in the credential. The subject (guest or host) receives a copy of the signed credential for presentation to relying parties.

### 6.2 Expiry

All OpenSTR credentials must include a `validUntil` timestamp. Relying parties must reject credentials where `validUntil` is in the past. Recommended maximum validity periods:

| Credential type | Recommended maximum validity |
|---|---|
| `email_verified` GuestCredential | 2 years |
| `identity_verified` GuestCredential | 1 year |
| `payment_verified` GuestCredential | 1 year |
| HostCredential | 1 year |

Issuers may use shorter validity periods. They must not use longer periods without implementing revocation.

### 6.3 Revocation

Credential revocation allows an issuer to invalidate a credential before its `validUntil` date. OpenSTR v0.1 supports revocation via the W3C Bitstring Status List specification.

#### 6.3.1 Bitstring Status List

The issuer publishes a `BitstringStatusListCredential` — itself a signed W3C VC — at a stable HTTPS URI. This credential contains a compressed bitfield where each bit position corresponds to a credential issued by the issuer. A bit value of `1` indicates the credential at that index is revoked. A bit value of `0` indicates it is not revoked.

Each revocable credential includes a `credentialStatus` object:

```json
"credentialStatus": {
  "id": "https://issuer.example.com/status/2026-01#94372",
  "type": "BitstringStatusListEntry",
  "statusPurpose": "revocation",
  "statusListIndex": "94372",
  "statusListCredential": "https://issuer.example.com/status/2026-01"
}
```

#### 6.3.2 Revocation Check Process

When a relying party (host system or agent) receives a credential with a `credentialStatus` entry, it must:

1. Fetch the `statusListCredential` URI
2. Verify the status list credential's proof signature
3. Decode the compressed bitfield
4. Check the bit at `statusListIndex`
5. Reject the credential if the bit is `1` (revoked)

Relying parties should cache the status list credential with appropriate TTL headers. Status lists should be refreshed at least once per hour for `identity_verified` and `payment_verified` credentials.

#### 6.3.3 Revocation Without Status List

Issuers that do not implement Bitstring Status List must omit the `credentialStatus` field. In this case relying parties rely on `validUntil` only. Such issuers must use validity periods of no more than 90 days for `identity_verified` and `payment_verified` credentials. Issuers seeking accreditation as HostCredential issuers must implement revocation — it is not optional for host credentials.

### 6.4 Rotation and Re-issuance

Guests and hosts may request re-issuance of their credential at any time before expiry, for example following a change of address, renewed identity document, or addition of a new property. Re-issued credentials carry a new `id` URI and a new `validFrom` timestamp. The previous credential is not automatically revoked on re-issuance — the issuer should revoke it explicitly if it is no longer valid.

---

## 7. Trusted Issuer Registry

### 7.1 Guest Credential Issuers

OpenSTR maintains a public registry of recognised guest credential issuers at `https://openstr.org/trusted-issuers/guest`. Any IDV provider that meets the following requirements may apply for inclusion:

- Operates a publicly documented identity verification process
- Supports issuance of W3C Verifiable Credentials with `Ed25519Signature2020` proof
- Issues credentials conforming to the OpenSTR `GuestCredential` schema
- Maintains a public verification key document at a stable HTTPS URI
- Publishes a privacy policy describing how guest data is handled
- Accepts liability for the accuracy of credentials issued at `identity_verified` or `payment_verified` level

For `email_verified` credentials, the bar is intentionally low — many services can issue these. For `identity_verified` and `payment_verified`, issuers must additionally demonstrate:

- Integration with a recognised document verification provider or government identity service
- Liveness detection capability for document checks
- A revocation mechanism (Bitstring Status List strongly recommended)
- Compliance with applicable data protection regulations in their operating jurisdiction

Candidate issuers for the initial registry include: Stripe Identity, Onfido, Yoti, Persona, Jumio, Apple (via Apple ID), and Google (via Google Identity). OpenSTR makes no endorsement of specific providers and the registry is open to any qualifying issuer.

### 7.2 HostCredential Issuers

OpenSTR maintains a separate registry of accredited HostCredential issuers at `https://openstr.org/trusted-issuers/host`. The bar for accreditation is higher than for guest credential issuers, reflecting the additional responsibility of property-level verification and right-to-let attestation.

#### Bootstrap note

At the time of writing, no third-party HostCredential issuers exist — the service category is new and specific to the OpenSTR protocol. To solve this bootstrap problem without compromising the independence of openstr.org as a standards body, a separate certification company is planned to operate as the first accredited HostCredential issuer under the openstr.org accreditation scheme. This company will be legally and operationally independent of openstr.org, subject to the same accreditation requirements as any future issuer, and will serve as the reference implementation of what a compliant HostCredential issuer looks like in practice.

This two-entity model — openstr.org as standards body, a separate company as bootstrap issuer — deliberately mirrors the PCI DSS model, where the PCI Security Standards Council defines the standard and Qualified Security Assessors are independent certified companies that perform the actual compliance work commercially. openstr.org has no financial interest in the certification company and the accreditation process is governed independently to prevent conflicts of interest.

Early adopters wishing to obtain a HostCredential should monitor `https://openstr.org/trusted-issuers/host` for the listing of the first accredited issuer. Hosts may self-publish an unaccredited listing in the interim — agents should surface the absence of a verified HostCredential to users rather than treating it as a blocking error during the bootstrap phase.

#### Accreditation requirements for HostCredential issuers

**Identity verification:**
- Must verify individual host identity against a government-issued document, or business host identity against company registration documents in the host's jurisdiction
- Must implement liveness detection for individual identity checks

**Property verification — right-to-let:**
- Must verify the host's legal right to let each property declared in the credential
- Acceptable verification methods include: review of title deeds or land registry records, review of a lease agreement containing a subletting clause, review of a landlord consent letter, or confirmation of a valid property management agreement
- The verification method used must be recorded and declared in `right_to_let_method`
- Verification must be repeated if a property changes ownership or the host's tenancy changes

**Property verification — address:**
- Must verify the property address against an authoritative source such as a national address register, land registry, or equivalent

**Local registration:**
- Where the property is located in a jurisdiction that requires short-term let registration or licensing (e.g. Chicago, Edinburgh, Barcelona, Amsterdam, Paris, New York), the issuer must verify that the registration number declared in `local_registration_number` is current and valid before including it in the credential

**Technical requirements:**
- Must support W3C Verifiable Credentials with `Ed25519Signature2020` proof
- Must issue credentials conforming to the OpenSTR `HostCredential` schema
- Must implement Bitstring Status List revocation — this is mandatory for HostCredential issuers
- Must refresh revocation status within 24 hours of a revocation event

**Governance:**
- Must accept OpenSTR's accreditation terms and conditions
- Must notify openstr.org within 48 hours of any security incident affecting credential integrity
- Must cooperate with OpenSTR dispute resolution processes in v0.3 and later

### 7.3 Registry Governance

The trusted issuer registries are maintained by openstr.org. Registry decisions — additions, removals, and requirement changes — are made by the OpenSTR maintainers. Community input on registry decisions is welcome via the GitHub repository. Issuers may be removed from the registry if they fail to maintain accreditation requirements or if a material security or compliance failure is identified.

The governance model will be reviewed at v1.0 with a view to transitioning to a community-governed model as the protocol matures.

---

## 8. Verification Procedures

### 8.1 Agent Verifying a HostCredential

Before submitting a booking request, the guest's agent must verify the HostCredential referenced in the property listing:

1. Fetch the credential document at `host_credential.credential_uri`
2. Confirm `type` includes `OpenSTRHostCredential`
3. Confirm `validUntil` is in the future
4. Confirm `issuer` URI is present in the OpenSTR host trusted issuer registry
5. Fetch the issuer's verification key at the URI declared in `proof.verificationMethod`
6. Verify the `Ed25519Signature2020` proof against the credential document
7. If `credentialStatus` is present, perform a Bitstring Status List revocation check
8. Confirm the `properties` array contains an entry whose `listing_id` matches the listing being booked
9. Confirm `right_to_let_verified` is `true` for that property entry

If any step fails, the agent must not proceed with the booking and must notify the user that the host credential could not be verified.

### 8.2 Host System Verifying a GuestCredential

Upon receiving a booking request, the host system must verify the GuestCredential:

1. Fetch the credential document at `guest_credential.credential_uri`
2. Confirm `type` includes `OpenSTRGuestCredential`
3. Confirm `validUntil` is in the future
4. Confirm `issuer` URI is present in the OpenSTR guest trusted issuer registry
5. Fetch the issuer's verification key at the URI declared in `proof.verificationMethod`
6. Verify the `Ed25519Signature2020` proof against the credential document
7. If `credentialStatus` is present, perform a Bitstring Status List revocation check
8. Confirm `credentialSubject.idv_level` meets or exceeds the listing's `damage_guarantee.idv_level` requirement (applying the IDV hierarchy: `payment_verified` ≥ `identity_verified` > `email_verified`)

If any step fails, the host system must return `403 Forbidden` with an appropriate error description.

---

## 9. Privacy Considerations

**9.1 Minimal disclosure.** The GuestCredential is designed to disclose the minimum personal information necessary. It does not contain the guest's name, date of birth, address, or document number. The `issuer_guest_id` is an opaque reference — the host receives proof of verification without receiving the underlying personal data.

**9.2 Guest consent for optional fields.** The optional `country_of_residence` and `age_range` fields may only be included in a GuestCredential with the explicit consent of the guest. Issuers must not include these fields without consent.

**9.3 Issuer surveillance risk.** When a relying party fetches a credential at `credential_uri`, the issuer learns that the credential has been presented. This is a privacy risk mitigated by the Bitstring Status List approach — revocation checks against a status list do not reveal which specific credential is being checked. Issuers are encouraged to minimise logging of credential fetch events.

**9.4 Host data minimisation.** The HostCredential contains property address and right-to-let verification data. Issuers must not include more property data than is necessary for verification purposes. The verified address in the HostCredential is the issuer's internal record — it is not exposed to agents or guests.

**9.5 Data retention.** Issuers must declare their credential data retention policies in their published privacy policy. Guest identity documents used for `identity_verified` verification must not be retained longer than required by applicable law.

---

## 10. Security Considerations

**10.1 Key management.** Issuer private keys must be stored in hardware security modules (HSMs) or equivalent secure key management infrastructure. Key rotation must be supported — old keys must remain valid for verification of credentials issued under them until those credentials expire.

**10.2 Credential document integrity.** Credential documents published at `credential_uri` must be immutable once issued. The proof signature covers the full document — any modification invalidates the proof. Issuers must not modify credential documents after issuance.

**10.3 Man-in-the-middle.** All credential fetch operations must use HTTPS with valid certificates. Relying parties must reject credentials fetched over plain HTTP or from URIs with invalid certificates.

**10.4 Replay attacks.** A credential presented in one booking request should not be reusable in a fabricated second request. The booking flow's `quote_id` mechanism (defined in `rfc.booking_confirmation.md`) mitigates this — the credential alone is not sufficient to create a booking without a valid, unexpired quote.

**10.5 Issuer compromise.** If an accredited issuer's signing key is compromised, all credentials issued by that issuer must be considered invalid. openstr.org must be able to remove a compromised issuer from the trusted issuer registry within 24 hours of notification. Relying parties should re-validate cached issuer trust status at least once per day.

---

## 11. Open Questions

**11.1 Decentralised Identifiers (DIDs).** This RFC uses DID URIs as credential subject identifiers in examples but does not mandate a specific DID method. A future SEP may define a preferred DID method for OpenSTR credentials.

**11.2 Selective disclosure.** W3C VC supports selective disclosure mechanisms (e.g. BBS+ signatures) that allow a guest to prove a claim without revealing the full credential. This is not required in v0.1 but is a privacy-enhancing extension worth considering for v1.0.

**11.3 Cross-issuer reputation portability.** In v0.2, reputation data will be anchored to OpenSTR booking references. How reputation data from bookings made under different issuers is aggregated and ported when a guest switches issuers is an open question.

**11.4 Registry decentralisation.** The trusted issuer registries are maintained centrally by openstr.org in v0.1. A decentralised governance model — potentially using a community vote or on-chain registry — is proposed for consideration at v1.0.

**11.5 Local registration coverage.** The list of jurisdictions requiring short-term let registration is growing rapidly. A maintained, machine-readable list of registration requirements by jurisdiction is proposed as a community resource at `openstr.org/local-regulations`.

**11.6 Business guest credentials.** The GuestCredential schema is designed for individual guests. A business travel variant — where a company credential covers multiple employees — is not defined in v0.1 and is deferred to a future SEP.

---

## 12. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |

---

## 13. References

- OpenSTR `rfc.property_listing.md` — Property listing schema and discovery
- OpenSTR `rfc.availability_query.md` — Availability and pricing query
- OpenSTR `rfc.booking_confirmation.md` — Booking request, confirmation and cancellation
- [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/)
- [W3C Bitstring Status List](https://www.w3.org/TR/vc-bitstring-status-list/)
- [Ed25519Signature2020](https://w3c.github.io/vc-di-eddsa/)
- [Decentralised Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-core/)
- [ISO 3166-1 alpha-2 country codes](https://www.iso.org/iso-3166-country-codes.html)
- [ISO 8601 — Date and time format](https://www.iso.org/iso-8601-date-and-time-format.html)
- [Stripe Identity](https://stripe.com/identity)
- [Onfido](https://onfido.com)
- [Yoti](https://www.yoti.com)
