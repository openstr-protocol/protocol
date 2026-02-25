# RFC: Property Listing

**RFC ID:** openstr-rfc-001  
**Title:** Property Listing Schema and Discovery  
**Status:** Draft  
**Version:** 0.1.3  
**Created:** February 2026  
**Authors:** Daniel Bloom (openstr.org)  
**Supersedes:** openstr-rfc-001 v0.1.2  

---

## Abstract

This RFC defines the OpenSTR property listing schema — the foundational data structure that enables AI agents to discover, interpret, and reason about short-term rental properties without requiring a centralised platform intermediary. It specifies the required and optional fields for a machine-readable property listing, the discovery mechanism via a well-known endpoint, a three-tier property classification system, location disclosure conventions, the pricing declaration model including static pricing rules and dynamic pricing delegation, minimum stay rules including dynamic gap-fill and seasonal overrides, environment and safety disclosures, and the damage guarantee model based on identity verification tiers.

---

## 1. Motivation and Problem Statement

Short-term rental properties are currently discoverable and bookable only through closed platform intermediaries such as Airbnb, Vrbo, and Booking.com. These platforms provide significant value — aggregating supply and demand, providing trust infrastructure, and standardising the booking experience — but they do so at the cost of host autonomy, high commission rates, and lock-in.

As AI agents become the primary interface through which people search and book accommodation, the absence of an open, machine-readable standard for property listings creates a structural dependency on these intermediaries. An agent wishing to book accommodation on behalf of a user has no standardised way to discover or interpret a property outside of a platform's proprietary API.

OpenSTR addresses this by defining a common vocabulary for property listings that any host can publish and any agent can consume. This RFC defines the property listing schema — the first and most foundational layer of the OpenSTR protocol.

---

## 2. Prior Art

### 2.1 Schema.org LodgingBusiness

