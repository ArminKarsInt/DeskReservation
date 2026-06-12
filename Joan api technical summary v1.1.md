# Joan API – Technical Summary & Test Results
**Project:** Guest Visit Automation (Power Automate integration)
**Author:** Armin Nemalhabib
**Date:** 2026-06-11
**Version:** 1.1
**Status:** Test phase complete — all blocking questions resolved, ready for Power Automate build

---

## 1. Goal of This Document

This document explains what the Joan API can do, how we connected to it, which tests were run, and what the results mean. It serves as the technical foundation for building the Power Automate flow that automatically books a desk and parking spot when a guest visit is registered.

---

## 2. What Is the Joan API?

Joan is the workplace booking system used at Karsten International. It manages desk and parking reservations across all floors of the Headquarters building. Joan has an API (Application Programming Interface) — a way for external software (like Power Automate) to talk to Joan programmatically, without a human clicking in the interface.

In plain terms: **the API lets us tell Joan "book desk X for person Y on date Z" from a script or automated flow**, and Joan will do it exactly as if a user had done it manually through the app.

---

## 3. Authentication — How We Log In

Before any API call can be made, the system must prove it is allowed to act on behalf of Karsten International. This works as follows:

**Endpoint:** `POST https://portal.getjoan.com/api/token/`

**How it works:**
1. Send the API Key and API Secret (stored securely, never in code).
2. Joan returns a **Bearer token** — a temporary password valid for **10 hours (36,000 seconds)**.
3. Every subsequent request includes this token in its header.

**Important discovery:** Joan's server silently drops connections that don't include a `User-Agent` header (a field that identifies what software is making the request). Without it, the connection was forcibly closed before any response was received. Adding a User-Agent string fixed this immediately. **Any future integration (Office Scripts, Power Automate HTTP) must send a User-Agent header.**

**Token response example:**
```json
{
  "access_token": "***masked-example***",
  "expires_in": 36000,
  "token_type": "Bearer",
  "scope": "read write"
}
```

Access tokens are short-lived credentials and must never be stored in documents or code.

**Implication for Power Automate:** The token must be fetched at the start of each flow run. Since flows are typically triggered by a single guest registration event and run in seconds, the 10-hour expiry is not a concern — a fresh token per run is the cleanest approach.

---

## 4. Reading Data — What Joan Can Tell Us

### 4.1 All Desks
**Endpoint:** `GET /api/2.0/portal/desks/`

Returns a list of all desks in the system. There are **178 desks** across all floors of Headquarters. Each desk object contains:

| Field | Meaning |
|---|---|
| `id` | Unique identifier (used when booking) |
| `name` | Human-readable name, e.g. `6-11B` |
| `floor.name` | Which floor, e.g. `6th floor` |
| `building.name` | Always `Headquarters` |
| `active` | Whether the desk can be booked |
| `tz` | Timezone — always `Europe/Amsterdam` |

**Notes:**
- The API returns results in pages of 50. To get all 178 desks, the code must follow the `next` link in the response until it runs out. A pagination helper was built for this; the full enumeration is saved as `desks_all.json`.
- The desks endpoint also returns **non-desk bookables**: 10 "Laptop lockers" entries and a "Parking" floor entry. Always filter by `floor.name` when selecting real desks.

### 4.2 All Parking Spots (Assets)
**Endpoint:** `GET /api/2.0/portal/assets/`

Returns all bookable assets. There are **25 parking spots**, including visitor spots (`Nr. 1 Visitors`, `Nr. 2 Visitors`) and numbered employee spots (`P03`–`P25`, of which P22–P25 are marked FLEX). Each asset has an `asset_type` field confirming it is a `Parking` type.

### 4.3 Existing Reservations
**Endpoint:** `GET /api/2.0/portal/desks/reservations/`

Tested facts:

