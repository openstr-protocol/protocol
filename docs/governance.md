# OpenSTR Governance

**Document:** docs/governance.md  
**Version:** 0.1.0  
**Status:** Active  
**Authors:** Daniel Bloom (openstr.org)  
**Last updated:** February 2026  

---

## 1. Purpose

This document defines how the OpenSTR protocol is governed — who maintains it, how changes are proposed and decided, what the contribution process looks like, and how the project will evolve its governance model as it matures. It exists to make the project's decision-making transparent and predictable for contributors, implementers, and adopters.

---

## 2. Principles

**Transparency.** All significant decisions about the protocol are made in public, with reasoning documented. Decisions made in private — even correct ones — undermine trust in an open standard.

**Stability for implementers.** People building on OpenSTR need to trust that the protocol will not change arbitrarily. The change process is designed to be deliberate, with adequate notice and a clear rationale for every breaking change.

**Low barriers to contribution.** Anyone can propose a change, report an issue, or submit an implementation report. You do not need to be invited or approved to participate in the public process.

**Separation of standards and commerce.** openstr.org governs the protocol specification. It does not operate as a booking platform, payment processor, or credential issuer. Commercial activity in the OpenSTR ecosystem — including the certification company and any hosted tooling — operates independently of the standards body.

**Honest about scale.** In the early phase of this project, governance is intentionally lightweight. A small maintainer group makes decisions efficiently. The governance model will evolve toward broader community participation as the contributor base grows.

---

## 3. Roles

### 3.1 Maintainers

Maintainers are responsible for the day-to-day health of the protocol. They review contributions, make decisions on Spec Enhancement Proposals, merge approved changes, and maintain the trusted issuer registries.

**Current maintainers:**

