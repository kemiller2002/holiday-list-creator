# Data contracts: normalized v2

**Status:** the private data migration is complete. Application implementation and automated integration tests are not complete.

## Authority and historical data

The **only editable contact authority** is the PRIVATE repository file:
https://github.com/kemiller2002/holiday-list/blob/main/private-data/contacts-v2.json

- schema_version = 2.0.0
- Current baseline = 61 contacts with 90 named people
- Historical import = 154 original rows in frozen v1.1.0 archives
- Separate old contacts-2026.json, people-2026.json, source-records-2024.json, and christmas-cards-2026.json are READ-ONLY historical files, not editable projections.
- Never write real names or addresses to this public creator repository, browser application assets, CI logs, or public deployments.
- Structural contract: ../schemas/contacts-v2.schema.json

## Four logical entities

1. **Contact:** single mailing household with one structured postal address and a nested people array.
2. **Person:** one named person stored exactly once under that contact, with immutable person_id and preserved name_as_entered plus editable display_name.
3. **HolidayListVersion:** a persisted draft or immutable finalized revision with list_id and version_id.
4. **HolidayListEntry:** a selected contact's snapshot of names, people roles, address, and mailing label; never a mutable link to current contact data.

Raw imported source records remain an immutable archive outside the editable domain.

## Contact fields and IDs

| Field ID | Path | Contract |
| --- | --- | --- |
| FR-C-001 | contact_id | Immutable, stable across edits and soft deletes |
| FR-C-002 | people[] | Only editable collection of named individuals; no separate writable people index |
| FR-C-003 | people[].person_id | Unique ID; each person.contact_id matches parent |
| FR-C-004 | people[].name_as_entered | Preserve original entered name, never overwrite as a result of edits |
| FR-C-005 | people[].display_name | Current editable person name; reset derived-name flag when manually edited |
| FR-C-006 | guest_roles.guest_1_person_id | Required, refers to a named active person in people[] |
| FR-C-007 | guest_roles.guest_2_person_id | Optional nullable ID of a distinct named person in people[] |
| FR-C-008 | guest_roles.guest_2_unnamed_label | Optional null or unnamed source placeholder, mutually exclusive with named Guest 2 |
| FR-C-009 | surname | String or null; don't infer when unknown |
| FR-C-010 | additional_names_or_household | String or null; original household/extra text retained |
| FR-C-011 | household_designation | String or null |
| FR-C-012 | card_addressee_override | String or null; imported card recipients preserved exactly here |
| FR-C-013 | address.line_1 | String or null, required for a mail-ready address |
| FR-C-014 | address.line_2 | String or null, optional |
| FR-C-015 | address.city | String or null, required for mail-ready |
| FR-C-016 | address.state | String or null, required for domestic US mail-ready |
| FR-C-017 | address.postal_code | String or null; never coerce to number |
| FR-C-018 | address.country | String or null; no invented country |
| FR-C-019 | missing_fields | Recompute from null current fields on save; not independently editable |
| FR-C-020 | address_last_verified | Timestamp or null; never assume a 2024 address is current |
| FR-C-021 | source_record_ids | Preserve immutable references to original 154 source rows |
| FR-C-022 | legacy_import | Immutable name, address, original card label, review and duplicate evidence |
| FR-C-023 | review_flags | Previously suspected duplicates and source review reasons; informational |
| FR-C-024 | revision | Increment on edit; reject stale expected Git blob SHA |
| FR-C-025 | notes | Nullable editable text |
| FR-C-026 | is_deleted | Boolean; false by default |
| FR-C-027 | deleted_at | UTC ISO8601 timestamp or null |
| FR-C-028 | deleted_by | Actor ID or null |
| FR-C-029 | deletion_reason | Optional text or null |

All properties remain present even if null. A missing optional Guest 2 or address line 2 is not a mail-readiness error.

## Guest 1 and Guest 2

The UI has **two separately labeled guest fields**. Their current editable names resolve to people[].display_name by guest_roles person ID. Do not introduce separate editable guest_1_name/guest_2_name strings; those create synchronization bugs.

- Guest 1 must reference exactly one active person on a new contact.
- Guest 2 is optional and must be different from Guest 1 if named.
- Three imported Guest 2 values are literal "Guest" placeholders; their guest_2_unnamed_label is retained, with no invented person ID. When the user names the person, create a new person ID and clear the live placeholder, while keeping the old source value in legacy_import.
- Additional named people beyond two guest slots stay in people[] and in each saved list's snapshot.
- Editing a person's display name does not erase their imported name_as_entered or source records.
- Editing guests when card_addressee_override is present must prompt to keep or update that override, never silently change the mailing addressee.
- Deleting a person referenced as Guest 1 requires a deliberate reassignment before the contact can be used for new mailings.

## Derived state

Mailing readiness is computed using active state, guest role resolution, and required address fields. Search indexes, display addressee (when override is null), and missing_fields are derived. Do not persist an independently editable mailing_status or split person index. Historical imported mailing_status is preserved only in legacy_import.

## Saved holiday lists (schema v2)

Structural contract: ../schemas/holiday-list-v2.schema.json

- A stable list_id groups versions; version_id uniquely identifies each version; revision is monotonic; previous_version_id connects a copied prior final version.
- Status is **draft** or **finalized**. Finalized version files must never be overwritten, including when changing archive status.
- A draft can be updated with optimistic concurrency and saved durably. Do not automatically refresh draft snapshots when contacts change; offer an explicit diff/refresh action.
- Each entry carries entry_id, contact_id, include, guest_1_person_id, guest_2_person_id, guest_2_unnamed_label, people_snapshot[], card_addressee_snapshot, surname_snapshot, additional_names_or_household_snapshot, household_designation_snapshot, address_snapshot, missing_fields_snapshot, source_contact_revision and optional notes.
- Guest IDs in a list entry resolve **only to its copied people_snapshot[]**. CSV and historical display must not fetch new contact values.
- Store drafts under private-data/holiday-lists/{list_id}/draft.json, final versions under private-data/holiday-lists/{list_id}/versions/{version_id}.json, and archive flags in separate private-data/holiday-lists/index.json. Immutable finalized versions never change merely because they're archived.
- An entry is unique by contact_id within a list; unrelated recipients at the same postal address are not automatically merged.
- Default CSV includes Guest 1, Guest 2, Additional Named People, and Additional Names / Household, each separately, plus recipient and all address columns.

## Preservation checks and implementation gates

The migration verified 61 unchanged contact IDs, 90 unchanged person IDs and person objects, untouched 154 original source records, retained source links, original names, addresses, import evidence, addressees and soft-delete fields. Guest 1 is resolved in all 61 records; 26 have named Guest 2 references and three have unnamed Guest 2 placeholders.

**NOT_RUN:** JSON Schema library validation, F# typed-domain integration, contact edit/reload, secure API checks, list snapshot persistence, CSV interoperability, browser accessibility, and runtime writes. Build agents must not report those passing without execution evidence.
