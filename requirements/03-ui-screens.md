# 03. UI screens and interaction requirements

## Design baseline

- F# + Limen WASM frontend; **Forma** components and palette/tokens. Do not implement ad hoc substitutes for working Forma buttons, inputs, tables, dialogs, and navigation.
- Mobile-first at phone widths 320-430 px, tablet 768 px, desktop 1280 px. Do not rely on hover or horizontal page scrolling for primary operations.
- Default light theme unless the owner selects otherwise. Focus on legible contrast, consistent input spacing, date/time display, clear empty states, and safe touch targets.
- Build pages using proper semantic headings and accessible field labels. Keyboard traversal, screen-reader status announcements, descriptive errors, visible focus, and accessible destructive-operation dialogs are required.
- No real contact names/addresses in Storybook-like examples, screenshots, E2E fixtures, or public CI artifacts.

## Navigation map

```text
Sign in / loading
     |
Dashboard
  |-- Contacts ---------------- Directory
  |                               |-- Contact detail (history + people)
  |                               |-- Add/Edit contact
  |                               |-- Deleted contacts / Restore
  |
  |-- Holiday lists ----------- Saved list index
                                  |-- Create list wizard
                                  |-- Draft editor / roster
                                  |-- Preview and finalize
                                  |-- Historical list detail (read-only)
                                  |-- Export CSV
```

## UI-001: Sign in and authorized loading
- Clear authentication state, retry, logout, expired session, and non-disclosing permission errors.
- App shell and loading affordance while Limen WASM initializes. Never flash protected contacts to unauthenticated clients.
- A public viewer cannot reach contact routes, APIs, or bundled data.

## UI-002: Dashboard
- Two primary actions: **Manage Contacts** and **Create Holiday List**.
- Show count of active contacts, count needing address review, number of saved lists, and recent lists.
- Counts must be computed from authorized data; totals are not static 2024 import constants.

## UI-003: Contact directory
- Search by card recipient, Guest 1, Guest 2, surname, and address. Filters: Active, Deleted, All, Missing Address, Potential Duplicate.
- Columns/cards: addressee, Guest 1, Guest 2, location (city/state), mailing status, actions.
- On phone, each contact is a stacked card with accessible actions; desktop can use a sortable table. Hide sensitive address preview in compact summary if shoulder-surfing mode is enabled (enhancement).
- Pagination or virtualized render with stable sort and IDs. Display match count and no-results state. Search must never permanently mutate data.

## UI-004: Contact details
- Show all stored names, including additional/household people, with distinction between source-entered and derived display.
- Show address with each missing field visibly marked **Missing** rather than silently disappearing.
- Show current status, address last verified (or "Not verified"), source record count, possible duplicate warning, and history.
- Buttons: Edit, Soft Delete, Restore (only when deleted), Back.
- Deleting a household must not implicitly delete individual named-person records.

## UI-005: Add / edit contact form

Recommended field order:

```text
Contact / recipient
  Guest 1                   [________________________________]
  Guest 2                   [________________________________]  (optional)
  Surname                   [________________________________]  (optional)
  Additional named people   [________________________________]  (optional)
  Household designation     [________________________________]  (optional)
  Card addressee            [________________________________]  (editable)
Mailing address
  Street address            [________________________________]
  Apt / Suite / Line 2      [________________________________]
  City                      [________________________________]
  State / Region            [________________________________]
  Postal code               [________________________________]
  Country                   [________________________________]
  Address verified          [status and verified-at value]
Other
  Notes (contact)           [________________________________]  (optional)
  [Cancel]                                [Save Contact]
```

- **Guest 1 and Guest 2 are separate inputs, role references into `contact.people[]`, focus targets, and export columns.** Do not duplicate editable name strings outside person records.
- Show auto-suggested card addressee built from current people; allow manual overrides without silently overwriting. On imported contacts, `card_addressee_override` is populated: when guest names change, prompt to keep or refresh the override.
- Guest 1 is required on a newly created contact; Guest 2 is optional. Three source values are unnamed `Guest` placeholders: show as unknown, not an identified person. Naming the placeholder creates a new person record without changing the original import evidence. Additional named people remain visible and editable independently.
- Mark required vs optional. When a mailing field is missing, show "Missing" and allow a draft-quality contact save. Do not invent addresses.
- Set postal code as text with no numeric coercion; keep leading zeroes.
- Changes that might indicate duplicate contacts prompt a review notice without automatic merging.
- Unsaved edits prompt before navigation. Disable Save while a write is pending and show server validation failures inline.
- Do not edit original historical source records in this form.

