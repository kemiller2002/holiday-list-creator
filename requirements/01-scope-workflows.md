# 01. Product scope and user journeys

Status: proposed baseline, 2026-11-04.

## Product intent

One person may have two separate guest name inputs and an optional household/family designation. A **contact** is a mailing household/recipient record. A **person** is a named individual linked to that contact. A **holiday list** is an independently saved mailing campaign (for example "Christmas 2026"). A **list entry** is a selected contact plus the mailing snapshot used by that specific list.

Never collapse Guest 1 and Guest 2 into an irreversible display-only string. Do not drop `Plus` or additional named people when migrating old records. Preserve the imported values even when an editable presentation is cleaner.

## User journeys

### J-01: Open contact directory
1. Sign in and load the current contact dataset from the private server.
2. See search, filters, active/inactive indicator, mailing-readiness status, and recipient overview.
3. Open an existing contact with Guest 1, Guest 2, surname/household details, address, missing-field indicators, and source provenance.
4. Filter by active, deleted, missing-address, and potential duplicate. Default is active only.

### J-02: Add and edit a contact
1. Select "Add contact" or edit an existing contact.
2. Enter **Guest 1** separately; optionally enter **Guest 2**. Each field is independently editable. A lone Guest 1 is valid.
3. Preserve a separate surname, optional other named guests, family/household label, free-text addressee override, and all structured address fields.
4. Missing optional values stay `null` with an explicit missing state, not fabricated filler or implicit omissions. A record with incomplete address can still be saved, but not silently classified mail-ready.
5. Save with inline validation and duplicate warnings. Save edits under the same immutable contact ID.

### J-03: Soft-delete and restore
1. Use a clearly labeled Delete action in contact detail or the directory.
2. Confirm and optionally enter a reason. The server sets `is_deleted=true`, UTC `deleted_at`, authenticated actor `deleted_by`, and `deletion_reason`.
3. The contact disappears from default active selection but remains searchable under Deleted; retain all people and source history.
4. Restore reverses active-state metadata per the private repository's v1.1.0 deletion contract; no physical deletion.
5. Deleting a person separately does not cascade delete the household; deleting a household hides it from current mailing selections without erasing historical lists.

### J-04: Create a holiday list
1. Select "New holiday list"; provide title, year, and optional occasion/notes.
2. Initialize from currently active contacts, either all selected or selected by search/filter. Never import deleted contacts by default.
3. Show a one-row-per-household roster with distinct Guest 1 and Guest 2 columns; manual add/remove and bulk selection.
4. Save draft. Leaving and reopening the application must retain every selection, custom note, and current list snapshot.
5. Mark items as included/excluded and review "missing mailing information" before finalizing. Excluding an entry from a list **does not** delete its master contact.

### J-05: Finalize and preserve history
1. Preview the address roster and the count of selected, excluded, incomplete, and soft-deleted source contacts.
2. Finalization freezes all selected list entries as a dated snapshot, with contact ID linkage and address/name values as of finalization.
3. Finalized lists are read-only. To change a finalized list, create a new revision or copy; earlier revision remains available unchanged.
4. Historical list view remains readable even if a contact is edited, changes address, or is later soft-deleted.

### J-06: Export
1. Open a draft or saved list and choose Export CSV.
2. Show a preview of included rows and a warning for missing required mailing information; allow either "only mail-ready" or "all included with missing values blank and warnings" with an explicit user choice.
3. Export one row per selected contact/list entry with separate `Guest 1` and `Guest 2` columns, recipient, address lines, city, state, ZIP/postal code, country, and contact ID.
4. Preserve commas, quotes, line breaks, Unicode, and postal-code leading zeroes. No labels or "send to printer" control fields. Exporting never changes the saved list.
5. Show total exported vs skipped and a machine-readable warning report (in-app, without exposing contacts publicly).

## In scope for version 1

Authenticated owner workflow; contact CRUD without physical deletes; two-guest editing; explicit missing values; historical lists with immutable finalized versions; review/preview; CSV export; phone-friendly, accessible interface; unit and integration tests.

## Out of scope for version 1

Sending mail, printing directly to a printer, label-layout designer, bulk email/SMS, CRM sync, AI inference of missing addresses, automatic postal validation, family-relationship inference, public account registration, and multi-tenant billing.

## Non-negotiable invariants

- A human name is never lost because of merging, cleanup, deletion, or CSV export.
- A saved/finalized list is historical evidence, not a live query over today's contacts.
- Zero sensitive source data appears in this public application repository or public deployments.
- No action labeled "Remove from list" can soft-delete a master contact.
