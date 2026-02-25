# Contributing to OpenSTR

Thank you for your interest in contributing to the OpenSTR protocol. This document explains how to get involved, what kinds of contributions are welcome, and how the process works.

---

## Before you start

Please read [`docs/governance.md`](docs/governance.md) before contributing. It defines the project's roles, the RFC process, and the Spec Enhancement Proposal (SEP) mechanism that governs significant changes to the protocol.

---

## What contributions are welcome

**Issues** — bug reports, questions, implementation feedback, and SEP proposals are all welcome. Search existing issues before opening a new one to avoid duplicates.

**Pull requests** — welcome for:
- Editorial improvements to RFC text (typos, clarity, non-normative changes)
- Additions to vocabulary lists (amenity types, environment tags, property sub-types) that do not alter existing entries
- New entries in `docs/implementations.md`
- Examples, diagrams, or supporting documentation
- Implementations of accepted SEPs

**Spec Enhancement Proposals (SEPs)** — the mechanism for proposing significant changes to the protocol. See the SEP process in [`docs/governance.md`](docs/governance.md).

**Implementation reports** — if you have built a working implementation of any part of OpenSTR, please open an issue with the label `implementation-report` and describe what you built, what worked well, and where you encountered friction. Implementation feedback carries significant weight in SEP decisions.

---

## What requires a SEP

Before opening a pull request, check whether your change requires a SEP. A SEP is required for any change that alters normative behaviour — field definitions, endpoint behaviour, allowed values, credential requirements, or accreditation rules. See [`docs/governance.md`](docs/governance.md) Section 5.1 for the full list.

If you open a pull request for a change that requires a SEP without the SEP being accepted first, the pull request will be closed with a request to follow the SEP process.

---

## Pull request checklist

- [ ] References the relevant issue or SEP number
- [ ] Updates `CHANGELOG.md` with a brief description of the change
- [ ] Does not introduce breaking changes without an accepted SEP
- [ ] RFC text uses UK English spelling
- [ ] JSON examples are valid and consistent with the schema
- [ ] No unnecessary whitespace or formatting changes mixed with substantive changes

---

## Style guide

**Language:** UK English throughout. Favour "colour" over "color", "licence" (noun) over "license", "organisation" over "organization".

**RFC text:** Write in the present tense for normative statements ("The host system must return...") and the future tense for planned behaviour ("v0.2 will define..."). Use "must", "should", and "may" in their RFC 2119 senses.

**JSON examples:** Use 2-space indentation. All string values in examples should be realistic and self-explanatory rather than placeholder values like `"string"` or `123`.

**Field names:** snake_case throughout.

---

## Code of conduct

OpenSTR follows the [Contributor Covenant Code of Conduct v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). All contributors are expected to engage respectfully. Maintainers may remove, edit, or reject contributions that do not meet this standard.

---

## Getting help

If you have a question about the protocol or the contribution process, open an issue with the label `question`. Maintainers aim to respond within 7 days.

For security-related issues, do not open a public issue. Email [`hello@openstr.org`](mailto:hello@openstr.org) with the subject line `SECURITY:` and a description of the issue.