- Total volume at time of testing: **~21,955 reservation records**, paginated 50 per page (~439 pages), sorted **ascending by start date beginning June 2024** — current and future bookings sit at the END of the pagination chain.
- Fetching everything and filtering locally is therefore **not viable** (439 API calls per flow run against a shared 240 req/min rate limit).
- **Server-side date filtering is supported and tested** (see Test 5): `?start=<ISO>&end=<ISO>` returns only reservations in that window. This is how the Power Automate availability check will query Joan — one call per visit window.
- `?ordering=-start` (newest first) and `?limit=500` (larger pages) also work. Not needed for the main flow, but useful for diagnostics.
- The reservation **list** objects contain exactly these fields: `id, user, desk, start, end, tz, timeslot_id, recurring, checked_in, visit_id, floor, building`. They do **not** contain `additional_info` (see 4.4).
- Recurring bookings (e.g. freq `every_weekday`) are returned **pre-expanded as individual reservation rows** — no recurrence logic is needed on our side.
- The API token sees **all reservations company-wide** (all employees and the guest account), so an availability check automatically respects every booking channel: employees via the app, externals via the guest login, and manual Office Management bookings.

### 4.4 The `additional_info` field — partially hidden

`additional_info` is accepted on creation and echoed in the creation response, but:

- It is **not returned in reservation list responses**.
- It **is returned** by the single-reservation detail endpoint `GET /desks/reservations/{id}/` (verified in Test 6).
- It is **not visible anywhere in the MyJoan interface** — all external bookings simply appear as "Guest Karsten".

**Implications:**
- We **cannot search Joan by our RequestID**. The reservation `id` returned at creation time, stored immediately in SharePoint, is the primary link between a SharePoint request and its Joan booking.
- Given a reservation `id`, the detail GET can **verify** which request a booking belongs to — useful for troubleshooting.
- SharePoint (not Joan) answers the question "which external sits at which desk". This is by design: visitor identity lives in our Microsoft 365 environment, Joan only holds the anonymous guest booking.

---

## 5. Test Results

All tests were run on **2026-06-11** using the `gast@karsten.nl` guest account. All test bookings were created and then deleted. No permanent reservations were left in the system.

---

### Test 1 — Desk Booking Lifecycle

**Goal:** Confirm that the API can create and delete a desk reservation.

**What was sent:**
```json
{
  "user_email": "gast@karsten.nl",
  "desk_id": "6e7f7263-d0f1-4011-9f9e-df2aadec0352",
  "start": "2026-06-12T06:00:00.000Z",
  "end": "2026-06-12T16:00:00.000Z",
  "tz": "Europe/Amsterdam",
  "timeslot_id": "bbeb6871-4776-4696-acd8-42f129ecb0ae",
  "additional_info": "TEST-001 Armin"
}
```

**What Joan returned (201 Created):** a full reservation object including the new reservation `id`, the desk name (`6-11B`), the booked times, and the `additional_info` echoed back.

**Deletion:** `DELETE /api/2.0/portal/desks/reservations/{id}/` → **204 No Content**

**Result: PASS**

---

### Test 2 — Desk Duplicate / Overlap Protection

**Goal:** Find out whether Joan itself blocks two bookings for the same desk at the same time.

**What was done:** The exact same booking payload was sent twice in a row.

**Result:** The second attempt returned **400 Bad Request** — Joan rejected it.

**Meaning (corrected in v1.1):** Joan enforces overlap protection server-side, which acts as a **safety net against race conditions**. The flow **still performs an availability check** — its purpose is to *select* which desk from the pool to book (including the multi-day "same desk for all dates" intersection logic), not to prevent double-booking. Joan's 400 is the guarantee; our check is the chooser.

**Result: PASS**

---

### Test 3 — Parking (Asset) Booking Lifecycle

**Goal:** Confirm that parking spots follow the same pattern as desks, and whether `timeslot_id` is required for parking.

**What was sent:** Same structure as Test 1 but with `asset_id` instead of `desk_id`, endpoint `/api/2.0/portal/assets/reservations/`, and **no `timeslot_id`**.