## UI-006: Soft-delete / restore confirmation
- Use explicit text: "Hide this contact from new holiday lists? Existing saved lists will remain unchanged."
- Provide optional reason, confirm/cancel, safe default focus. Include deleted by and UTC timestamp after success.
- Deleted contacts appear in a distinct filter with Restore.
- Excluding from a holiday list uses **Remove from this list** or **Exclude**, never the master-contact Delete control.

## UI-007: Saved holiday lists
- Table/cards: name, year, occasion, status (Draft/Finalized/Archived), last updated/finalized date, selected count.
- Search/filter by year/status. Open a past list without rehydrating contact details from the live directory.
- Actions: Create, Open, Resume Draft, Copy/Reopen as New Revision, Archive, Export CSV.
- Show data last saved time and any edit-conflict warning.

## UI-008: Create list wizard
- Step 1: List name, year, occasion, notes.
- Step 2: Filter active contacts and choose included households. Defaults and counts are visible. Option to include all active or start with none.
- Step 3: Preview names/address completeness and confirm Save Draft. Never force completion of addresses to save.
- Show duplicates/shared-address warnings but do not auto-merge distinct households.
- Allow adding a new contact without losing the wizard state.

## UI-009: Draft list editor
- One row/card per contact: include checkbox, recipient, Guest 1, Guest 2, city/state, address completeness indicator, edit-entry note.
- Search, filter, select/unselect, bulk select visible results with an explicit description of scope.
- Selecting a contact snapshots its current data. Later contact edits require user action to refresh draft entry.
- Show changes as "list snapshot vs current contact" without auto-syncing. Provide save state and conflict recovery.
- Add/remove entries without modifying master directory. Excluded entries may remain with `include=false` for traceability.

## UI-010: Preview and finalize
- Display final count, included vs excluded, mail-ready vs incomplete, potential duplicates, and estimated exported rows.
- Warn about unverified addresses. Explicitly confirm finalization and immutable history.
- After finalization, disable edit and provide Copy as New Revision. Historical view shows snapshot values, not updated contacts.

## UI-011: Export
- Choose **Mail-ready only** or **All included** (explicit warning for incomplete). Preview first five sample rows in user session, not logs.
- Show selected list title/revision and number of rows that will be exported.
- CSV columns in **exact order**:
  `Contact ID, Guest 1, Guest 2, Additional Named People, Card Recipient, Additional Names / Household, Address Line 1, Address Line 2, City, State, Postal Code, Country`. Resolve guest names and additional named people from the **saved list's people_snapshot[]**, not the live contact.
- Use one row per list entry where `include=true`, optional missing fields represented as empty CSV cells. Guest 2 is a distinct column even when absent. Unnamed `Guest` placeholders are labeled as such for review, not treated as verified person names.
- User receives a local download; do not upload exported CSV to a public artifact bucket or place it in a build directory.
- Use filename `holiday-list-{year}-{safe-list-slug}-r{revision}.csv`, sanitize filename and prevent path traversal.

## UI-012: Error, empty, offline, and race states
- Empty contact list, empty holiday list, no search matches, missing required data, forbidden access, expired login, storage unreachable, saving, saved, edit conflict, and export failure.
- An interrupted write must not masquerade as saved. No destructive recovery or silent overwrite.

## UI-013: Visual regression guardrails
- Long names, 120-character street lines, ZIP+4, foreign state/region, and 150% text zoom must wrap without leaking out of card boundaries.
- Test rounded Forma containers, dialogs, mobile menus, two-name form, dates, input focus, sticky headers, and long validation messages.
- Components must be Playwright-drivable using stable accessible labels/test identifiers when appropriate.
