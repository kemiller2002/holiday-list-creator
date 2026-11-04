# 04. Architecture, privacy, persistence, and API

## Recommended version-1 architecture

```text
Authenticated browser (F# / Limen WASM / Forma)
                |
             HTTPS API
                |
       F# server-side monolith
       |      |          |
    Auth     Rules     JSON adapter
                         |
              GitHub App (server secret)
                         |
          PRIVATE kemiller2002/holiday-list
        canonical contact JSON + saved lists
```

The creator repository is public. The source `holiday-list` repository is private. **Public GitHub Pages is acceptable for static UI assets only** after verifying that no customer/person/contact data, GitHub secret, token, API key, or authenticated response is embedded or cached in publicly accessible assets. A protected API service is mandatory for persistence to the private repo. Limen/Forma package versions must be verified at implementation time rather than guessed here.

Do not add Kubernetes, Terraform, a general microservice platform, or an unnecessary database in v1. Use Folio only for a later printable card/report export, not required for CSV.

## A-01: Server-mediated repository access

- The server uses GitHub App installation authorization limited to the private `holiday-list` repository and minimum file-content permissions. Never embed a personal access token, GitHub App private key, or installation token in the WASM client.
- The private `private-data/contacts-v2.json` is the **only editable master** (schema 2.0.0): people are stored once, nested within contacts. The v1.1.0 bundle and split files are frozen historical artifacts and MUST NOT be modified by the creator application.
- Store drafts under `private-data/holiday-lists/{list_id}/draft.json`, immutable final versions under `private-data/holiday-lists/{list_id}/versions/{version_id}.json`, and archive/visibility metadata in a separate private index. The list entry contains a complete historical snapshot, including all named people.
- Protect every API endpoint with authenticated owner authorization; disallow public anonymous list enumeration and exports.
- The backend must not expose arbitrary repository paths, let clients specify a GitHub ref arbitrarily, or permit access to other repositories.

## A-02: Atomic writes and versioning

- Read master blob SHA, validate submitted `If-Match`/version, apply edits, write a new commit; reject stale updates with HTTP 409 and show a meaningful conflict UI.
- A contact edit is a single revision-checked write to `contacts-v2.json`; do not dual-write or regenerate legacy v1.1.0 people/contact files. Validate all guest role IDs, immutable provenance, and missing/null fields before commit. For list version transitions that change multiple paths, use a single atomic Git tree commit or an idempotent recoverable protocol, never a partial finalized state.
- Saved-list changes use a checked revision. Finalized list versions must not be overwritten, even by a retry or stale client.
- Use idempotency keys (or validated unique operation IDs) for contact create, list create, and finalize to prevent duplicate records after retry.
- Atomic operation history must be durable: Git commit SHA, UTC time, and actor, with no full PII values in log payload.
- Failed writes are reported, not silently retried as last-write-wins. Recover/read the latest state before retrying.

## A-03: Privacy and threat model

- Do not publish addresses or names in the public `holiday-list-creator` repository, CI logs, fixtures, screenshots, console messages, static HTML/JS/WASM files, source maps with embedded test data, or GitHub Pages artifacts.
- Require HTTPS, secure HttpOnly cookies or similarly secure server-managed sessions, short idle expiry, CSRF protection for cookie-backed write endpoints, same-site controls, and restricted CORS origin list.
- Authorization is server-side on read, write, export, restore, archive, and history APIs; hiding a UI button is insufficient.
- Content Security Policy, safe output encoding, input length limits, denial-of-service safeguards, and server-side validation for unexpected JSON fields.
- CSV formula injection: escape/sanitize user-controlled cell content beginning with spreadsheet formula triggers (=, +, -, @; account for leading control characters). Define a documented spreadsheet-safe output policy that preserves underlying source data unchanged.
- Responses containing private data: `Cache-Control: no-store`, no public CDN caching; service workers must not persist private JSON.
- No telemetry containing real names, addresses, export content, personal notes, or raw request bodies. Make auth failures and changes auditable via IDs and outcome categories.
- Automated tests use synthetic names and addresses only.
- Backups: Git history plus periodic private exports of canonical state; verify restore and access controls. Privacy-sensitive backups remain private.
- All saved historical lists contain PII snapshots and must be kept in the same protected repository, not the public app.

## A-04: Minimal API contract (proposed)

| Method / path | Behavior |
| --- | --- |
| `GET /api/session` | Current authenticated owner summary; no private data on failed auth. |
| `GET /api/contacts?status=&search=&page=` | Filtered contact list and revision. |
| `GET /api/contacts/{id}` | Single contact, nested people, provenance flags, version. |
| `POST /api/contacts` | Create with unique ID, validate Guest 1, return version. |
| `PUT /api/contacts/{id}` | Update with expected version, validate fields; 409 on conflict. |
| `POST /api/contacts/{id}/soft-delete` | Mark inactive with server actor/time and reason. |
| `POST /api/contacts/{id}/restore` | Restore while retaining Git audit history. |
| `GET /api/holiday-lists` | Summaries of saved lists including historical versions. |
| `POST /api/holiday-lists` | Create saved draft and initial snapshot. |
| `GET /api/holiday-lists/{id}` | Read stored list values and revision. |
| `PUT /api/holiday-lists/{id}/draft` | Modify a draft with expected version. |
| `POST /api/holiday-lists/{id}/finalize` | Immutable finalized version; idempotent. |
| `POST /api/holiday-lists/{id}/revisions` | Create a new editable revision from frozen list. |
| `POST /api/holiday-lists/{id}/archive` | Hide from normal index without physical removal. |
| `GET /api/holiday-lists/{id}/export.csv?mode=` | Authenticated CSV download, current saved snapshot only. |

API payload fields use explicit nullable values; JSON schema validation is mandatory. Return structured error codes such as `CONTACT_DUPLICATE_CANDIDATE`, `MISSING_REQUIRED_GUEST_1`, `STALE_REVISION`, `FINALIZED_LIST_READ_ONLY`, `MAIL_FIELDS_INCOMPLETE`, and `UNAUTHORIZED`. A duplicate candidate is a **warning** rather than an automatic overwrite.

## A-05: Persistence authority and rollback

- No localStorage as the authority for contacts or saved lists. Draft unsaved UI edits may be held in memory, but confirmed Save requires server persistence.
- Use stable IDs for records across years; never derive IDs solely from names, ZIP codes, or array position.
- Restoring a deleted contact does not retroactively change already finalized lists.
- Load and write `contacts-v2.json` as schema 2.0.0 only. The reviewed migration from 1.1.0 has already produced v2 and retained the original imports. Protect the v1.1.0 files as read-only and reject unrecognized future schemas instead of dropping their fields.
- All existing names, raw source values, source IDs, and null/missing fields survive a read-modify-write round trip. Guest 1 and Guest 2 are IDs pointing into the contact's single nested people array. The export and saved-list snapshot must resolve guest names from the saved people snapshot, not from live contacts.

## A-06: Before go-live

- Confirm owner authentication provider and backend hosting. GitHub OAuth sign-in may be practical; Fides can be used if its production-ready integration is verified, but do not assume an unavailable system is working.
- Confirm GitHub App installation rights and repo scope, test staging without real data, then verify live read-only access before enabling mutation.
- If GitHub App write permissions cannot be safely granted, ship a **read-only** application until this is solved; never downgrade to a frontend-held token.