- Daniel Bloom ([@danielbloom](https://github.com/danielbloom)) — founding maintainer

New maintainers may be appointed by Daniel Bloom at any time. Appointments are announced in the repository and recorded in this document. There is no minimum number of maintainers required, but the project aims to have at least two active maintainers at all times to avoid single points of failure.

Maintainers may step down at any time by notifying the other maintainers and opening a pull request to update this document. A maintainer who has been inactive for six months with no response to maintainer-directed communications may be moved to emeritus status.

### 3.2 Contributors

Anyone who opens an issue, submits a pull request, or participates in a SEP discussion is a contributor. No approval or invitation is required. Contributors are listed in `CONTRIBUTORS.md`.

### 3.3 Implementers

Implementers are individuals or organisations who have built a working implementation of the OpenSTR protocol — host-side endpoints, guest-side agent integrations, or tooling. Implementers are encouraged to register their implementations by opening a pull request to add themselves to `docs/implementations.md`. Registered implementers carry additional weight in SEP discussions because their feedback is grounded in real deployment experience.

### 3.4 Accredited Issuers

Accredited HostCredential and GuestCredential issuers operate under the accreditation framework defined in `rfc.identity_trust.md`. Accreditation decisions are made by the maintainers. Issuers are listed in the trusted issuer registries at `openstr.org/trusted-issuers/guest` and `openstr.org/trusted-issuers/host`. Accredited issuers are not maintainers of the protocol by virtue of their accreditation.

---

## 4. The RFC Process

Core protocol documents are called RFCs (Requests for Comments). RFCs define the normative behaviour of the OpenSTR protocol. The current RFCs are:

| RFC | Title | Status |
|---|---|---|
| `rfc.property_listing.md` | Property Listing Schema and Discovery | Draft |
| `rfc.availability_query.md` | Availability and Pricing Query | Draft |
| `rfc.booking_confirmation.md` | Booking Request, Confirmation, and Cancellation | Draft |
| `rfc.identity_trust.md` | Host and Guest Identity, Credentials, and Trust | Draft |

RFCs progress through the following statuses:

| Status | Meaning |
|---|---|
| `Draft` | Actively being developed. Breaking changes expected. |
| `Candidate` | Feature-complete for the target version. Seeking implementation feedback. Breaking changes require strong justification. |
| `Stable` | Ratified for the target version. Breaking changes not permitted without a new version increment. |
| `Superseded` | Replaced by a newer RFC version. |
| `Withdrawn` | Abandoned without replacement. |

New RFCs may be proposed by any contributor via the SEP process (Section 5). The maintainers decide whether a proposed RFC is accepted into the core protocol.

---

## 5. Spec Enhancement Proposals (SEPs)

A Spec Enhancement Proposal (SEP) is the formal mechanism for proposing significant changes to the OpenSTR protocol. All breaking changes, new RFCs, and substantive additions to existing RFCs must go through the SEP process. Minor editorial changes — typo fixes, clarifications that do not alter normative behaviour — may be submitted as pull requests without a SEP.

### 5.1 When a SEP is required

A SEP is required for:

- Any change that alters the required fields, field types, or allowed values in a schema
- Any change that alters the defined behaviour of an endpoint
- Any new RFC or major RFC revision
- Any change to the trusted issuer accreditation requirements
- Any change to this governance document
- Any change to the versioning or compatibility model

A SEP is not required for:

- Correcting a factual error or typo
- Improving the clarity of non-normative text (examples, rationale, notes)
- Adding entries to the amenity vocabulary, environment tag vocabulary, or sub-type vocabulary that do not alter existing entries
- Updating the implementations list or contributor list

If you are unsure whether your change requires a SEP, open an issue and ask.

### 5.2 SEP template

Open a GitHub issue with the label `SEP` and the following structure:

```
Title: SEP-NNNN: [Short descriptive title]

## Summary
One paragraph describing the proposed change and its motivation.

## Motivation
Why is this change needed? What problem does it solve?
Reference any relevant issues, implementation reports, or prior discussion.

## Proposed change
Describe the change precisely. For schema changes, show the before and after.
For new endpoints, describe the full request and response.

## Backward compatibility
Is this a breaking change? If so, what is the migration path?
If not, explain why existing implementations are unaffected.

## Alternatives considered
What other approaches were considered and why were they rejected?

## Open questions
List any unresolved questions that need community input.
```

The SEP number (`NNNN`) is assigned by a maintainer when the issue is opened. SEPs are numbered sequentially from SEP-0001.

### 5.3 SEP decision process

1. **Proposal** — contributor opens a GitHub issue using the SEP template
2. **Triage** — a maintainer assigns a SEP number and labels the issue within 7 days
3. **Public comment period** — 21 days from the date the SEP is labelled. Anyone may comment. The proposer is encouraged to engage with feedback and update the proposal accordingly.
4. **Decision** — after the comment period closes, the maintainers make a decision: `accepted`, `rejected`, or `deferred`. The decision is posted as a comment on the issue with documented reasoning.
5. **Implementation** — if accepted, the proposer or a maintainer opens a pull request implementing the change. The pull request references the SEP issue.
6. **Merge** — a maintainer reviews and merges the pull request. The SEP issue is closed and the SEP status is recorded in `docs/seps.md`.

Maintainers may fast-track a SEP — skipping or shortening the comment period — only for security fixes or corrections to clear errors. Fast-tracked SEPs are labelled `fast-track` and the reason for fast-tracking is documented.

### 5.4 SEP status vocabulary

| Status | Meaning |
|---|---|
| `open` | Comment period active |
| `accepted` | Approved for implementation |
| `rejected` | Declined, with documented reasoning |
| `deferred` | Not rejected, but not scheduled — may be revisited |
| `withdrawn` | Withdrawn by the proposer |
| `implemented` | Merged into the protocol |

---

## 6. Versioning

OpenSTR uses semantic versioning with the following interpretation:

| Component | Meaning |
|---|---|
| **Major** (e.g. `1.0`) | Stable, ratified release. Breaking changes from the previous major version are documented in a migration guide. |
| **Minor** (e.g. `0.2`) | New RFCs or significant new optional features added. Backward compatible with the previous minor version. |
| **Patch** (e.g. `0.1.1`) | Clarifications, editorial changes, or non-breaking additions to existing RFCs. |

The current version is `0.1` — an early draft. During the `0.x` series, breaking changes may occur between minor versions. Implementers should treat `0.x` as unstable and monitor the changelog.

The `1.0` release will be the first stable version. It will be preceded by a `0.x` candidate phase during which the RFCs are declared `Candidate` status and a public implementation period of at least 90 days is observed before ratification.

### 6.1 Deprecation policy

From `1.0` onwards, fields or behaviours may be deprecated in a minor version and removed in the next major version. Deprecated items are marked in the RFC with a deprecation notice and the version in which removal is planned. No deprecation and removal in the same version is permitted.

### 6.2 Changelog

All changes are recorded in `CHANGELOG.md` at the repository root. Each entry references the relevant SEP or pull request.

---

## 7. Contribution Guidelines

### 7.1 Opening issues

Issues are welcome for bug reports, questions, implementation feedback, and SEP proposals. Before opening an issue, search existing issues to avoid duplicates. Use the appropriate label:

| Label | Use for |
|---|---|
| `bug` | Errors or contradictions in the RFC text |
| `question` | Questions about the protocol |
| `SEP` | Spec Enhancement Proposals |
| `implementation-report` | Reports from people building on the protocol |
| `editorial` | Typos, formatting, non-normative text improvements |
| `trusted-issuer` | Questions or applications related to the issuer registry |

### 7.2 Pull requests

Pull requests are welcome for editorial changes, vocabulary additions, and implementations of accepted SEPs. For any change that requires a SEP, open the SEP issue first and get it accepted before opening a pull request.

Pull request checklist:
- References the relevant issue or SEP
- Updates the changelog
- Does not introduce breaking changes without an accepted SEP
- RFC text uses UK English spelling
- JSON examples are valid and consistent with the schema

### 7.3 Code of conduct

OpenSTR follows the [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). All contributors are expected to engage respectfully. Maintainers may remove, edit, or reject contributions that violate this standard.

---

## 8. Trusted Issuer Registry Governance

The trusted issuer registries — guest credential issuers at `openstr.org/trusted-issuers/guest` and host credential issuers at `openstr.org/trusted-issuers/host` — are maintained by the OpenSTR maintainers.

### 8.1 Applying for inclusion

Issuers wishing to be listed must open a GitHub issue with the label `trusted-issuer` and provide:

- Organisation name and website
- Credential type (GuestCredential or HostCredential)
- Evidence of meeting the accreditation requirements defined in `rfc.identity_trust.md`
- A publicly accessible verification key document
- A published privacy policy
- Contact details for security notifications

Maintainers will review the application and respond within 30 days. Applications may be accepted, rejected with documented reasoning, or returned with requests for additional information.

### 8.2 Removal from the registry

Issuers may be removed from the registry if they:

- Fail to maintain the accreditation requirements
- Experience a material security or compliance failure affecting credential integrity
- Cease to operate or become unresponsive to maintainer communications
- Violate the terms under which they were accredited

Removal decisions are made by the maintainers and documented. Issuers will be given 14 days notice before removal except in cases of active security incidents where immediate removal may be necessary.

### 8.3 Registry governance evolution

Registry governance will be reviewed at v1.0 with a view to establishing a more independent oversight process, potentially involving a technical advisory committee that includes representation from accredited issuers, implementers, and the broader community.

---

## 9. Governance Evolution

This governance model is intentionally lightweight for the early phase of the project. As the contributor base, implementation count, and issuer ecosystem grow, the model will evolve. Planned evolution milestones:

**At v0.3 or when 3+ active implementers exist:**
Review the SEP decision process. Consider moving from maintainer decision to rough consensus among active contributors with maintainer tiebreaker.

**At v1.0:**
Establish a formal Technical Steering Committee (TSC) with representation from implementers and accredited issuers. Transfer day-to-day governance to the TSC while openstr.org retains final authority over the trusted issuer registries. Review whether to adopt dual-licensing — CC BY 4.0 for RFC documents, Apache 2.0 for reference code — which is more precise for a specification project and consistent with how W3C and the OpenAPI Initiative licence their work.

**Post v1.0:**
Explore transfer of the protocol to an established standards body or foundation if community scale warrants it.

All governance changes require a SEP and follow the standard 21-day comment period.

---

## 10. Relationship Between openstr.org and Commercial Entities

openstr.org is the standards body for the OpenSTR protocol. It does not operate as a booking platform, payment processor, or credential issuer. Commercial entities — including the planned bootstrap HostCredential certification company and any hosted endpoint services — operate independently of openstr.org under the accreditation scheme defined in `rfc.identity_trust.md`.

openstr.org has no financial interest in any commercial entity operating in the OpenSTR ecosystem. Maintainers who have a financial interest in a commercial entity affected by a proposed SEP must declare that interest when participating in the decision process. Maintainers with a declared conflict of interest may not cast a deciding vote on the relevant SEP, though they may contribute to the discussion.

---

## 11. References

- `rfc.identity_trust.md` — accreditation requirements for trusted issuers
- `CHANGELOG.md` — protocol change history
- `CONTRIBUTORS.md` — contributor list
- `docs/implementations.md` — registered implementations
- `docs/seps.md` — SEP index and status
- [Contributor Covenant Code of Conduct v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/)