[Schema.org](https://schema.org/LodgingBusiness) defines a vocabulary for lodging businesses widely used for SEO and structured data. Schema.org provides a `petsAllowed` boolean field but does not define maximum pet counts, pet fees, property sub-types, environment categories, safety disclosures, or static pricing rules. OpenSTR defines its own native schema for all fields rather than inheriting Schema.org vocabulary directly, to ensure consistency and avoid agents needing to reason across two schemas simultaneously. The optional `schema_org` compatibility field allows hosts to also publish a full Schema.org object for web crawler compatibility.

### 2.2 Property Classification Conventions

The classification of short-term rental properties by type, sub-type, and occupancy mode is an industry-wide convention that predates any single platform. The OpenSTR property classification vocabulary is derived independently from multiple sources including the OpenTravel Alliance (OTA) accommodation type codes, Schema.org's `LodgingBusiness` and `Accommodation` type hierarchy, common real estate and hospitality industry terminology in wide use across multiple jurisdictions, and the W3C's work on accommodation vocabularies. Property descriptors such as `flat`, `house`, `houseboat`, `cottage`, `villa`, `yurt`, and `treehouse` are ordinary English-language descriptions of real-world property types that exist independently of any platform and are not the intellectual property of any single organisation. The occupancy classification of entire place, private room, and shared room is similarly a long-standing industry convention used across OTA schemas, hotel booking systems, and STR platforms globally. OpenSTR makes no claim that its classification vocabulary is derived from or endorsed by any specific platform.

### 2.3 iCal / CalDAV

The iCalendar standard (RFC 5545) is widely used for availability synchronisation between STR platforms. It addresses calendar state but not property description, pricing, or booking initiation.

### 2.4 OTA Connect / OpenTravel Alliance

The OpenTravel Alliance defines XML-based schemas for travel industry data exchange. These are primarily B2B standards designed for enterprise integrations and are not suitable for lightweight agent-to-host discovery.

### 2.5 Agentic Commerce Protocol (ACP)

ACP (OpenAI / Stripe) defines a standardised checkout and payment flow for AI agent commerce. OpenSTR is designed to be compatible with ACP for payment execution but addresses a different layer — property discovery and booking initiation — which ACP does not cover.

### 2.6 Google's Agent Payments Protocol (AP2)

AP2 (Google) defines agent-to-agent payment flows and includes a travel booking example. Like ACP, it addresses payment execution but not property discovery or description.

**Gap:** None of the above standards (2.1 through 2.6) define an open, lightweight, agent-readable property listing format with a three-tier property classification, availability querying, dynamic and static pricing rules, dynamic minimum stay rules, environment tagging, safety disclosures, bilateral trust credentials, or IDV-backed damage guarantees. This is the gap OpenSTR addresses.

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

---

## 4. Specification

### 4.1 Discovery Mechanism

```
GET /.well-known/openstr.json
```

Returns an `OpenSTRListing` object or array. Must be served over HTTPS with `Content-Type: application/json` and an `ETag` header.

### 4.2 OpenSTRListing Object

#### 4.2.1 Required Fields

| Field | Type | Description |
|---|---|---|
| `openstr_version` | string | Protocol version. Must be `"0.1"` for this spec. |
| `listing_id` | string | Host-assigned unique identifier. Stable across updates. |
| `listing_name` | string | Property name or title. Max 100 characters. |
| `listing_url` | string (URI) | Canonical URL of the human-readable listing page. |
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
| `host_credential` | object | Host credential reference. See `rfc.identity_trust.md`. |
| `policies` | object | Core policies object. See Section 4.8. |
| `safety_disclosures` | object | Safety disclosures object. See Section 4.9. |
| `updated_at` | string (ISO 8601) | Timestamp of last listing update. |

#### 4.2.2 Optional Fields

| Field | Type | Description |
|---|---|---|
| `amenities` | array[string] | Amenity vocabulary. See Section 4.10. |
| `environment_tags` | array[string] | Environment vocabulary. See Section 4.11. |
| `images` | array[object] | Image references. See Section 4.12. |
| `accessibility` | object | Accessibility features. See Section 4.13. |
| `year_built` | integer | Year the property was built. |
| `property_size` | object | `{ "value": number, "unit": "sqm" or "sqft" }` |
| `house_rules` | string | Free-text house rules. Max 1000 characters. |
| `neighbourhood_description` | string | Description of the surrounding area. Max 500 characters. |
| `transit` | string | Description of nearby public transport. Max 300 characters. |
| `languages_spoken` | array[string] | ISO 639-1 language codes. |
| `schema_org` | object | Full Schema.org `LodgingBusiness` object for web crawler compatibility. |
| `tags` | array[string] | Freeform host-assigned tags. Max 20 tags, 30 characters each. |

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
| `tiny_house` | `unique_space` | Compact purpose-built small home |
| `houseboat` | `unique_space` | Permanently or semi-permanently moored vessel |
| `shepherds_hut` | `unique_space` | Traditional or contemporary shepherd's hut |
| `yurt` | `unique_space` | Circular tent structure |
| `glamping_pod` | `unique_space` | Insulated outdoor sleeping pod |
| `converted_barn` | `unique_space` | Agricultural building converted to residential use |
| `castle` | `unique_space` | Historic castle or fortified building |
| `lighthouse` | `unique_space` | Converted lighthouse |
| `guest_suite` | `secondary_unit` | Self-contained suite attached to host's property |
| `garden_flat` | `secondary_unit` | Ground-floor or basement flat with garden access |

### 4.4 Location Object

#### Required location fields

| Field | Type | Description |
|---|---|---|
| `city` | string | City or town name |
| `region` | string | State, county, or region |
| `country` | string | ISO 3166-1 alpha-2 country code |
| `postcode_area` | string | Partial postcode or zip code (outward code only) |
| `approximate_lat` | number | Latitude rounded to 2 decimal places (~1km precision) |
| `approximate_lng` | number | Longitude rounded to 2 decimal places (~1km precision) |

#### Optional location fields

| Field | Type | Description |
|---|---|---|
| `neighbourhood` | string | Named neighbourhood or district |
| `landmarks` | array[string] | Nearby landmarks useful for agent reasoning |

**Note:** Agents must not reverse-geocode approximate coordinates to derive an exact address. Hosts may apply a random offset of up to 0.01 degrees to prevent triangulation. Full address and exact coordinates are released in the booking confirmation `location_detail` object.

### 4.5 Capacity Object

| Field | Type | Required | Description |
|---|---|---|---|
| `guests_max` | integer | Yes | Maximum number of guests permitted |
| `guests_recommended` | integer | No | Recommended guest count for comfortable occupancy |
| `beds` | integer | Yes | Total number of beds |
| `bed_types` | array[string] | No | `king`, `queen`, `double`, `single`, `sofa_bed`, `bunk` |

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

### 4.7 Minimum Stay Object

| Field | Type | Required | Description |
|---|---|---|---|
| `min_stay_nights` | integer | Yes | Default minimum stay in nights. For `dynamic` listings indicative only. |
| `min_stay_mode` | enum | Yes | `static` or `dynamic`. If `dynamic`, authoritative minimum stay for any date window is returned by the availability endpoint. |
| `max_stay_nights` | integer | No | Maximum stay in nights, if applicable. |

**Note:** Dynamic minimum stay rules — including gap-fill rules (reduced minimum to fill gaps between bookings) and seasonal rules (e.g. reduced minimum in low season) — are managed by the host's pricing system and resolved per date window in the availability response.

### 4.8 Policies Object

#### Required policy fields

| Field | Type | Description |
|---|---|---|
| `cancellation_policy` | enum | `flexible`, `moderate`, `strict`, `non_refundable` |
| `check_in_time` | string | Earliest check-in time in `HH:MM` 24-hour format |
| `check_out_time` | string | Latest check-out time in `HH:MM` 24-hour format |
| `pets` | object | Pet policy object. See Section 4.8.1. |
| `smoking_allowed` | boolean | Whether smoking is permitted |
| `events_allowed` | boolean | Whether events or parties are permitted |
| `instant_book` | boolean | Whether bookings are confirmed instantly or require host approval |
| `damage_guarantee` | object | Damage guarantee object. See Section 4.8.2. |
| `check_in_method` | enum | `in_person`, `self_checkin_keybox`, `self_checkin_smartlock`, `self_checkin_other` |

#### Optional policy fields

| Field | Type | Description |
|---|---|---|
| `children_allowed` | boolean | Whether children are permitted. Omission implies no restriction. |
| `infants_allowed` | boolean | Whether infants are permitted. |
| `age_restriction_min` | integer | Minimum guest age, if applicable. |
| `quiet_hours_start` | string | Start of quiet hours in `HH:MM` format. |
| `quiet_hours_end` | string | End of quiet hours in `HH:MM` format. |

#### 4.8.1 Pet Policy Object

| Field | Type | Required | Description |
|---|---|---|---|
| `allowed` | boolean | Yes | Whether pets are permitted |
| `pets_max` | integer | No | Maximum number of pets. Omission implies no stated limit if `allowed` is `true`. |
| `pet_fee` | object | No | `{ "amount": number, "currency": string, "per": "per_stay" or "per_night" }` |
| `pet_notes` | string | No | Freeform notes on pet policy. Max 300 characters. |

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
| `not_suitable_infants` | boolean | Property is not suitable for children under 2 years |
| `not_suitable_children` | boolean | Property is not suitable for children generally (e.g. unfenced drop, hazardous features) |
| `pool_present` | boolean | Pool present on property — relevant for families with young children |
| `hot_tub_present` | boolean | Hot tub present |
| `security_cameras_exterior` | boolean | Security cameras present on exterior of property. Hosts must disclose this. |
| `security_cameras_interior` | boolean | Security cameras present inside property. Interior cameras are prohibited in bedrooms and bathrooms. Must be disclosed if present in common areas. |
| `weapons_on_premises` | boolean | Weapons stored on the property. Must be disclosed. |
| `climbing_hazards` | boolean | Property has unfenced drops, staircases without guards, or other climbing hazards |
| `noise_disclaimer` | string | Freeform description of known noise sources. Max 300 characters. |
| `safety_notes` | string | Any additional safety information the host wishes to disclose. Max 500 characters. |

### 4.10 Amenity Vocabulary

Standardised values for the `amenities` array. Prefix freeform amenities with `custom:`.

**Connectivity:** `wifi`, `wifi_fast` (>100Mbps), `ethernet`

**Kitchen:** `kitchen_full`, `kitchen_kitchenette`, `dishwasher`, `washing_machine`, `dryer`, `coffee_maker`, `kettle`

**Climate:** `air_conditioning`, `heating_central`, `heating_portable`, `fireplace`

**Outdoor:** `garden_private`, `garden_shared`, `balcony`, `terrace`, `parking_private`, `parking_street`

**Safety:** `smoke_detector`, `carbon_monoxide_detector`, `fire_extinguisher`, `first_aid_kit` (also declare these in `safety_disclosures`)

**Accessibility:** `lift`, `step_free_access`, `wide_doorways`

**Entertainment:** `tv`, `tv_streaming`, `games_console`

**Work:** `dedicated_workspace`, `monitor_external`

### 4.11 Environment Tags Vocabulary

Standardised values for the `environment_tags` array. Describes the property's physical setting for agent-side filtering.

| Value | Description |
|---|---|
| `city_centre` | Located in or immediately adjacent to a city centre |
| `urban` | Urban residential area |
| `suburban` | Suburban residential area |
| `countryside` | Rural or semi-rural setting |
| `coastal` | Near the coast or seafront |
| `beachfront` | Direct beach access |
| `lakefront` | Direct lake access |
| `riverside` | Adjacent to a river |
| `mountain` | Mountain or highland setting |
| `forest` | Woodland or forest setting |
| `ski_in_ski_out` | Direct access to ski slopes |
| `desert` | Desert setting |
| `island` | Located on an island |
| `historic_district` | Located within a designated historic area |

### 4.12 Image Object

| Field | Type | Required | Description |
|---|---|---|---|
| `url` | string (URI) | Yes | HTTPS URL of the image |
| `caption` | string | No | Human-readable caption |
| `room_type` | enum | No | `bedroom`, `bathroom`, `kitchen`, `living_room`, `outdoor`, `entrance`, `other` |
| `width_px` | integer | No | Image width in pixels |
| `height_px` | integer | No | Image height in pixels |

### 4.13 Accessibility Object

All fields optional. Omission implies status unknown, not absent.

| Field | Type | Description |
|---|---|---|
| `step_free_access` | boolean | Step-free access from street to entrance |
| `lift_available` | boolean | Lift available if above ground floor |
| `wide_doorways` | boolean | Doorways of at least 80cm width throughout |
| `wet_room` | boolean | Step-free shower or wet room |
| `ground_floor` | boolean | Property entirely on ground floor |
| `accessibility_notes` | string | Freeform description. Max 500 characters. |

---

## 5. Example

A complete example of a valid OpenSTR property listing. The host uses PriceLabs for dynamic pricing and minimum stay management, requires identity-verified guests, and operates a smart lock entry system.

```json
{
  "openstr_version": "0.1",
  "listing_id": "host-abc123-prop-001",
  "listing_name": "Quiet Garden Apartment, Lincoln Park",
  "listing_url": "https://yourdomain.com/properties/lincoln-park-apartment",
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
    "guests_max": 4,
    "guests_recommended": 2,
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
    "credential_uri": "https://yourdomain.com/.well-known/openstr-host-credential.json",
    "issuer": "https://credentials.example-issuer.com",
    "issued_at": "2026-01-15T00:00:00Z"
  },
  "policies": {
    "cancellation_policy": "moderate",
    "check_in_time": "15:00",
    "check_out_time": "11:00",
    "pets": {
      "allowed": true,
      "pets_max": 1,
      "pet_fee": {
        "amount": 50,
        "currency": "USD",
        "per": "per_stay"
      },
      "pet_notes": "Small dogs and cats only. Must not be left unattended."
    },
    "smoking_allowed": false,
    "events_allowed": false,
    "instant_book": false,
    "damage_guarantee": {
      "mode": "idv_guarantee",
      "idv_level": "identity_verified"
    },
    "check_in_method": "self_checkin_smartlock",
    "children_allowed": true
  },
  "safety_disclosures": {
    "smoke_detector": true,
    "carbon_monoxide_detector": true,
    "fire_extinguisher": true,
    "first_aid_kit": true,
    "not_suitable_infants": false,
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
    "tv_streaming"
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
    "accessibility_notes": "Ground floor apartment with step-free access from street."
  },
  "languages_spoken": ["en"],
  "updated_at": "2026-02-01T09:00:00Z"
}
```

---

## 6. Agent Behaviour Guidelines

**6.1 Pricing:** For `dynamic` listings, `nightly_rate_indicative` is a floor estimate only. Agents must fetch authoritative pricing via the availability endpoint before presenting a confirmed price.

**6.2 Minimum stay:** For `dynamic` listings, `min_stay_nights` is indicative only. The authoritative minimum for a specific window is returned by the availability endpoint.

**6.3 Location:** Exact address is released only in the booking confirmation `location_detail` object. Agents must not attempt to derive exact addresses from approximate coordinates.

**6.4 Host credential:** Agents should verify the `host_credential` before presenting a listing as bookable. Listings with unverifiable or expired credentials should be flagged.

**6.5 IDV requirements:** Agents must check `damage_guarantee.idv_level` before initiating a booking and confirm the guest's credential satisfies it.

**6.6 Safety disclosures:** Agents must present `safety_disclosures` prominently when showing a listing to a user, particularly `not_suitable_infants`, `not_suitable_children`, `security_cameras_exterior`, `security_cameras_interior`, and `weapons_on_premises`.

**6.7 Pricing rules for static hosts:** For `static` listings, agents may use `pricing_rules` to pre-calculate estimated totals for agent-side filtering and pre-trip cost estimates. The availability endpoint remains the authoritative source before booking.

**6.8 Unknown fields:** Agents must ignore unknown fields to ensure forward compatibility.

---

## 7. Security Considerations

**7.1 HTTPS requirement:** All endpoints must be served over HTTPS.

**7.2 Host credential validation:** Agents must independently fetch and validate the `host_credential` document.

**7.3 URL injection:** Agents must validate that endpoint URIs are well-formed HTTPS URLs.

**7.4 Approximate location:** Hosts may apply a random offset of up to 0.01 degrees to prevent triangulation. This offset must not exceed 0.02 degrees.

**7.5 IDV credential trust:** A curated list of recognised OpenSTR IDV issuers will be maintained in `docs/trusted-issuers.md`. Candidate issuers include Google Identity, Apple, Stripe Identity, Onfido, and Yoti.

**7.6 Security camera disclosure:** Hosts must disclose all security cameras. Interior cameras in bedrooms or bathrooms are prohibited. Agents must surface camera disclosures to users before booking confirmation.

---

## 8. Open Questions

**8.1 Directory / aggregation:** Multi-host directory schema deferred to a future SEP.

**8.2 Multi-unit properties:** Parent/child listing pattern for multi-unit properties deferred.

**8.3 Cancellation policy terms:** Precise refund terms not standardised in v0.1.

**8.4 Recognised IDV issuers:** Community-governed trusted issuer registry proposed for v1.0.

**8.5 Review and reputation:** Post-stay reviews anchored to confirmed booking references defined in v0.2. Reviews will only be valid if cryptographically anchored to a confirmed OpenSTR booking reference.

**8.6 Smart lock integration:** Automatic time-limited access code generation via smart lock APIs is a v0.4 concern.

**8.7 Amenity and sub-type vocabulary completeness:** Both vocabularies are intentionally limited in v0.1. Community contributions welcome via the SEP process.

**8.8 Promotional pricing verification:** No mechanism currently exists for an agent to verify that a declared promotion is genuine. A promotion verification extension is deferred to a future SEP.

---

## 9. Changelog

| Version | Date | Notes |
|---|---|---|
| 0.1.0-draft | February 2026 | Initial draft |
| 0.1.1-draft | February 2026 | `name` → `listing_name`; pets expanded to object; `min_stay` expanded to object with `min_stay_mode`; two-stage location updated; `security_deposit` replaced with `damage_guarantee` with IDV tiers; Schema.org rationale clarified |
| 0.1.2-draft | February 2026 | `property_type` replaced with three-tier `property_classification` object; `pricing_rules` object added for static host discount and promotional structure; `safety_disclosures` object added as required field; `environment_tags` vocabulary added; `year_built` and `property_size` added as optional fields; sub-type vocabulary expanded |
| 0.1.3-draft | February 2026 | Section 2.2 added — Property Classification Conventions — documenting independent derivation of classification vocabulary from OTA, Schema.org, and industry-standard terminology |

---

## 10. References

- [Schema.org LodgingBusiness](https://schema.org/LodgingBusiness)
- [Agentic Commerce Protocol (ACP)](https://agenticcommerce.dev)
- [Google Agent Payments Protocol (AP2)](https://cloud.google.com/blog/products/ai-machine-learning/announcing-agents-to-payments-ap2-protocol)
- [W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)
- [RFC 5545 — iCalendar](https://datatracker.ietf.org/doc/html/rfc5545)
- [ISO 3166-1 alpha-2 country codes](https://www.iso.org/iso-3166-country-codes.html)
- [ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)
- [ISO 639-1 language codes](https://www.iso.org/iso-639-language-codes.html)
- OpenSTR `rfc.identity_trust.md` — Host and Guest Credential interfaces
- OpenSTR `rfc.availability_query.md` — Availability and pricing query
- OpenSTR `rfc.booking_confirmation.md` — Booking request and confirmation flow
