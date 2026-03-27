# RFC: Property Listing

**RFC ID:** openstr-rfc-001  
**Title:** Property Listing Schema and Discovery  
**Status:** Draft  
**Version:** 0.2.0  
**Created:** February 2026  
**Updated:** March 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Supersedes:** openstr-rfc-001 v0.1.5  

---

## Abstract

This RFC defines the OpenSTR property listing schema — the foundational data structure that enables AI agents to discover, interpret, and reason about short-term rental properties without requiring a centralised platform intermediary. It specifies the required and optional fields for a machine-readable property listing, the discovery mechanism via a well-known endpoint, a three-tier property classification system, location disclosure conventions, the pricing declaration model including static pricing rules and dynamic pricing delegation, minimum stay rules including dynamic gap-fill and seasonal overrides, environment and safety disclosures, and the damage guarantee model based on identity verification tiers.

**Breaking change in v0.2.0:** The well-known discovery path has been restructured. Listing documents are now served under `/.well-known/openstr/{listing_id}.json`. Hosts additionally serve a manifest at `/.well-known/openstr` and a per-listing credential at `/.well-known/openstr/{listing_id}/credential.json`. See Section 4.1.

---

## 1. Motivation and Problem Statement

Short-term rental properties are currently discoverable and bookable only through closed platform intermediaries such as Airbnb, Vrbo, and Booking.com. These platforms provide significant value — aggregating supply and demand, providing trust infrastructure, and standardising the booking experience — but they do so at the cost of host autonomy, high commission rates, and lock-in.

As AI agents become the primary interface through which people search and book accommodation, the absence of an open, machine-readable standard for property listings creates a structural dependency on these intermediaries. An agent wishing to book accommodation on behalf of a user has no standardised way to discover or interpret a property outside of a platform's proprietary API.

OpenSTR addresses this by defining a common vocabulary for property listings that any host can publish and any agent can consume. This RFC defines the property listing schema — the first and most foundational layer of the OpenSTR protocol.

---

## 2. Prior Art

### 2.1 Schema.org LodgingBusiness

