# 05. Acceptance and verification plan

Every requirement must map to a test or documented acceptance check. These are **planned** tests, not executed results. Use synthetic contact fixtures only in the public repository. F# unit and integration tests cover domain and API behavior; Playwright exercises the browser.

## Test catalog

| ID | Verifies | Test and expected outcome |
| --- | --- | --- |
| T-001 | CT-001, CT-002 | Add Guest 1 only, then add Guest 2. Both separately saved and reloaded, second guest may be null. |
| T-002 | CT-003, CT-004 | Edit names and other named people. No unrelated person or historical name lost. |
| T-003 | CT-005, CT-006 | Save incomplete address. Null and missing field state explicit, review status accurate. |
| T-004 | CT-007 | Postal code starting with zero survives JSON and CSV. |
| T-005 | CT-008, CT-009 | Soft-delete then restore. No physical deletion; actor and timestamp recorded. |
| T-006 | CT-010 | Deleting household leaves people and source provenance available and historical lists intact. |
| T-007 | CT-011 | Two contacts sharing address trigger warning, never automatic merge. |
| T-008 | CT-012 | Stale contact update rejected, current write not overwritten. |
| T-009 | HL-001, HL-002 | Create list from selected contacts, no duplicate membership. |
| T-010 | HL-003 | Include/exclude on saved list does not modify master directory. |
| T-011 | HL-004 | Draft survives logout, reload, and service restart with same selections and entries. |
| T-012 | HL-005, HL-006 | Finalize; edit source contact; old finalized snapshot unchanged and read-only. |
| T-013 | HL-007 | Soft-delete contact later; absent from new list but visible in previous finalized list. |
| T-014 | HL-008 | Copy finalized list into new editable revision without overwriting original. |
| T-015 | HL-009 | Archive list hides in default view but retains archival access. |
| T-016 | CSV-001, CSV-002 | Export one row per included household with Guest 1, Guest 2, and Additional Named People as distinct columns resolved from frozen people snapshots. |
| T-017 | CSV-003 | For incomplete address, mail-ready mode skips with count, all-included mode retains blank fields with warning. |
| T-018 | CSV-004 | Quoted fields, commas, newline, Unicode, leading-zero postal code parsed correctly. |
| T-019 | CSV-005 | Spreadsheet formula-looking value safely exported without modifying source JSON. |
| T-020 | CSV-006 | Export old finalized list uses its saved snapshot, not latest contact values. |
| T-021 | SEC-001 | Unauthenticated requests to contact, list, and export endpoints return no personal information. |
| T-022 | SEC-002 | Built public frontend, test fixtures, logs, and deploy assets contain no live personal records or credentials. |
| T-023 | SEC-003 | Competing edits produce one success and an explicit conflict, not data loss. |
| T-024 | SEC-004 | Failed contact edit does not alter the single canonical v2 JSON or any legacy files; interrupted multi-file list finalization is atomic or recoverable. |
| T-025 | SEC-005 | API enforces repository/path access restrictions and user authorization for all operations. |
| T-026 | SEC-006 | Read-modify-write of migrated v2 data preserves all contact/person IDs, import evidence, source IDs and optional nulls; rejects writes to frozen v1.1 files. |
| T-027 | UI-001, UI-002 | Complete contacts and holiday-list flows work at 375px and desktop widths without clipping. |
| T-028 | UI-003 | Fields/dialogs usable by keyboard, with accessible names, error announcements, and visible focus. |
| T-029 | UI-004 | Loading, no results, saved, offline, unauthorized, and edit-conflict states are accurate. |
| T-030 | UI-005 | Long names/addresses, 150% text sizing, and rounded Forma cards do not overflow. |
| T-031 | HL-010 | Retrying create/finalize does not produce duplicate contacts or historical versions. |
| T-032 | SEC-007 | Private responses are not cached in public assets or saved as permanent localStorage authority. |
| T-033 | CT-013, HL-011 | Additional named people survive list creation and export model representation. |
| T-034 | SYS-001 | All mandatory test results have evidence; no missing test is marked PASS. |
| T-035 | CT-014 | Each Guest 1 and named Guest 2 role resolves to one active person in the same contact; three existing unnamed Guest placeholders remain unlinked and preserved. |
| T-036 | CT-015 | Migration parity compares original and v2: 61 contacts, 90 person IDs/objects, 154 preserved source records, original name/address/addressee values and soft-delete fields. |
| T-037 | CT-016 | Editing an imported guest with a saved recipient override requires explicit keep/refresh choice, with no silent stale label rewrite. |
| T-038 | HL-015 | Archiving/unarchiving only changes catalog metadata; the exact finalized list-version bytes remain unchanged. |
| T-039 | SYS-002 | Structural JSON Schema validates contacts v2 and holiday-list versions v2; invalid references and duplicate IDs fail semantic validation. |
| T-040 | CT-017 | All 61 migrated contacts have unique, immutable Row IDs; sort, name edit, address edit, soft delete and restore preserve Row ID and Contact ID. |
| T-041 | CSV-002, CSV-007 | Saved-list CSV includes Row ID from the frozen entry snapshot; contact-directory CSV includes Row ID, Contact ID and Contact Revision before names. |
| T-042 | CSV-008 | Import dry-run rejects duplicate, missing, unknown or modified Row ID, mismatched Contact ID and stale revision without modifying any contacts. |

## Fixture coverage

Synthetic records must also cover stable row IDs after sorting, duplicate row IDs, changed row/contact ID pairing, missing/unknown row ID, stale contact revision, and a no-op failed reimport.

Synthetic records must cover: only Guest 1, Guest 1 plus Guest 2, additional guests, extended family, missing address line/city/state/ZIP, leading-zero postal code, long text, non-ASCII names, quotes/commas/line breaks, formula-looking text, two different contacts at one address, potential duplicates, deleted records, archive, source contact changing after finalization, and optimistic concurrency races.

## Release gates

1. Domain: all invariants, null/missing states, name preservation and historical snapshot tests pass.
2. Persistence: actual authorized single-file v2 contact writes, read after restart, safe conflicts, immutable legacy archive, atomic/recoverable list finalization, restore/rollback.
3. Privacy: authorization checks on all paths, public artifact checks, safe logs and export access.
4. CSV: all specified fields, quoting, leading zero, missing-value modes, spreadsheet-safe output.
5. UI: responsive mobile and desktop, keyboard access, screen-reader labels, loading/error states, Forma overflow checks.
6. Evidence: requirement IDs map to test IDs and test results. Any not executed test is NOT_RUN. No fake or stub success.

A reduced but tested end-to-end slice is preferable to an apparently complete application with missing persistence or failed checks.