**Result:** **201 Created** — parking does not require a timeslot; raw `start`/`end` times apply. The response uses `"asset"` instead of `"desk"` — Power Automate Parse JSON actions must use the correct schema for each type.

**Deletion:** `DELETE /api/2.0/portal/assets/reservations/{id}/` → **204 No Content**

**Result: PASS**

---

### Test 4 — Parking Duplicate / Overlap Protection

**Same test as Test 2, for parking.** The second identical booking attempt returned **400 Bad Request**. Same corrected conclusion as Test 2.

**Result: PASS**

---

### Test 5 — Server-Side Date Filtering (the former blocker)

**Goal:** Can the reservations endpoint filter by date window, so the flow doesn't have to read ~22,000 records?

**What was done:** A marker booking was created on 2026-07-01, then several query-parameter spellings were tried against the list endpoint.

| Query | Result |
|---|---|
| *(baseline, no params)* | count=21,955, oldest data first |
| `?start=2026-07-01T00:00:00Z&end=2026-07-01T23:59:59Z` | **count=31 — filtering works** (all results on July 1, marker included) |
| `?from=...&to=...` | ignored (baseline count) |
| `?ordering=-start` | works — newest first |
| `?limit=500` | works — larger pages |

**Meaning:** The availability check fetches exactly the visit window in **one API call** using `?start=...&end=...`. The blocker is resolved.

**Result: PASS**

---

### Test 6 — Detail GET Returns `additional_info`

**Goal:** Does `GET /desks/reservations/{id}/` work and include `additional_info`?

**Result:** **200 OK**, full reservation object returned **including** `"additional_info": "TEST-FILTER-001 Armin"`. This provides a verification path for troubleshooting (see 4.4).

**Result: PASS**

---

### Test 7 — Overlap Error Body Is Recognizable

**Goal:** When Joan rejects an overlapping booking, can the flow distinguish it from other errors?

**Result:** The 400 response body is:
```json
{
  "_error": [
    "Desk is already booked for that time."
  ]
}
```

**Meaning:** The flow can check the error text for "already booked" → **retry with the next desk in the pool**; any other error → **escalate to Office Management**. Implemented as a condition on the `_error` content.

**Result: PASS**

---

## 6. Key Technical Facts (Reference Card)

| Item | Value |
|---|---|
| Token endpoint | `POST https://portal.getjoan.com/api/token/` |
| Auth method | HTTP Basic Auth with API Key + Secret |
| Token lifetime | 36,000 seconds (~10 hours) — fetch fresh per flow run |
| User-Agent required | Yes — server drops connection without it |
| Desk booking endpoint | `POST /api/2.0/portal/desks/reservations/` |
| Parking booking endpoint | `POST /api/2.0/portal/assets/reservations/` |
| Delete desk booking | `DELETE /api/2.0/portal/desks/reservations/{id}/` |
| Delete parking booking | `DELETE /api/2.0/portal/assets/reservations/{id}/` |
| Reservation detail | `GET /desks/reservations/{id}/` — includes `additional_info` |
| **Date filtering** | **`?start=<ISO>&end=<ISO>` — supported, tested** |
| Extra query params | `?ordering=-start`, `?limit=500` also work |
| All-day timeslot ID (desks) | `bbeb6871-4776-4696-acd8-42f129ecb0ae` (08:00–18:00 Amsterdam) |
| Timeslot required for parking? | No — parking uses raw `start`/`end` times |
| Overlap protection | Server-side 400 with body `{"_error": ["Desk is already booked for that time."]}` |
| Timeslot behaviour | When provided, Joan uses the timeslot's own hours and ignores your `start`/`end` |
| Total desks | 178 (paginated 50/page; includes laptop lockers — filter by floor) |
| Total parking assets | 25 (2 visitor spots, P22–P25 marked FLEX) |
| Reservation volume | ~22,000 records, ascending by start date from June 2024 |
| Timezone | Always `Europe/Amsterdam` |

