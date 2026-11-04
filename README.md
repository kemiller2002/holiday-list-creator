# Holiday List Creator

Private-data holiday card manager. This repository contains **application code and requirements only**. Production names and mailing addresses live in the private [holiday-list](https://github.com/kemiller2002/holiday-list) repository and must never be committed here, embedded in static WebAssembly assets, GitHub Pages, screenshots, fixtures, build logs, or deployment artifacts.

## Goal

Create and preserve holiday mailing lists from a reusable contact directory; add/edit/soft-delete/restore contacts; enter **Guest 1 and Guest 2** independently; select households; review missing mailing information; save draft and finalized historical lists; export a selected saved list as CSV for printing and mail merge.

## Implementation baseline

- **F# end-to-end**. Limen for the WebAssembly client, Forma for reusable UI components, Folio only if print formatting is required. No Bolero by default.
- Thin authenticated F# API mediates all access to the private `holiday-list` repository. Never access private GitHub data directly from an unauthenticated public client.
- Initial persistence: JSON in private `holiday-list` with authenticated, revision-checked writes and durable history. A managed database is a *later option*, not a prerequisite for this small mailing-list application.
- No Kubernetes, Terraform, or a new custom framework. Prefer a simple monolith and existing Echelon packages when their current versions and APIs have been verified.
- Unit tests, contract tests, integration tests, and browser smoke tests are required.

## Requirements

1. [Scope and user journeys](requirements/01-scope-workflows.md)
2. [Data contracts and saved-list semantics](requirements/02-data-contracts.md)
3. [UI and interaction design](requirements/03-ui-screens.md)
4. [Architecture, security, persistence](requirements/04-architecture-security.md)
5. [Acceptance and verification](requirements/05-acceptance-tests.md)
6. [Build sequence, scope, decisions](requirements/06-build-plan.md)
7. [Machine-readable requirements register](requirements/requirements.json)

**Status:** requirements authored; no implementation or tests are claimed complete. See open design decisions in the build plan. The canonical private source data was inspected at schema version `1.1.0` on 2026-11-04.

## Data separation

| Repository | Visibility on requirements authoring | Role |
| --- | --- | --- |
| `kemiller2002/holiday-list-creator` | Public | Requirements, F# app, synthetic tests, CI. No real contacts. |
| `kemiller2002/holiday-list` | Private | Canonical contacts and historical JSON lists; source provenance; address data. |

Do not enable unprotected deployment or live-data access merely because the UI builds.
