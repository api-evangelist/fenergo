---
name: Create, update and verify a Fenergo entity draft
description: Run the draft/verify write path on the Entity Data Command API inside a journey, including the concurrency and rejection rules.
api: openapi/fenergo-entitydatacommand-v3-0-openapi.json
operations: [CreateEntityDraft, UpdateEntityDraftV3, EntityDraftConflicts, VerifyEntityDraft, RejectEntityDraft, GetEntityDraftById, GetEntityDraftProposedChanges]
generated: '2026-09-09'
method: generated
source: openapi/ + https://docs.fenergox.com/developer-hub/api-overview/api-principles-and-patterns
---

# Create, update and verify a Fenergo entity draft

Fen-X does not let you patch a verified legal entity directly. Changes are staged as a **draft** attached to
a journey, then **verified** (published) or **rejected**. This is the platform's core write pattern.

Token scopes: `fenx.entitydata.write` (plus `fenx.entitydata.read` to read back, `fenx.journey.write` if you
also create the journey). Headers: `Authorization: Bearer …` and `x-tenant-id: …`.

Base URLs: `https://api.fenergox.com/entitydatacommand` and `https://api.fenergox.com/entitydataquery`.

## 1. Create the draft

`CreateEntityDraft` — `POST /api/v3/entity/{entityId}/draft`

The draft belongs to a journey. Create or identify the journey first with `CreateJourney`
(`POST /api/journey-instance` on `journeycommand`, scope `fenx.journey.write`).

## 2. Update it

`UpdateEntityDraftV3` — `PUT /api/v3/entity/{entityId}/draft/{id}`

Related operations on the same draft:

- `UpdateEntityDraftRisk` — `PUT /api/v3/entity/{entityId}/draft/{id}/risk`
- `UpdateEntityDraftRole` — `PUT /api/v3/entity/{entityId}/draft/{id}/role`
- `UpdateEntityDraftAccessLayers` — `PUT /api/v3/entity/{entityId}/draft/{id}/access-layers`
- `UpdateSharedDataTemplate` / `ClearSharedDataTemplate` — attach or clear a shared data template.

## 3. Handle conflicts

If another writer changed the same aggregate you get **HTTP 409 "Conflict saving changes in expected
version"**. That is optimistic concurrency on an event-sourced aggregate.

**There is no idempotency key on this API.** A timed-out `POST /api/v3/entity/{entityId}/draft` may or may
not have created a draft. Before retrying a create, call `GetEntityDraftsIds`
(`GET /api/v2/entity/{entityId}/drafts-ids`) or `GetEntityDraftById` and check — do not blind-retry a write.

On a 409 for an update: re-read with `GetEntityDraftById`, re-apply your change to the current version, and
resend. For structured conflicts use `EntityDraftConflicts` — `PUT /api/v3/entity/{entityId}/draft/{id}/conflicts`.

## 4. Review the diff

`GetEntityDraftProposedChanges` — `GET /api/v2/entity/proposedchanges/draft/{entityDraftId}` returns the
delta between the draft and the verified record. Show this before verifying.

## 5. Verify or reject

- `VerifyEntityDraft` — `PUT /api/v3/entity/{entityId}/draft/{id}/verify` publishes the draft.
- `RejectEntityDraft` — `PUT /api/v3/entity/{entityId}/draft/{id}/reject` discards it.

`RejectEntityDraft` is the reversal path for an unverified change and it works at any time before verify.
**Once a draft is verified there is no published undo operation and no stated reversal window** — the way
back is a new draft carrying the correction. Treat verify as the point of no return.

## 6. Confirm

Verification is asynchronous. Do not immediately re-read the Query API and assume failure if the change is
absent. Wait for `entitydata:draftverified` then `entitydata:dataPublished` on your webhook or polling feed
(see `asyncapi/fenergo-event-notifications-webhooks.yml`), then read with `GetEntityById`.
