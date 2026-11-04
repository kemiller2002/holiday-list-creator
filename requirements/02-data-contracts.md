# 02. Data contracts, IDs, and history

## Existing source of truth

The private repository `kemiller2002/holiday-list` is the starting authority for contact data. Inspected canonical file: `private-data/christmas-cards-2026.json`, `meta.schema_version = 1.1.0`. Existing associated projections: `contacts-2026.json`, `people-2026.json`, `source-records-2024.json`. They currently contain 61 contacts, 90 person entries, and 154 original source rows. These are *baseline figures*, not counts to hardcode.

Contact fields already include `contact_id`, `card_recipient`, `names` (including `guest_1`/`guest_2`), `people`, `household_designation`, `address`, `address_as_entered`, `address_last_verified`, `missing_fields`, `review_reasons`, `mailing_status`, source references, and four soft-delete fields.

### Field contract (FR identifiers)

| ID | Field | Type / rules |
| --- | --- | --- |
| FR-C-001 | `contact_id` | Unique immutable string; never recycled, including after soft deletion. |
| FR-C-002 | `guest_1` | Editable nonblank string for newly created records; preserve raw imported value in provenance. |
| FR-C-003 | `guest_2` | Editable string or explicit `null`, independent of Guest 1. |
| FR-C-004 | `surname` | String or `null`; do not assign automatically to unverified additional guests. |
| FR-C-005 | `additional_names_or_household` | String or `null`; preserve names and "Family" indications. |
| FR-C-006 | `card_recipient` | Editable addressee / generated suggestion, stored separately from guest inputs; never overwrite manual overrides. |
| FR-C-007 | `address.line_1` | String or `null`; required for mail-ready. |
| FR-C-008 | `address.line_2` | String or `null`; optional. |
| FR-C-009 | `address.city` | String or `null`; required for mail-ready. |
| FR-C-010 | `address.state` | String or `null`; required for US mail-ready. |
| FR-C-011 | `address.postal_code` | String or `null`, never integer; required for mail-ready. |
| FR-C-012 | `address.country` | String or `null`; `US` is an existing default only when known, not inferred for unknown imported records. |
| FR-C-013 | `missing_fields` | Explicit missing-path index; synchronize with null fields. |
| FR-C-014 | `address_last_verified` | Timestamp or `null`, no automatic verification claims. |
| FR-C-015 | `is_deleted` | Boolean default `false`. |
| FR-C-016 | `deleted_at` | UTC ISO 8601 timestamp or `null`. |
| FR-C-017 | `deleted_by` | Actor identifier or `null`. |
| FR-C-018 | `deletion_reason` | String or `null`. |
| FR-C-019 | `people` | Array with immutable `person_id`, original `name_as_entered`, display name, role; retain every named person. |
| FR-C-020 | `source_record_ids` | Original provenance IDs; never replace or rewrite historical source values. |
| FR-C-021 | `version` | Monotonic revision for compare-and-swap writes; derive for legacy import if not already present. |

Existing legacy name shape remains readable: `names.guest_1`, `names.guest_2`, `names.combine_guests`, `names.surname`, `names.surname_combined`, `names.additional_names_or_household`, and `names.full_name`. Display of names must not assume one person per contact. New schema may be an additive v1.2 migration, not a destructive rewrite.

## Proposed saved-list JSON v1 contract

A private JSON file at `private-data/lists/{year}/{list_id}.json` is one versioned list aggregate; a private index `private-data/lists/index.json` may speed browsing, but all list files remain authoritative for their own content.

```json
{
  "schema_version": "1.0.0",
  "list_id": "HL-2026-0001",
  "title": "Christmas Cards 2026",
  "year": 2026,
  "occasion": "Christmas",
  "notes": null,
  "status": "draft",
  "revision": 1,
  "created_at": "2026-11-04T18:00:00Z",
  "created_by": "owner",
  "updated_at": "2026-11-04T18:00:00Z",
  "finalized_at": null,
  "prior_revision_id": null,
  "entries": [
    {
      "entry_id": "HLE-0001",
      "contact_id": "HC-2026-000",
      "include": true,
      "guest_1": "Sample",
      "guest_2": null,
      "card_recipient": "Sample Recipient",
      "additional_names_or_household": null,
      "address": {
        "line_1": null,
        "line_2": null,
        "city": null,
        "state": null,
        "postal_code": null,
        "country": null
      },
      "missing_fields": {
        "address.line_1": "missing",
        "address.line_2": "missing",
        "address.city": "missing",
        "address.state": "missing",
        "address.postal_code": "missing",
        "address.country": "missing"
      },
      "notes": null,
      "source_contact_version": 1
    }
  ]
}
```

**The example is synthetic.** The timestamps and IDs are examples only. Production IDs must be unique, not copied from examples.

### Saved-list rules

- **L-01:** `list_id` stable across revisions. Each finalized version gets a unique revision identity and immutable bytes. Saving a draft increments revision / storage version.
- **L-02:** Every entry has immutable `entry_id` and `contact_id` plus **copied values** for guest names, addressee, complete address, missing-field state, and any optional personal message.
- **L-03:** The full canonical contact remains elsewhere; no extra person is deleted by making a two-guest mailing row. Additional named people remain available in contact detail and can be carried into a `named_people_snapshot` array when present.
- **L-04:** Saved drafts are persisted, not reconstructed on each load from current contacts. User can deliberately refresh a draft entry from the current contact and must see the changed-field diff before applying it.
- **L-05:** Finalized snapshots are immutable. To amend, copy to a new revision linked via `prior_revision_id`. Never mutate a finalized historical copy.
- **L-06:** Soft deletion of a current contact never alters prior lists. A new list excludes deleted contacts by default; an old list may show a "source contact deleted" banner without changing its snapshot.
- **L-07:** Deleted list records can be archived/hidden using `is_archived` and timestamps, not physically removed. Distinguish archive from contact `is_deleted`.
- **L-08:** One list may contain a given `contact_id` at most once. Distinct contacts at the same postal address remain distinct unless a user explicitly consolidates them (a separate reviewed operation).
- **L-09:** Missing is not the same as blank string. On save, normalize empty *new* optional input to `null` but preserve imported raw values in historical source records.
- **L-10:** Historical list creation and export are independent of the live address book, except for explicit user-requested draft refreshes.

## Migration and consistency

- Build an importer that reads existing v1.1.0 data and preserves all contact IDs, person IDs, names, 154 source records, and deleted metadata.
- Existing contact JSON bundle remains canonical. Writes to current contact data must update its split projection files in the same revision-protected logical transaction; partial writes are detectable and recoverable.
- Do not move source records into list snapshots; retain stable references for provenance.
- New/edited contacts must retain original `name_as_entered` information and not invent known surnames for guests.
- Validate each write: unique IDs, valid status transitions, no missing required JSON keys, soft-delete consistency, snapshot completeness.