[Schema.org](https://schema.org/LodgingBusiness) defines a vocabulary for lodging businesses widely used for SEO and structured data. Schema.org provides a `petsAllowed` boolean field but does not define maximum pet counts, pet fees, property sub-types, environment categories, safety disclosures, or static pricing rules. OpenSTR defines its own native schema for all fields rather than inheriting Schema.org vocabulary directly, to ensure consistency and avoid agents needing to reason across two schemas simultaneously. The optional `schema_org` compatibility field allows hosts to also publish a full Schema.org object for web crawler compatibility.

OpenSTR is architecturally distinct from Schema.org: it defines machine-callable endpoints, not passive HTML metadata. An OpenSTR listing document is a live, queryable resource that an AI agent can act on directly — not a hint to a search crawler. See `docs/protocol-landscape.md` for the full positioning.

### 2.2 Property Classification Conventions

The classification of short-term rental properties by type, sub-type, and occupancy mode is an industry-wide convention that predates any single platform. The OpenSTR property classification vocabulary is derived independently from multiple sources including the OpenTravel Alliance (OTA) accommodation type codes, Schema.org's `LodgingBusiness` and `Accommodation` type hierarchy, common real estate and hospitality industry terminology in wide use across multiple jurisdictions, and the W3C's work on accommodation vocabularies. Property descriptors such as `flat`, `house`, `houseboat`, `cottage`, `villa`, `yurt`, and `treehouse` are ordinary English-language descriptions of real-world property types that exist independently of any platform and are not the intellectual property of any single organisation. The occupancy classification of entire place, private room, and shared room is similarly a long-standing industry convention used across OTA schemas, hotel booking systems, and STR platforms globally. OpenSTR makes no claim that its classification vocabulary is derived from or endorsed by any specific platform.

### 2.3 iCal / CalDAV

The iCalendar standard (RFC 5545) is widely used for availability synchronisation between STR platforms. It addresses calendar state but not property description, pricing, or booking initiation.

### 2.4 OTA Connect / OpenTravel Alliance

The OpenTravel Alliance defines XML-based schemas for travel industry data exchange. These are primarily B2B standards designed for enterprise integrations and are not suitable for lightweight agent-to-host discovery.

### 2.5 Universal Commerce Protocol (UCP)

Google's Universal Commerce Protocol (UCP) defines a standardised agent-to-merchant checkout and order management flow. OpenSTR is architecturally aligned with UCP: both operate within the same agent commerce paradigm. OpenSTR addresses property discovery, listing data, availability state, and booking intent — the layer that precedes UCP's Quote and Order phases. The two protocols are complementary. OpenSTR capabilities are declared under the `org.openstr.*` vendor namespace within the UCP capability model. See `docs/protocol-landscape.md`.

### 2.6 Google's Agent Payments Protocol (AP2)

AP2 (Google) defines agent-to-agent payment flows and includes a travel booking example. Like UCP, it addresses payment execution but not property discovery or description. OpenSTR's `payment_token` field in RFC-003 is designed to be compatible with AP2 mandate token semantics.

### 2.7 Model Context Protocol (MCP) and Agent-to-Agent Protocol (A2A)

MCP (Anthropic) defines a standard for exposing tools and data sources to AI agents. A2A (Google) defines agent-to-agent task delegation. OpenSTR endpoints are intended to be exposed as MCP tools in reference implementations, and the RFC-003 booking confirmation flow is designed to support an A2A transport binding. Neither MCP nor A2A defines property listing or availability schemas — they are transport and capability-discovery layers that OpenSTR builds on top of.

**Gap:** None of the above standards (2.1 through 2.7) define an open, lightweight, agent-readable property listing format with a three-tier property classification, availability querying, dynamic and static pricing rules, dynamic minimum stay rules, environment tagging, safety disclosures, bilateral trust credentials, or IDV-backed damage guarantees. This is the gap OpenSTR addresses.

---

## 3. Design Decisions

### 3.1 OpenSTR-Native Schema

OpenSTR defines all fields natively rather than inheriting from Schema.org. This ensures agents need only understand one schema and avoids conflicts where Schema.org and OpenSTR definitions overlap. Schema.org compatibility is provided via an optional `schema_org` field for web crawler and SEO use cases.

### 3.2 Two-Tier Schema

The property listing schema is divided into two tiers:

- **Required fields** — the minimum a host must publish for a listing to be considered OpenSTR-compliant. These fields are guaranteed to be present and agents can rely on them without defensive handling.
- **Optional fields** — additional fields that enrich the listing for agents capable of using them. Optional fields improve discoverability, filtering accuracy, and agent reasoning but are not required for a valid listing.

### 3.3 Three-Tier Property Classification

Property type is expressed as three distinct fields forming a hierarchy:

- **`category`** — the broadest classification of what kind of place it is (e.g. Flat, House, Unique Space)
- **`sub_type`** — a more specific classification within the category (e.g. Houseboat, Tiny House within Unique Space). Optional.
- **`occupancy_type`** — whether the guest has the entire place, a private room, or a shared room

This mirrors the classification model used by major STR platforms and gives agents structured data to filter on at each level of specificity.

### 3.4 Two-Stage Location Disclosure

Exact property coordinates are not published in the listing. Instead, the listing includes an approximate location sufficient for an agent to reason about location relative to a guest's requirements. Upon booking confirmation, the host system returns a `location_detail` object containing full address, exact coordinates, a what3words reference, and a maps deep-link URI.

### 3.5 Static vs Dynamic Pricing

Not all hosts use third-party dynamic pricing systems. Hosts managing their own rates declare `pricing_mode: static` and provide a `pricing_rules` object defining their discount and promotional structure. This allows agents to pre-filter and estimate total costs accurately without an availability query for static hosts. For `dynamic` hosts, the `pricing_rules` object is indicative only — the pricing system overrides all values at availability query time.

### 3.6 Minimum Stay Rules and Dynamic Overrides

A base minimum stay is declared in the listing for agent pre-filtering. Dynamic hosts commonly apply gap-fill and seasonal rules managed through their pricing system. The listing declares `min_stay_mode` of either `static` or `dynamic`. For `dynamic` listings the authoritative minimum for any date window is returned by the availability endpoint.

### 3.7 Environment Tags

Hosts may declare structured environment tags describing the property's physical setting (lakefront, forest, city centre, etc.) to support agent-side filtering and recommendation. A defined vocabulary is provided to ensure consistency across listings.

### 3.8 Safety Disclosures

Safety disclosures are distinct from amenities. They include both positive safety features (alarms, extinguishers) and important limitations or hazards that an agent must communicate to a guest (not suitable for young children, pool present, security cameras disclosed). These are structured separately from amenities to ensure agents treat them with appropriate prominence.

### 3.9 Access Code Delivery

The listing declares the check-in method as a discovery-time signal. The actual access credential is a post-booking concern defined in `rfc.booking_confirmation.md`. Smart lock API integration is a v0.4 operational tooling concern.

### 3.10 Damage Guarantee and IDV Tiers

OpenSTR replaces the traditional security deposit with a `damage_guarantee` object supporting three modes: `none`, `deposit`, and `idv_guarantee`. For `idv_guarantee`, the host declares a minimum IDV level. Three tiers are defined: `email_verified`, `identity_verified` (document-level, equivalent to Airbnb's passport/ID check), and `payment_verified`.

### 3.11 `listing_id` as Host-Assigned Cross-Protocol Identifier

The `listing_id` is assigned by the host (or their channel manager) and is opaque to the protocol. OpenSTR places no structural constraints on its format. The protocol RECOMMENDS (non-normative) that `listing_id` values use lowercase alphanumeric characters and hyphens for readability, but this is not enforced.

The `listing_id` is the stable, cross-protocol identifier for the property. When a host operates both OpenSTR and UCP endpoints, the `listing_id` is the shared reference that links the two. Agents and channel managers SHOULD treat `listing_id` as the canonical unit of identity for a property across protocols.

To avoid namespace collisions in multi-host or multi-channel-manager deployments, hosts and channel managers SHOULD prefix `listing_id` values with a domain segment or other organisational identifier (e.g. `guesty-abc123-prop-001`, `yourdomain-com-cottage-1`). OpenSTR does not enforce this — it is a deployment convention.

### 3.12 Well-Known Path Namespacing

OpenSTR discovery paths are scoped under `/.well-known/openstr/` in accordance with RFC 8615. This namespacing makes protocol membership unambiguous to crawling agents, avoids collision with other protocols using `/.well-known/`, and mirrors the pattern used by UCP (`/.well-known/ucp`). See Section 4.1.

### 3.13 Per-Listing Credentials

Host credentials are served at a per-listing path rather than a per-host path. This is because the listing is the atomic unit that channel managers and index services reason about — not the host entity. A credential MAY cover multiple listings (same host, same legal entity) but MUST be reachable at the per-listing path. See RFC-004.

### 3.14 UCP Coexistence

OpenSTR and UCP are complementary protocols operating at different layers of the agent commerce stack. A host MAY publish both an OpenSTR listing document and a UCP merchant manifest. The optional `ucp_manifest_url` field in the listing document provides a cross-reference for agents that support both protocols. See Section 4.2.2 and `docs/protocol-landscape.md`.

---

## 4. Specification

### 4.1 Discovery Mechanism

#### 4.1.1 Well-Known Paths

OpenSTR discovery is served under the `/.well-known/openstr/` namespace. Three distinct paths are defined:

**Host manifest:**
```
GET /.well-known/openstr
```
Returns an `OpenSTRManifest` object listing all OpenSTR listing IDs published by the host at this domain. See Section 4.1.2.

**Listing document:**
```
GET /.well-known/openstr/{listing_id}.json
```
Returns a single `OpenSTRListing` object for the specified `listing_id`. This is the primary document consumed by AI agents and index crawlers.

**Listing credential:**
```
GET /.well-known/openstr/{listing_id}/credential.json
```
Returns the `HostCredential` object for the specified listing. See RFC-004.

All endpoints must be served over HTTPS with `Content-Type: application/json` and an `ETag` header.

#### 4.1.2 Host Manifest Object

The manifest at `/.well-known/openstr` declares the host's OpenSTR presence at this domain and enables agents to discover all listings without prior knowledge of individual `listing_id` values.

| Field | Type | Required | Description |
|---|---|---|---|
| `openstr_version` | string | Yes | Protocol version. Must be `"0.2"` for this spec. |
| `host_id` | string | No | Host identifier, if registered with an index. |
| `listings` | array[object] | Yes | Array of listing summary objects. See below. |
| `updated_at` | string (ISO 8601) | Yes | Timestamp of last manifest update. |

Each item in the `listings` array:

| Field | Type | Required | Description |
|---|---|---|---|
| `listing_id` | string | Yes | The listing identifier. |
| `listing_name` | string | Yes | Human-readable listing name. |
| `listing_endpoint` | string (URI) | Yes | Full URL of the listing document (`/.well-known/openstr/{listing_id}.json`). |
| `availability_endpoint` | string (URI) | Yes | Full URL of the availability endpoint. |
| `booking_endpoint` | string (URI) | Yes | Full URL of the booking endpoint. |
| `credential_endpoint` | string (URI) | Yes | Full URL of the credential document (`/.well-known/openstr/{listing_id}/credential.json`). |

**Example manifest:**

```json
{
  "openstr_version": "0.2",
  "host_id": "host-0ut0etlrxbks",
  "listings": [
    {
      "listing_id": "walled-garden-hut-snettisham-001",
      "listing_name": "Walled Garden Hut — Pet-Friendly Shepherd's Hut, Norfolk",
      "listing_endpoint": "https://availability.openstr.org/.well-known/openstr/walled-garden-hut-snettisham-001.json",
      "availability_endpoint": "https://availability.openstr.org/availability/walled-garden-hut-snettisham-001",
      "booking_endpoint": "https://availability.openstr.org/booking/walled-garden-hut-snettisham-001",
      "credential_endpoint": "https://availability.openstr.org/.well-known/openstr/walled-garden-hut-snettisham-001/credential.json"
    }
  ],
  "updated_at": "2026-03-27T00:00:00Z"
}
```

#### 4.1.3 Legacy Path Retirement

The following paths defined in v0.1.x are retired in v0.2.0:

| Legacy path | Replacement |
|---|---|
| `/.well-known/openstr.json` | `/.well-known/openstr` (manifest) |
| `/.well-known/{listing_id}.json` | `/.well-known/openstr/{listing_id}.json` |
| `/.well-known/host-credential.json` | `/.well-known/openstr/{listing_id}/credential.json` |

Implementations upgrading from v0.1.x SHOULD return HTTP 301 redirects from legacy paths to new paths during a transition period, but are not required to.

---

### 4.2 OpenSTRListing Object

#### 4.2.1 Required Fields

| Field | Type | Description |
|---|---|---|
| `openstr_version` | string | Protocol version. Must be `"0.2"` for this spec. |
| `listing_id` | string | Host-assigned unique identifier. Stable across updates. Opaque to the protocol — see Section 3.11. |
| `listing_name` | string | Property name or title. Max 100 characters. |
| `property_classification` | object | Three-tier property classification. See Section 4.3. |
| `description` | string | Plain text description. Max 2000 characters. |
| `location` | object | Approximate location object. See Section 4.4. |
| `capacity` | object | Guest capacity object. See Section 4.5. |
| `bedrooms` | integer | Number of bedrooms. Studio properties use `0`. |
| `bathrooms` | number | Number of bathrooms. Accepts `.5` for half-bathrooms. |
| `pricing` | object | Pricing declaration object. See Section 4.6. |
| `min_stay` | object | Minimum stay declaration object. See Section 4.7. |
| `availability_endpoint` | string (URI) | URL of the availability query endpoint. |
| `booking_endpoint` | string (URI) | URL of the booking request endpoint. |
| `host_credential` | object | Host credential reference. See Section 4.2.3 and RFC-004. |
| `policies` | object | Core policies object. See Section 4.8. |
| `safety_disclosures` | object | Safety disclosures object. See Section 4.9. |
| `updated_at` | string (ISO 8601) | Timestamp of last listing update. |

**Note:** `listing_url` is optional in v0.2.0 (was required in v0.1.x). See Section 4.2.2.

#### 4.2.2 Optional Fields

| Field | Type | Description |
|---|---|---|
| `listing_url` | string (URI) | Canonical URL of a human-readable listing page for the property. No constraints on host or platform. Agents must not treat this as a machine-callable endpoint — see guideline 6.14. |
| `ucp_manifest_url` | string (URI) | URL of the host's UCP merchant manifest, if the host also publishes a UCP endpoint. Enables agents that support both protocols to link the OpenSTR listing to the UCP checkout flow. See Section 3.14. |
| `amenities` | array[string] | Amenity vocabulary. See Section 4.10. |
| `environment_tags` | array[string] | Environment vocabulary. See Section 4.11. |
| `images` | array[object] | Image references. See Section 4.12. |
| `accessibility` | object | Accessibility features. See Section 4.13. |
| `year_built` | integer | Year the property was built. |
| `property_size` | object | `{ "value": number, "unit": "sqm" or "sqft" }` |
| `neighbourhood_description` | string | Description of the surrounding area. Max 500 characters. |
| `transit` | string | Description of nearby public transport. Max 300 characters. |
| `languages_spoken` | array[string] | ISO 639-1 language codes. |
| `schema_org` | object | Full Schema.org `LodgingBusiness` object for web crawler compatibility. |
| `tags` | array[string] | Freeform host-assigned tags. Max 20 tags, 30 characters each. |

#### 4.2.3 Host Credential Reference Object

The `host_credential` field in the listing document is a reference to the full credential document served at `/.well-known/openstr/{listing_id}/credential.json`. It contains enough information for an agent to locate and fetch the credential without a prior manifest lookup.

| Field | Type | Required | Description |
|---|---|---|---|
| `credential_uri` | string (URI) | Yes | Full URL of the credential document at `/.well-known/openstr/{listing_id}/credential.json`. |
| `issuer` | string | Yes | URI of the credential issuer. |
| `issued_at` | string (ISO 8601) | Yes | Timestamp when the credential was issued. |

---

### 4.3 Property Classification Object

All three fields are required within this object.

| Field | Type | Description |
|---|---|---|
| `category` | enum | Primary property category. See Section 4.3.1. |
| `sub_type` | enum | More specific classification within category. See Section 4.3.2. Optional — omit if no applicable sub-type. |
| `occupancy_type` | enum | `entire_place`, `private_room`, or `shared_room` |

#### 4.3.1 Category Vocabulary

| Value | Description |
|---|---|
| `flat` | Apartment or flat |
| `house` | Detached, semi-detached, or terraced house |
| `secondary_unit` | Guest suite, garden flat, or annexe within or adjacent to a host's primary residence |
| `unique_space` | A distinctive or unconventional property type. Use with `sub_type`. |
| `bed_and_breakfast` | B&B with host present and breakfast included |
| `boutique_hotel` | Small hotel or guest house with distinct rooms |

#### 4.3.2 Sub-Type Vocabulary

Sub-types are only meaningful for certain categories. Agents should treat unknown sub-type values gracefully.

| Value | Typical category | Description |
|---|---|---|
| `studio` | `flat` | Single open-plan room with no separate bedroom |
| `penthouse` | `flat` | Top-floor apartment |
| `cottage` | `house` | Small, typically rural dwelling |
| `townhouse` | `house` | Multi-storey terraced or semi-detached house |
| `villa` | `house` | Larger detached property, typically with outdoor space |
| `cabin` | `unique_space` | Wooden cabin or lodge |
| `treehouse` | `unique_space` | Elevated structure built in or around a tree |
| `tiny_home` | `unique_space` | Compact purpose-built small home |
| `houseboat` | `unique_space` | Permanently or semi-permanently moored vessel |
| `narrowboat` | `unique_space` | Canal narrowboat |
| `shepherds_hut` | `unique_space` | Traditional or contemporary shepherd's hut |
| `yurt` | `unique_space` | Circular tent structure |
| `tipi` | `unique_space` | Conical tent structure |
| `dome` | `unique_space` | Geodesic or inflatable dome structure |
| `glamping_pod` | `unique_space` | Insulated outdoor sleeping pod |
| `tent` | `unique_space` | Permanent or semi-permanent tent structure |
| `barn` | `unique_space` | Agricultural building converted to residential use |
| `farm_stay` | `unique_space` | Accommodation on a working farm |
| `boat` | `unique_space` | Vessel not permanently moored |
| `camper_van` | `unique_space` | Converted camper van or motorhome |
| `bus` | `unique_space` | Converted bus or double-decker |
| `plane` | `unique_space` | Converted aircraft |
| `train` | `unique_space` | Converted railway carriage |
| `castle` | `unique_space` | Historic castle or fortified building |
| `tower` | `unique_space` | Converted tower or turret |
| `lighthouse` | `unique_space` | Converted lighthouse |
| `windmill` | `unique_space` | Converted windmill |
| `cave` | `unique_space` | Cave dwelling or underground structure |
| `earthen_home` | `unique_space` | Earth-sheltered or hobbit-style dwelling |
| `hut` | `unique_space` | Simple hut or bothy |
| `riad` | `unique_space` | North African courtyard house |
| `ranch` | `unique_space` | Ranch or working estate accommodation |
| `religious_building` | `unique_space` | Converted chapel, church, or other religious building |
| `shipping_container` | `unique_space` | Converted shipping container |
| `campsite` | `unique_space` | Designated campsite pitch or area |
| `guest_suite` | `secondary_unit` | Self-contained suite attached to host's property |
| `garden_flat` | `secondary_unit` | Ground-floor or basement flat with garden access |
| `other` | any | Distinctive property not covered by the above vocabulary |

---

### 4.4 Location Object

#### Required location fields

| Field | Type | Description |
|---|---|---|
| `city` | string | City or town name |
| `region` | string | Region, county, or state |
| `country` | string | ISO 3166-1 alpha-2 country code |
| `approximate_lat` | number | Approximate latitude, rounded to 2 decimal places |
| `approximate_lng` | number | Approximate longitude, rounded to 2 decimal places |

#### Optional location fields

| Field | Type | Description |
|---|---|---|
| `postcode_area` | string | Partial postcode or zip code area (e.g. `SW1`, `60614`) |
| `neighbourhood` | string | Neighbourhood or district name |
| `landmarks` | array[string] | Nearby landmarks for agent-side reasoning. Max 5 items. |
| `timezone` | string | IANA timezone identifier (e.g. `Europe/London`) |

---

### 4.5 Capacity Object

| Field | Type | Required | Description |
|---|---|---|---|
| `guests` | integer | Yes | Maximum number of guests. |
| `beds` | integer | Yes | Number of beds. |
| `bed_types` | array[string] | No | Bed type vocabulary: `single`, `double`, `queen`, `king`, `bunk`, `sofa_bed`, `floor_mattress`. |

**Note:** `guests_max` and `guests_recommended` used in some v0.1.x examples are replaced by the single `guests` field (maximum) in v0.2.0.

---

### 4.6 Pricing Object

#### Required pricing fields

| Field | Type | Description |
|---|---|---|
| `currency` | string | ISO 4217 currency code |
| `nightly_rate_indicative` | number | Base or floor nightly rate. For `dynamic` listings this is the floor rate only. |
| `cleaning_fee` | number | Fixed cleaning fee per stay. `0` if none. |
| `pricing_mode` | enum | `static` or `dynamic` |

#### Optional pricing fields

| Field | Type | Description |
|---|---|---|
| `pricing_rules` | object | Discount and promotional rules. See Section 4.6.1. For `static` hosts this is authoritative; for `dynamic` hosts indicative only. |
| `pricing_provider` | string | Name of external dynamic pricing provider. Informational only. |
| `pricing_endpoint` | string (URI) | URI of the external pricing API. Informational only. |
| `extra_guest_fee` | object | `{ "above": integer, "fee_per_night": number }` |

#### 4.6.1 Pricing Rules Object

Declares the host's discount and promotional structure. For `static` hosts, agents may use these rules to pre-calculate estimated totals. For `dynamic` hosts, the pricing system overrides these values at availability query time.

| Field | Type | Required | Description |
|---|---|---|---|
| `weekly_discount_pct` | number | No | Percentage discount for stays of 7+ nights |
| `monthly_discount_pct` | number | No | Percentage discount for stays of 28+ nights |
| `last_minute_discount` | object | No | `{ "within_days": integer, "discount_pct": number }` — discount applied when check-in is within N days |
| `trip_length_discounts` | array[object] | No | Length-of-stay discounts: `[{ "min_nights": integer, "discount_pct": number }]` — evaluated in order, first match applies |
| `promotions` | array[object] | No | Time-limited promotional rates. See Section 4.6.2. |

#### 4.6.2 Promotion Object

| Field | Type | Required | Description |
|---|---|---|---|
| `promotion_id` | string | Yes | Host-assigned identifier for this promotion |
| `description` | string | Yes | Human-readable description. Max 100 characters. |
| `discount_pct` | number | Yes | Percentage discount |
| `valid_from` | string (ISO 8601 date) | Yes | First date the promotion applies |
| `valid_to` | string (ISO 8601 date) | Yes | Last date the promotion applies |
| `min_stay_nights` | integer | No | Minimum stay required to qualify for this promotion |

---

### 4.7 Minimum Stay Object

| Field | Type | Required | Description |
|---|---|---|---|
| `min_stay_nights` | integer | Yes | Default minimum stay in nights. For `dynamic` listings indicative only. |
| `min_stay_mode` | enum | Yes | `static` or `dynamic`. If `dynamic`, authoritative minimum stay for any date window is returned by the availability endpoint. |
| `max_stay_nights` | integer | No | Maximum stay in nights, if applicable. |

**Note:** Dynamic minimum stay rules — including gap-fill rules (reduced minimum to fill gaps between bookings) and seasonal rules (e.g. reduced minimum in low season) — are managed by the host's pricing system and resolved per date window in the availability response.

---

### 4.8 Policies Object

#### Required policy fields

| Field | Type | Description |
|---|---|---|
| `cancellation_policy` | enum | `flexible`, `moderate`, `firm`, `strict` |
| `check_in_time_from` | string | Earliest check-in time in `HH:MM` 24-hour format |
| `check_in_time_to` | string | Latest check-in time in `HH:MM` 24-hour format. Omit if no upper limit. |
| `check_out_time` | string | Latest check-out time in `HH:MM` 24-hour format |
| `pets` | object | Pet policy object. See Section 4.8.1. |
| `smoking_allowed` | boolean | Whether smoking, vaping or e-cigarettes are permitted |
| `events_allowed` | boolean | Whether events or parties are permitted |
| `instant_book` | boolean | Whether bookings are confirmed instantly or require host approval |
| `damage_guarantee` | object | Damage guarantee object. See Section 4.8.2. |
| `check_in_method` | enum | `in_person`, `self_checkin_keybox`, `self_checkin_smartlock`, `self_checkin_other` |

#### Optional policy fields

| Field | Type | Description |
|---|---|---|
| `quiet_hours_start` | string | Start of quiet hours in `HH:MM` format |
| `quiet_hours_end` | string | End of quiet hours in `HH:MM` format |
| `age_restrictions` | array[object] | Age suitability restrictions. See Section 4.8.3. |
| `good_track_record_required` | boolean | Host requires guests with a verified positive booking history. Maps to GuestCredential `booking_history` claim in RFC-004. |
| `ev_charging_permitted` | boolean | Whether electric vehicle charging via property sockets is permitted. Omission implies no stated restriction. |
| `availability_window_months` | integer | How many months in advance the property is bookable. |
| `advance_notice` | object | Advance notice requirements. See Section 4.8.4. |
| `house_rules` | array[string] | Structured list of house rules. Max 20 items, 200 characters each. |

#### 4.8.1 Pet Policy Object

| Field | Type | Required | Description |
|---|---|---|---|
| `allowed` | boolean | Yes | Whether pets are permitted |
| `pets_max` | integer | No | Maximum number of pets. Omission implies no stated limit if `allowed` is `true`. |
| `pet_fee` | object | No | `{ "amount": number, "currency": string, "per": "per_stay" or "per_night" }` |
| `pet_restrictions` | array[string] | No | Structured list of pet restrictions. Max 10 items, 200 characters each. |
| `host_pets_on_property` | boolean | No | Whether the host's own pets live at or visit the property. Guests may encounter host pets during their stay. |

#### 4.8.2 Damage Guarantee Object

| Field | Type | Required | Description |
|---|---|---|---|
| `mode` | enum | Yes | `none`, `deposit`, or `idv_guarantee` |
| `deposit_amount` | number | If `mode` is `deposit` | Cash deposit amount held at booking |
| `deposit_currency` | string | If `mode` is `deposit` | ISO 4217 currency code for deposit |
| `idv_level` | enum | If `mode` is `idv_guarantee` | Minimum IDV level required |
| `idv_accepted_issuers` | array[string] | No | Accepted credential issuer URIs. If omitted, any recognised issuer accepted. |

**IDV Level Vocabulary:**

| Value | Description |
|---|---|
| `email_verified` | Email address verified by a recognised credential issuer |
| `identity_verified` | Identity verified against a government-issued document (passport, national ID, or driving licence) by a recognised IDV provider |
| `payment_verified` | Payment method verified with financial guarantee in place via the credential issuer |

#### 4.8.3 Age Restrictions Array

An array of objects describing age-based suitability restrictions. Each object has the following fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `not_suitable_under_age` | integer | Yes | Property is not suitable for guests or accompanying children under this age |
| `reason` | string | Yes | Freeform explanation for the restriction. Max 300 characters. |

**Example:**

```json
"age_restrictions": [
  {
    "not_suitable_under_age": 12,
    "reason": "High entrance step — not suitable for guests with mobility issues or children under 12. Not suitable for infants or toddlers."
  }
]
```

Multiple restrictions may be declared if different features give rise to different age thresholds.

**Agent behaviour:** Agents must surface age restrictions prominently when showing a listing to a user, and must prompt the user to confirm whether any member of their party falls below a stated age threshold before proceeding to booking.

#### 4.8.4 Advance Notice Object

Declares the host's requirements for notice between booking and arrival.

| Field | Type | Required | Description |
|---|---|---|---|
| `min_days_notice` | integer | Yes | Minimum number of days notice required between booking and arrival. `0` means same-day bookings considered. |
| `allow_short_notice_requests` | boolean | No | Whether the host will consider requests with less notice than `min_days_notice`. If `true`, agents should indicate that short-notice requests may be accepted at host discretion. |

---

### 4.9 Safety Disclosures Object

Safety disclosures are distinct from amenities. They communicate both protective features and important hazards or limitations. Agents must present safety disclosures prominently when showing a listing to a user.

All fields are required within this object. Where a condition does not apply, the field should be set to `false` or `null` as appropriate.

#### Required safety fields

| Field | Type | Description |
|---|---|---|
| `smoke_detector` | boolean | Smoke detector present and functional |
| `carbon_monoxide_detector` | boolean | Carbon monoxide detector present and functional |
| `fire_extinguisher` | boolean | Fire extinguisher present |
| `first_aid_kit` | boolean | First aid kit present |

#### Optional safety fields

| Field | Type | Description |
|---|---|---|
| `pool_present` | boolean | Pool present on property — relevant for families with young children |
| `hot_tub_present` | boolean | Hot tub present |
| `security_cameras_exterior` | boolean | Security cameras present on exterior of property. Hosts must disclose this. |
| `security_cameras_interior` | boolean | Security cameras present inside property. Interior cameras are prohibited in bedrooms and bathrooms. Must be disclosed if present in common areas. |
| `weapons_on_premises` | boolean | Weapons stored on the property. Must be disclosed. |
| `climbing_hazards` | boolean | Property has unfenced drops, staircases without guards, or other climbing hazards |
| `noise_disclaimer` | string | Freeform description of known noise sources. Max 300 characters. |
| `safety_notes` | string | Any additional safety information the host wishes to disclose. Max 500 characters. |

**Note:** Age-based suitability restrictions (e.g. not suitable for children under 12) are declared in `policies.age_restrictions` rather than in `safety_disclosures`, to allow agents to reason about them structurally. Host pet disclosures are declared in `policies.pets.host_pets_on_property`. Agents must treat both as safety-relevant and surface them prominently alongside `safety_disclosures`.

---

### 4.10 Amenity Vocabulary

Standardised values for the `amenities` array. Prefix freeform amenities with `custom:`.

**Connectivity:** `wifi`, `wifi_fast` (>100Mbps), `ethernet`

**Kitchen:** `kitchen_full`, `kitchen_kitchenette`, `dishwasher`, `washing_machine`, `dryer`, `coffee_maker`, `kettle`, `toaster`, `oven`, `cooker`, `microwave`, `mini_fridge`, `fridge`, `freezer`, `cooking_basics`, `crockery_and_cutlery`, `wine_glasses`, `dining_table`

**Climate:** `air_conditioning`, `heating_central`, `heating_portable`, `indoor_fireplace`

**Outdoor:** `garden_private`, `garden_shared`, `balcony`, `patio`, `terrace`, `outdoor_dining_area`, `outdoor_furniture`, `sun_loungers`, `parking_private`, `parking_street`, `private_entrance`

**Safety:** `smoke_detector`, `carbon_monoxide_detector`, `fire_extinguisher`, `first_aid_kit` (also declare these in `safety_disclosures`)

**Accessibility:** `lift`, `step_free_access`, `wide_doorways`

**Entertainment:** `tv`, `tv_streaming`, `board_games`, `books`, `games_console`

**Work:** `dedicated_workspace`, `monitor_external`

**Bedroom and bathroom:** `bed_linen`, `towels`, `room_darkening_blinds`, `clothes_storage`, `hair_dryer`, `body_soap`, `shampoo`, `conditioner`, `shower_gel`, `hot_water`, `cleaning_products`

---

### 4.11 Environment Tags Vocabulary

Standardised values for the `environment_tags` array. Describes the property's physical setting for agent-side filtering.

| Value | Description |
|---|---|
| `city_centre` | Located in or immediately adjacent to a city centre |
| `urban` | Urban residential area |
| `suburban` | Suburban residential area |
| `rural` | Rural or semi-rural setting |
| `countryside` | Open countryside or agricultural landscape |
| `coastal` | Near the coast or seafront |
| `beachfront` | Direct beach access |
| `lakefront` | Direct lake access |
| `riverside` | Adjacent to a river |
| `mountain` | Mountain or highland setting |
| `forest` | Woodland or forest setting |
| `ski_in_ski_out` | Direct access to ski slopes |
| `desert` | Desert setting |
| `historic_district` | Located within a designated historic area |
| `historic` | Property or grounds of historic character or significance |
| `estate` | Located within a private estate or country estate grounds |
| `farm` | Located on or adjacent to a working farm |
| `nature_reserve` | Adjacent to or within a nature reserve or protected area |
| `village` | Located in a village or small settlement |

---

### 4.12 Image Object

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | string (URI) | Yes | HTTPS URL of the image |
| `caption` | string | No | Human-readable caption |
| `room_type` | enum | No | `bedroom`, `bathroom`, `kitchen`, `living_room`, `outdoor`, `entrance`, `other` |
| `width_px` | integer | No | Image width in pixels |
| `height_px` | integer | No | Image height in pixels |

---

### 4.13 Accessibility Object

All fields optional. Omission implies status unknown, not absent.

| Field | Type | Description |
|---|---|---|
| `step_free_access` | boolean | Step-free access from street to entrance |
| `lift_available` | boolean | Lift available if above ground floor |
| `wide_doorways` | boolean | Doorways of at least 80cm width throughout |
| `wet_room` | boolean | Step-free shower or wet room |
| `ground_floor` | boolean | Property entirely on ground floor |
| `flights_to_listing` | integer | Number of stair flights between street level and the listing entrance. `0` means street level or lift access. Convention-agnostic alternative to floor numbering. |
| `floors_in_building` | integer | Total number of floors in the building. |
| `accessibility_notes` | string | Freeform description. Max 500 characters. |

---

## 5. Example

A complete example of a valid OpenSTR v0.2 property listing. The host uses PriceLabs for dynamic pricing and minimum stay management, requires identity-verified guests, and operates a smart lock entry system.

```json
{
  "openstr_version": "0.2",
  "listing_id": "yourdomain-com-lincoln-park-001",
  "listing_name": "Quiet Garden Apartment, Lincoln Park",
  "listing_url": "https://yourdomain.com/properties/lincoln-park-apartment",
  "ucp_manifest_url": "https://yourdomain.com/.well-known/ucp",
  "property_classification": {
    "category": "flat",
    "sub_type": "garden_flat",
    "occupancy_type": "entire_place"
  },
  "description": "A peaceful two-bedroom garden apartment on a quiet residential street in Lincoln Park, Chicago. Five minutes walk from Fullerton Avenue CTA station. Private garden, fully equipped kitchen, dedicated workspace.",
  "location": {
    "city": "Chicago",
    "region": "Illinois",
    "country": "US",
    "postcode_area": "60614",
    "approximate_lat": 41.92,
    "approximate_lng": -87.64,
    "neighbourhood": "Lincoln Park",
    "landmarks": [
      "Lincoln Park Zoo",
      "North Avenue Beach",
      "Fullerton Avenue CTA station"
    ]
  },
  "capacity": {
    "guests": 4,
    "beds": 2,
    "bed_types": ["double", "double"]
  },
  "bedrooms": 2,
  "bathrooms": 1,
  "pricing": {
    "currency": "USD",
    "nightly_rate_indicative": 150,
    "cleaning_fee": 75,
    "pricing_mode": "dynamic",
    "pricing_provider": "PriceLabs",
    "pricing_rules": {
      "weekly_discount_pct": 10,
      "monthly_discount_pct": 20,
      "last_minute_discount": {
        "within_days": 3,
        "discount_pct": 15
      }
    }
  },
  "min_stay": {
    "min_stay_nights": 3,
    "min_stay_mode": "dynamic",
    "max_stay_nights": 90
  },
  "availability_endpoint": "https://yourdomain.com/openstr/availability",
  "booking_endpoint": "https://yourdomain.com/openstr/booking",
  "host_credential": {
    "credential_uri": "https://yourdomain.com/.well-known/openstr/yourdomain-com-lincoln-park-001/credential.json",
    "issuer": "https://credentials.example-issuer.com",
    "issued_at": "2026-01-15T00:00:00Z"
  },
  "policies": {
    "cancellation_policy": "moderate",
    "check_in_time_from": "15:00",
    "check_in_time_to": "21:00",
    "check_out_time": "11:00",
    "pets": {
      "allowed": true,
      "pets_max": 1,
      "pet_fee": {
        "amount": 50,
        "currency": "USD",
        "per": "per_stay"
      },
      "pet_restrictions": [
        "Small dogs and cats only.",
        "Pets must not be left unattended."
      ],
      "host_pets_on_property": false
    },
    "smoking_allowed": false,
    "events_allowed": false,
    "instant_book": false,
    "damage_guarantee": {
      "mode": "idv_guarantee",
      "idv_level": "identity_verified"
    },
    "check_in_method": "self_checkin_smartlock",
    "quiet_hours_start": "22:00",
    "quiet_hours_end": "08:00",
    "good_track_record_required": true,
    "ev_charging_permitted": false,
    "availability_window_months": 12,
    "advance_notice": {
      "min_days_notice": 1,
      "allow_short_notice_requests": false
    },
    "house_rules": [
      "No smoking, vaping or e-cigarettes.",
      "No parties or events.",
      "Quiet hours 22:00–08:00.",
      "Maximum 4 guests."
    ]
  },
  "safety_disclosures": {
    "smoke_detector": true,
    "carbon_monoxide_detector": true,
    "fire_extinguisher": true,
    "first_aid_kit": true,
    "security_cameras_exterior": false,
    "security_cameras_interior": false
  },
  "amenities": [
    "wifi_fast",
    "kitchen_full",
    "washing_machine",
    "dryer",
    "garden_private",
    "dedicated_workspace",
    "heating_central",
    "tv_streaming",
    "bed_linen",
    "towels",
    "hair_dryer"
  ],
  "environment_tags": ["urban", "city_centre"],
  "year_built": 1932,
  "property_size": {
    "value": 85,
    "unit": "sqm"
  },
  "accessibility": {
    "step_free_access": true,
    "ground_floor": true,
    "flights_to_listing": 0,
    "floors_in_building": 1,
    "accessibility_notes": "Ground floor apartment with step-free access from street."
  },
  "languages_spoken": ["en"],
  "updated_at": "2026-03-27T09:00:00Z"
}
```

---

## 6. Agent Behaviour Guidelines

**6.1 Pricing:** For `dynamic` listings, `nightly_rate_indicative` is a floor estimate only. Agents must fetch authoritative pricing via the availability endpoint before presenting a confirmed price.

**6.2 Minimum stay:** For `dynamic` listings, `min_stay_nights` is indicative only. The authoritative minimum for a specific window is returned by the availability endpoint.

**6.3 Location:** Exact address is released only in the booking confirmation `location_detail` object. Agents must not attempt to derive exact addresses from approximate coordinates.

**6.4 Host credential:** Agents should verify the `host_credential` before presenting a listing as bookable. The credential is fetched from `host_credential.credential_uri`. Listings with unverifiable or expired credentials should be flagged.

**6.5 IDV requirements:** Agents must check `damage_guarantee.idv_level` before initiating a booking and confirm the guest's credential satisfies it.

**6.6 Safety disclosures:** Agents must present `safety_disclosures` prominently when showing a listing to a user, particularly `security_cameras_exterior`, `security_cameras_interior`, and `weapons_on_premises`.

**6.7 Pricing rules for static hosts:** For `static` listings, agents may use `pricing_rules` to pre-calculate estimated totals for agent-side filtering and pre-trip cost estimates. The availability endpoint remains the authoritative source before booking.

**6.8 Unknown fields:** Agents must ignore unknown fields to ensure forward compatibility.

**6.9 Age restrictions:** Agents must surface `policies.age_restrictions` prominently alongside safety disclosures. Before proceeding to booking, agents must prompt the user to confirm whether any member of their party falls below a stated age threshold. A booking must not be initiated if the guest confirms a party member is below a stated threshold, unless the host explicitly overrides this via a booking request response.

**6.10 Check-in window:** Where `check_in_time_from` and `check_in_time_to` are both present, agents must communicate the full window to guests rather than only the earliest time.

**6.11 Advance notice:** Where `policies.advance_notice` is present, agents must not present same-day or near-term dates as bookable without checking. If `allow_short_notice_requests` is `true`, agents may indicate that short-notice requests may be considered at host discretion but must not confirm availability without an availability query.

**6.12 Good track record requirement:** Where `policies.good_track_record_required` is `true`, agents must check that the guest holds a GuestCredential with a valid `booking_history` claim before initiating a booking. See RFC-004.

**6.13 Host pets:** Where `policies.pets.host_pets_on_property` is `true`, agents must disclose this to the guest before booking, particularly where the guest has stated allergies or preferences.

**6.14 `listing_url`:** Where `listing_url` is present, agents may surface it as a human-readable reference for the user. Agents must not treat `listing_url` as an endpoint — it is a human-readable page, not a machine-callable resource.

**6.15 `ucp_manifest_url`:** Where `ucp_manifest_url` is present and the agent supports UCP, the agent MAY use it to initiate a UCP checkout flow after booking intent is confirmed via RFC-003. The presence of `ucp_manifest_url` does not change OpenSTR's booking flow — RFC-003 remains the authoritative booking initiation mechanism.

**6.16 Manifest discovery:** Agents discovering a new host domain SHOULD first request `/.well-known/openstr` to enumerate all available listings before fetching individual listing documents.

---

## 7. Security Considerations

**7.1 HTTPS requirement:** All endpoints must be served over HTTPS.

**7.2 Host credential validation:** Agents must independently fetch and validate the `host_credential` document from the `credential_uri` path.

**7.3 URL injection:** Agents must validate that endpoint URIs are well-formed HTTPS URLs.

**7.4 Approximate location:** Hosts may apply a random offset of up to 0.01 degrees to prevent triangulation. This offset must not exceed 0.02 degrees.

**7.5 IDV credential trust:** A curated list of recognised OpenSTR IDV issuers will be maintained in `docs/trusted-issuers.md`. Candidate issuers include Google Identity, Apple, Stripe Identity, Onfido, and Yoti.

**7.6 Security camera disclosure:** Hosts must disclose all security cameras. Interior cameras in bedrooms or bathrooms are prohibited. Agents must surface camera disclosures to users before booking confirmation.

**7.7 `listing_id` namespace collisions:** In multi-host or multi-channel-manager deployments, hosts SHOULD prefix `listing_id` values to avoid collisions. See Section 3.11.

---

## 8. Open Questions

**8.1 Directory / aggregation:** Multi-host directory schema deferred to a future SEP.

**8.2 Multi-unit properties:** Parent/child listing pattern for multi-unit properties deferred.

**8.3 Cancellation policy terms:** Precise refund terms for each tier not standardised in v0.1.

**8.4 Recognised IDV issuers:** Community-governed trusted issuer registry proposed for v1.0.

**8.5 Review and reputation:** Post-stay reviews anchored to confirmed booking references defined in v0.2. Reviews will only be valid if cryptographically anchored to a confirmed OpenSTR booking reference.

**8.6 Smart lock integration:** Automatic time-limited access code generation via smart lock APIs is a v0.4 concern.

**8.7 Amenity and sub-type vocabulary completeness:** The sub-type vocabulary was significantly expanded in v0.1.4 using cross-industry property classification references. Further additions welcome via the SEP process.

**8.8 Promotional pricing verification:** No mechanism currently exists for an agent to verify that a declared promotion is genuine. A promotion verification extension is deferred to a future SEP.

**8.9 EV charging:** The `ev_charging_permitted` field covers the common restriction case. A future SEP may define a structured EV charging capability field (charge speed, connector type, fee) for properties that actively offer EV charging as an amenity.

**8.10 Post-payment calendar blocking:** When a payment is completed via an external payment rail (UCP, Stripe, or equivalent), the mechanism for triggering calendar blocking on the availability endpoint is not yet defined. Options include a UCP Order Management webhook, an iCal feed update via the channel manager, or a dedicated OpenSTR booking confirmation callback endpoint (RFC-003 extension). This is acknowledged as a v0.1 limitation; resolution deferred to v0.2. See RFC-003 Section 8.

**8.11 `ucp_manifest_url` cross-reference standardisation:** The `ucp_manifest_url` field enables coexistence but does not define a formal binding between an OpenSTR booking confirmation and a UCP order. A formal cross-protocol glue specification (linking RFC-003 booking references to UCP order IDs) is deferred to a future RFC.

---

## 9. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |
| 0.1.1-draft | February 2026 | `name` → `listing_name`; pets expanded to object; `min_stay` expanded to object with `min_stay_mode`; two-stage location updated; `security_deposit` replaced with `damage_guarantee` with IDV tiers; Schema.org rationale clarified |
| 0.1.2-draft | February 2026 | `property_type` replaced with three-tier `property_classification` object; `pricing_rules` object added for static host discount and promotional structure; `safety_disclosures` object added as required field; `environment_tags` vocabulary added; `year_built` and `property_size` added as optional fields; sub-type vocabulary expanded |
| 0.1.3-draft | February 2026 | Section 2.2 added — Property Classification Conventions — documenting independent derivation of classification vocabulary from OTA, Schema.org, and industry-standard terminology |
| 0.1.4-draft | March 2026 | Sub-type vocabulary expanded to 37 values; `check_in_time` split into window; `policies.age_restrictions` array added; `policies.advance_notice` added; `policies.good_track_record_required` added; `policies.ev_charging_permitted` added; `policies.availability_window_months` added; `policies.house_rules` promoted to structured array; `pets.pet_restrictions` array added; `pets.host_pets_on_property` added; amenity and environment vocabulary expanded; cancellation policy vocabulary corrected |
| 0.1.5-draft | March 2026 | `accessibility` object: `flights_to_listing` and `floors_in_building` added; `capacity.guests_max` → `capacity.guests` rename |
| 0.2.0-draft | March 2026 | **Breaking change.** Well-known paths restructured: listing documents moved to `/.well-known/openstr/{listing_id}.json`; host manifest added at `/.well-known/openstr`; per-listing credential path defined at `/.well-known/openstr/{listing_id}/credential.json`. Legacy paths `/.well-known/openstr.json`, `/.well-known/{listing_id}.json`, and `/.well-known/host-credential.json` retired. `listing_url` demoted from required to optional with SHOULD NOT guidance against third-party platform URLs. `ucp_manifest_url` optional field added for UCP coexistence. `listing_id` opaqueness and cross-protocol UID role formally documented (Section 3.11). `listing_id` namespace collision guidance added (Section 3.11, Section 7.7). `capacity.guests` field confirmed (replacing v0.1.x `guests_max`). Prior art section updated: ACP removed; UCP, MCP, A2A sections added. Agent behaviour guidelines 6.14–6.16 added. Open questions 8.10–8.11 added. `openstr_version` updated to `"0.2"`. |

---

## 10. References

- [Schema.org LodgingBusiness](https://schema.org/LodgingBusiness)
- [Google Universal Commerce Protocol (UCP)](https://developers.google.com/commerce/ucp)
- [Google Agent Payments Protocol (AP2)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io)
- [Agent-to-Agent Protocol (A2A)](https://developers.google.com/agent-to-agent)
- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
- [RFC 5545 — iCalendar](https://datatracker.ietf.org/doc/html/rfc5545)
- [RFC 8615 — Well-Known URIs](https://datatracker.ietf.org/doc/html/rfc8615)
- [ISO 3166-1 alpha-2 country codes](https://www.iso.org/iso-3166-country-codes.html)
- [ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)
- [ISO 639-1 language codes](https://www.iso.org/iso-639-language-codes.html)
- OpenSTR `rfc.identity_trust.md` — Host and Guest Credential interfaces
- OpenSTR `rfc.availability_query.md` — Availability and pricing query
- OpenSTR `rfc.booking_confirmation.md` — Booking request and confirmation flow
- OpenSTR `docs/protocol-landscape.md` — Protocol landscape and ecosystem positioning
