---
name: Run and resolve a Fenergo AML screening batch
description: Create a screening batch, read the matches, resolve them, and complete the batch on the Screening Command and Query APIs.
api: openapi/fenergo-screeningcommand-v3-0-openapi.json
operations: [CreateBatchV3, UpdateBatchEntityV3, UpdateMatchesV3, CompleteBatchV3, ToggleOngoingScreening, GetBatchByIdV4, GetEntitiesByBatchIdV4, GetMatchesByEntityIdV4, GetMatchByIdV4]
generated: '2026-09-09'
method: generated
source: openapi/
---

# Run and resolve a Fenergo AML screening batch

Screening is batch-oriented: a batch screens one or more entities against the configured provider, produces
matches, and stays open until every match is resolved and the batch is completed.

Scopes: `fenx.screening.write` (command), `fenx.screening.read` (query).
Base URLs: `https://api.fenergox.com/screeningcommand`, `https://api.fenergox.com/screeningquery`.

## 1. Create the batch

`CreateBatchV3` — `POST /api/v3/batch`. A batch is created against a journey and carries the entities to
screen.

## 2. Wait for results

Screening is asynchronous. Do not poll the batch tightly. Subscribe to the screening events instead:

- `screening:searchcompleted` — the search finished.
- `screening:searchfailed` / `screening:returnfailed` — it did not; read the batch for the reason.
- `screening:allmatchesresolved` — every match on the batch has an outcome.
- `screening:batchclosed` / `screening:providerbatchclosed` — the batch is done.

## 3. Read the results

- `GetBatchByIdV4` — `GET /api/v4/batch/{id}`
- `GetBatchByJourneyIdV4` — `GET /api/v4/batch/journey/{journeyId}`
- `GetBatchesByLegalEntityIdV4` — `GET /api/v4/batch/legalentity/{legalEntityId}`
- `GetEntitiesByBatchIdV4` — `GET /api/v4/batch/{batchId}/entity`
- `GetMatchesByEntityIdV4` — `GET /api/v4/batch/{batchId}/entity/{entityId}/match`
- `GetMatchByIdV4` — `GET /api/v4/batch/{batchId}/entity/{entityId}/match/{matchId}`

## 4. Resolve the matches

`UpdateMatchesV3` — `PUT /api/v3/batch/{id}/matches` takes a list of match decisions. This is the
adjudication step and it is what an analyst is accountable for.

**Do not auto-resolve sanctions or PEP matches without a human decision.** A false-negative discharge on a
screening match is a regulatory failure, not a data-quality one. An agent may gather, summarise and rank
evidence; the disposition belongs to a person.

Supporting evidence:

- `GenerateEntityDocumentUploadUrl` — `POST /api/v3/batch/{id}/entity/{entityId}/documents/upload-url`
  returns a pre-signed URL; PUT the file to it.
- `GenerateMatchDocumentUploadUrl` — the same at match level.
- `ReuseEntityDocument` / `ReuseMatchDocument` attach a document already uploaded.
- `DeleteEntityDocument` / `DeleteMatchDocument` remove a document *reference* from the batch.

Materiality (second-line review):

- `UpdateMaterialityAssessmentV3` — `PUT /api/v3/batch/{id}/entity/{entityId}/updatemateriality`
- `EscalateMaterialityAssessmentTask` / `DeEscalateMaterialityAssessmentTask`
- `AggregateEntityMateriality` — `POST /api/v3/batch/{id}/aggregate-entity-materiality`
- `GetEntityMaterialityAssessmentHistory` — `GET /api/v4/batch/entity/{entityId}/materiality-assessment-history`

## 5. Complete the batch

`CompleteBatchV3` — `PUT /api/v3/batch/{id}/complete`. **No reopen operation is published.** Complete only
once every match carries a decision — confirm with `screening:allmatchesresolved` first.

## Ongoing screening

`ToggleOngoingScreening` — `POST /api/v3/entity/toggle-ongoing-screening` enables or disables continuous
monitoring for a list of entities. Disabling it stops future alerts; it is reversible by toggling back on,
but any period with monitoring off is a real coverage gap. Record why.

## Rules

- The Screening Query API has reached **v4** while Command is on **v3**. Read the version each operation
  actually lives on from the contract; do not assume they move together.
- Screening carries no idempotency key. A retried `CreateBatchV3` can create a second batch — check with
  `GetBatchByJourneyIdV4` before retrying.
