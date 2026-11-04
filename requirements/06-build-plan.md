# 06. Build plan and implementation decisions

**Status: requirements only.** This document does not claim software, infrastructure, authentication, or tests are implemented.

## Smallest useful vertical slice

| Phase | Scope | Exit criteria |
| --- | --- | --- |
| 0. Preflight | Check F# SDK; Limen, Forma, and Folio compatibility; load private `contacts-v2.json` schema 2.0.0 and validate frozen import preservation; confirm hosting/auth | Verified, pinned working dependencies and safe synthetic fixture |
| 1. Domain | Typed v2 contact, nested person, guest role references, null/missing fields, soft delete, snapshots, CSV | Domain tests T-001..T-020 and T-035..T-039 pass on synthetic fixtures |
| 2. Persistence | Authenticated F# API, server-only GitHub App access, version-controlled contact and list repository writes | Tests for safe single-master read/edit, concurrency, unmodified legacy archive and service restart |
| 3. Contacts screens | Forma directory, details, Guest 1 + Guest 2 form, create, edit, soft delete, restore | Persisted UI workflows on phone and desktop |
| 4. List screens | New list, select members, save draft, reopen, finalize, historical revisions, archive | Historical values frozen; all data survives reload |
| 5. Export and hardening | CSV with separate guest columns, completeness warnings, access controls, accessibility and layout tests | All required test IDs pass and evidence recorded |

Do not publish a mock-only UI as complete. A feature becomes complete only when it safely saves and reloads its data.

## Implementation choices to resolve

- **D-01 Auth:** Prefer a verified existing Fides implementation; if unavailable use a small supported owner-only login with authenticated server sessions. No public user signup.
- **D-02 Hosting:** Minimal supported deployment for F# backend. Public static UI is possible only if protected API + no live data in assets.
- **D-03 GitHub App:** Server-side installation credential scoped only to `kemiller2002/holiday-list`. No credential, private JSON, or personal address in public client.
- **D-04 Atomicity:** Resolved: use ONLY `private-data/contacts-v2.json` for editable contacts, with one version-checked Git blob write. Legacy v1.1.0 files are immutable and must not be regenerated. List finalization across files must use an atomic Git tree commit or idempotent recoverable protocol.
- **D-05 List versioning:** Draft revision with compare-and-swap, immutable final revision, separate revision ID on copy.
- **D-06 Address check:** Incomplete values allowed for contact save and list draft; explicit warnings at finalization/export. No fabricated validation.
- **D-07 Additional people:** Carry all named people into each list entry's `people_snapshot[]`; default CSV has separate Guest 1, Guest 2, Additional Named People, and Additional Names / Household columns.
- **D-08 Recovery:** Restore from repository history; document audit metadata and idempotency semantics.
- **D-09 Security review:** Verify CORS, sessions, logging, secrets, browser storage, export protections before live access.

## Planned repository layout

```text
src/
  HolidayList.Domain/
  HolidayList.Api/
  HolidayList.Web/
tests/
  HolidayList.Domain.Tests/
  HolidayList.Api.Tests/
  HolidayList.Web.E2E/
requirements/
  01-scope-workflows.md
  02-data-contracts.md
  03-ui-screens.md
  04-architecture-security.md
  05-acceptance-tests.md
  06-build-plan.md
  requirements.json
```

Directories under `src/` and `tests/` are **planned**, not created by this requirements change.

## Engineering rules

- F# application and backend. Limen WASM frontend with Forma controls; Folio only when printing beyond CSV is needed. Do not default to Bolero.
- Use the latest **verified compatible** Echelon releases, not guessed package versions. Consider Conditor/Praxis for requirements, Ordo for patterns, Dokimos for tests when verified; record invocation evidence before claiming use.
- Strong IDs for requirements and tests. Merge gates must fail on incomplete or broken requirements, tests, or privacy controls.
- No Kubernetes, Terraform, microservice split, or database requirement for the initial small application. The private repository JSON backend is the selected initial persistence system; app domain uses contact schema v2.0.0.
- Never copy production names or addresses into this public repository. Use synthetic fixtures and sample names only.
- Preserve every contact, person, historical name, and source record through editing. The v1.1.0 files are permanently read-only; writes go solely to `contacts-v2.json` under a version-protected Git commit. Verify guest role references and historical list snapshots.
- Defer bulk imports, USPS checking, label templates, multiuser roles, CRM integrations, and automation until the core application has passed tests.

## First delivery acceptance

A signed-in owner can add two guests to one contact, save and reload it, soft-delete and restore it, create a holiday list from selected contacts, save/reopen a draft, finalize an immutable copy, and export a correctly formatted CSV. All of this works on a phone with protected data.
