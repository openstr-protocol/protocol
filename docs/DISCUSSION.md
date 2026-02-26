# Design Discussion: Booking Amendments (v0.2)

This document captures design considerations and open questions for the booking amendment endpoint planned for v0.2. It is intended to preserve real-world context and ensure the v0.2 design addresses genuine use cases rather than theoretical ones.

---

## Background

OpenSTR v0.1 does not include a booking amendment endpoint. All changes to a confirmed booking require cancellation and rebooking. This is documented in `rfc.booking_confirmation.md` as a known v0.1 limitation.

The decision to defer amendments was deliberate — a half-built amend endpoint that only handles zero-cost changes would create inconsistency and confusion for implementers. However, the cancel-and-rebook approach has real costs that a mature protocol should avoid:

- Two full payment transactions (refund + new charge) with associated provider fees on both
- Risk of the desired dates becoming unavailable in the window between cancellation and rebooking
- Poor guest experience, particularly for minor corrections made shortly after booking
- Potential cancellation policy penalties if the policy triggers on cancellation even when the guest immediately reBooks

These are not edge cases — they reflect common booking behaviour on existing platforms.

---

## Real-World Scenarios

The following scenarios were identified from operational experience with short-term rental platforms and inform the amendment type taxonomy below.

### Scenario A — Guest count correction, no price change

A guest searches for and books a property for 1 person. Shortly after confirming, they realise they forgot to include their partner. The property charges an extra guest fee above 4 guests; 2 guests falls within the base pricing tier. The correct total is identical to the original booking.

**Expected behaviour:** Guest count updated, no payment movement, confirmation re-issued with corrected guest count.

### Scenario B — Guest count correction with price change

Same as Scenario A but the additional guest pushes the booking above the extra guest fee threshold (e.g. from 4 to 5 guests where the fee applies above 4). The total increases.

**Expected behaviour:** Amendment triggers a top-up charge for the difference only. No cancellation, no full refund, no rebooking. Payment provider processes a partial charge against the original payment method.

### Scenario C — Pet addition

A guest forgets to declare a pet at the time of booking. The host charges a per-stay pet fee (e.g. £75). The guest contacts the host or amends via the booking interface after confirmation.

**Expected behaviour:** Pet fee added to the booking, top-up charge processed for the fee amount only. Booking reference unchanged, amended confirmation issued.

### Scenario D — Date change

A guest wishes to extend or shift their stay. The new dates may or may not be available. The new total may be higher or lower depending on nightly rate, length of stay, and applicable discounts.

**Expected behaviour:** Availability re-checked for new dates before amendment is accepted. If available, price difference calculated. If price increases, top-up charge processed. If price decreases, partial refund issued. Booking reference unchanged, amended confirmation issued with new dates.

---

## Amendment Type Taxonomy

Based on the scenarios above, amendments fall into three categories requiring different handling:

### Type 1 — Zero-cost amendment

Changes that do not alter the booking total.

Examples:
- Guest count change within the same pricing tier
- Correction of guest details (name, email)
- Addition or correction of special requests

Payment handling: none required. Simple record update and re-confirmation.

### Type 2 — Top-up amendment

Changes that increase the booking total.

Examples:
- Guest count increase crossing the extra guest fee threshold
- Pet addition where a pet fee applies
- Date extension at the same or higher nightly rate

Payment handling: partial charge for the difference only, processed against the original payment method or a new payment token. Must not trigger a full cancel/refund cycle.

### Type 3 — Repricing amendment

Changes that may increase or decrease the booking total, requiring full repricing.

Examples:
- Date change (any direction)
- Significant guest count change affecting discount tier (e.g. dropping below 7 nights loses weekly discount)

Payment handling: if total increases, top-up charge. If total decreases, partial refund. Requires new availability check before amendment is accepted.

---

## Open Design Questions

The following questions need resolution before the amendment endpoint can be specified.

### 1. Amendment window

Should amendments be permitted at any time before check-in, or only within a defined window (e.g. up to 24 hours after booking, or up to 48 hours before check-in)?

Different amendment types may warrant different windows — a guest name correction is low-risk at any point, while a date change close to check-in has operational implications for the host.