---

## 7. Design Decisions (Agreed)

- **External Desk Pool:** 9 approved 6th-floor desks: `6-4A, 6-4B, 6-6A, 6-6B, 6-7A, 6-7B, 6-13B, 6-14A, 6-14B` (decided 2026-06-11). Stored in the SharePoint config list "External Desk Pool" (seed file: `external_desk_pool.csv`) with Priority = list order. The flow only ever books from this list — desks near HR/Finance are excluded by design.
- **Standard test desk:** `6-13B` (lowest priority) — forgotten test bookings can only block the least valuable spot.
- **Booking identity:** All bookings are made under the shared guest account `gast@karsten.nl`, with the SharePoint RequestID written into `additional_info`. Visitor identity (name, email, license plate) lives in SharePoint, not in Joan.
- **Desk selection is automatic:** Externals do not pick a desk. The flow computes availability for the visit dates via the date-filtered reservations GET, intersects across dates (prefer the same desk for all days), and books the highest-priority free desk.
- **Cancellation:** No automated cancellation flow. Office Management cancels directly in MyJoan when needed. Consequence: **Joan is the source of truth for current booking state**; the SharePoint Status column may go stale after a manual cancellation and can be flipped to "Cancelled" by hand.
- **Timeslot implication:** For **desks**, the effective booking window is the timeslot's hours (08:00–18:00 Amsterdam) regardless of the `start`/`end` sent — no UTC conversion needed. For **parking**, raw `start`/`end` apply and UTC conversion **does** matter.

---

## 8. Open Questions

| # | Question | Why It Matters | Who |
|---|---|---|---|
| 1 | Do desk IDs remain stable when the floor plan is edited/republished in Joan? | If IDs change, our config list silently points at ghosts — drives the planned weekly reconciliation check | Domen (Joan support) |
| 2 | Can our plan have a second admin account or a restricted role? | Reduces single-person dependency on the one admin account | Domen (Joan support) |

All previously open questions (date filtering, guest pool, booking identity, cancellation) are resolved — see Sections 5 and 7.

---

## 9. Project Artifacts

| File | Purpose |
|---|---|
| `test.ipynb` | Python test notebook — the permanent Joan API diagnostic tool |
| `desks_all.json` | Full paginated desk enumeration (178 desks) |
| `external_desk_pool.csv` | Seed for the SharePoint External Desk Pool config list (9 desks, with priority) |
| `sixth_floor_desks.csv` | All 6th-floor desks, used in the approval walk-through |
| `desks_dump.json`, `assets_dump.json`, `reservations_dump.json` | Raw API page-1 dumps |
| `schemas/` | Single-object samples for Power Automate Parse JSON "Generate from sample" |

**Parse JSON caution:** `recurring`, `checked_in`, and `visit_id` alternate between `null` and real values. Schemas must type them as nullable (`["object","null"]` / `["string","null"]`), otherwise flow runs will fail on exactly the rows that matter.

---

## 10. Credentials

The API Key and API Secret are stored in a local `.env` file on Armin's workstation (test phase only — **to be moved to a restricted location before production**, e.g. inside the access-restricted bridge workbook or an environment variable of the production flow). They were created by Daniël Škorić via the Joan admin account (MyJoan → Settings → Integrations → API, token name `PowerAutomate-GuestVisits`) and can only be regenerated there.

---

## Changelog

- **v1.0** (2026-06-11) — Initial test results: auth, desk & parking lifecycle, duplicate protection.
- **v1.1** (2026-06-11) — Corrected Test 2/4 conclusions (availability check selects, Joan guards); added reservation-volume findings (~22k records, ascending sort); **date filtering confirmed working** (Test 5); detail GET returns `additional_info` (Test 6); overlap error body captured (Test 7); added desk pool decision (9 desks), cancellation decision, timeslot implications; reorganized open questions (2 remaining, both for Joan support); added artifacts list and credentials section.