### 2. Host approval

Should amendments require host approval, or should some or all amendment types be processed automatically?

Suggested approach: zero-cost amendments auto-approve; top-up and repricing amendments are auto-approved up to defined thresholds, require host approval above them. This mirrors Airbnb's amendment model.

### 3. Payment settlement protocol

The v0.1 booking endpoint accepts a `payment_token` representing the full booking amount, delegating payment processing to ACP/AP2 handlers. An amendment endpoint needs to settle only the difference.

Options:
- **Differential token:** guest agent submits a new `payment_token` for the difference amount only
- **Amendment authorisation:** host system requests a charge against the original payment method via the original token reference
- **Escrow model:** full amended amount held, original amount released, difference settled — avoids partial charge complexity but requires escrow infrastructure

This is the most technically complex open question and will likely require input from ACP/AP2 maintainers.

### 4. Cancellation policy interaction

If a guest amends a booking (e.g. shortens their stay), does the cancellation policy apply to the removed nights? 

For example: a guest books 7 nights, then amends to 5 nights 3 days before check-in. Under a moderate policy, a cancellation at this point would attract a penalty. Should the 2 removed nights be treated as a partial cancellation subject to the same policy?

Suggested approach: define amendment as a separate event from cancellation. Cancellation policy applies only on full cancellation. A host-configurable `amendment_policy` field is added to the listing schema in v0.2, with a sensible default (e.g. no penalty for amendments made more than 48 hours before check-in).

### 5. Amended confirmation format

An amended booking should clearly distinguish itself from the original confirmation — both for the guest record and for any downstream systems (PMS, calendar sync, etc.).

Suggested approach: amended confirmations carry the original `booking_reference` plus an `amendment_number` (integer, incrementing). The full amendment history is retrievable via the status endpoint.

### 6. Idempotency

Amendment requests should be idempotent — submitting the same amendment twice should not result in double charges or double refunds. This requires an `amendment_request_id` field analogous to the `quote_id` in the booking flow.

---

## Proposed Endpoint Sketch (for discussion)

```
POST /openstr/booking/:reference/amend
```

Request body:

```json
{
  "amendment_request_id": "amend-abc123",
  "booking_reference": "OSTR-XXXXXXXX",
  "changes": {
    "guests": 3,
    "pets": 1,
    "check_in": "2026-04-10",
    "check_out": "2026-04-15"
  },
  "payment_token": {
    "token": "top-up-token-xyz",
    "amount": 75.00,
    "currency": "USD"
  }
}
```

Notes:
- `changes` contains only the fields being amended, not the full booking
- `payment_token` is omitted for zero-cost amendments, required for top-up amendments
- For repricing amendments where the total decreases, `payment_token` is replaced by a `refund_authorisation` field

Response body mirrors the booking confirmation format with an added `amendment` object:

```json
{
  "booking_reference": "OSTR-XXXXXXXX",
  "amendment_number": 1,
  "amendment": {
    "requested_at": "2026-03-15T14:22:00Z",
    "confirmed_at": "2026-03-15T14:22:01Z",
    "status": "confirmed",
    "changes_applied": {
      "guests": { "from": 2, "to": 3 },
      "pets": { "from": 0, "to": 1 }
    },
    "pricing_adjustment": {
      "original_total": 894.00,
      "amended_total": 969.00,
      "difference": 75.00,
      "currency": "USD",
      "direction": "top_up"
    }
  }
}
```

---

## Related RFC Sections

- `rfc.booking_confirmation.md` — Section 7 (Cancellation) and Section 3 (Booking Request)
- `rfc.availability_query.md` — Section 4 (Pricing Breakdown) — repricing logic applies to amendments
- `rfc.property_listing.md` — `pricing_rules` object — discount tier changes on date amendments

---

## Status

Open for discussion. No SEP has been filed. This document is intended to inform the SEP when v0.2 planning begins.

To contribute to this discussion, open a GitHub issue with the label `SEP` and reference this document.

---

*Document author: Daniel Bloom — based on operational experience with short-term rental platforms.*
*Created: February 2026*
