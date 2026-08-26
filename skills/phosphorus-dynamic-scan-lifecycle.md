---
name: phosphorus-dynamic-scan-lifecycle
description: >-
  Create, tune, enable, disable and retire a Phosphorus dynamic scan configuration safely, given that
  the API offers no idempotency key and no undo for an update.
generated: '2026-08-26'
method: generated
source: openapi/phosphorus-api-openapi.yml
api: Phosphorus API
base_url: https://{tenant}.phosphorus.io
auth: X-API-KEY header
operations:
  - listDynamicScanConfigsv3
  - getDynamicScanConfigv3
  - createDynamicScanConfigv3
  - updateDynamicScanConfigv3
  - enableDynamicScanConfigv3
  - disableDynamicScanConfigv3
  - deleteDynamicScanConfigv3
  - submitDynamicScanDatav3
  - getProvidersv3
  - getSitev2
read_only: false
---

# Manage a Phosphorus dynamic scan configuration

This skill writes. A dynamic scan puts traffic on an operational IoT/OT network, so treat every step
here as consequential and get a human decision before the first write.

## The two rules that matter most

1. **There is no idempotency key.** `createDynamicScanConfigv3` is a plain POST with no
   `Idempotency-Key` header and no dedupe parameter. If a create times out, DO NOT retry it. Call
   `listDynamicScanConfigsv3` first and look for a config with your intended `name`; only create
   again if it is genuinely absent.
2. **An update cannot be undone.** `updateDynamicScanConfigv3` overwrites the config and the API
   publishes no way to read a prior version. Read the current config with `getDynamicScanConfigv3`
   and keep the response before you write over it — that copy is the only rollback you will have.

## Steps

1. **Survey.** `listDynamicScanConfigsv3` (`GET /api/v3/dynamic-scans`) returns every config with
   `id`, `name`, `description`, `enabled`, `siteId`/`siteName`, `providerId`/`providerName`,
   `createdAt`, `updatedAt`, `userId`, `lastSuccessfulScan` and a `metrics` block
   (`totalIPsScanned`, `totalScanRequests`). `lastSuccessfulScan` is the field that tells you whether
   a config is actually working.

2. **Resolve the site and the provider first.** A config binds to both by id. Confirm the site with
   `getSitev2` (`GET /api/v2/sites/{id}`) — check `online` and `acceptingScans` — and confirm the
   provider with `getProvidersv3` (`GET /api/v3/providers`). Creating a config against an offline
   site produces a config that will never scan.

3. **Create.** `createDynamicScanConfigv3` (`POST /api/v3/dynamic-scans`) with a
   `CreateDynamicScanConfigBody`. Record the returned `id` immediately — it is the only handle you
   have for every later operation.

4. **Start disabled where you can.** If the create leaves the config enabled and you are not ready
   for it to run, call `disableDynamicScanConfigv3` (`POST /api/v3/dynamic-scans/disable/{id}`)
   straight away. Enable and disable are a true two-way toggle: they set a state rather than
   appending one, so they are safe to repeat and safe to reverse.

5. **Tune.** Read with `getDynamicScanConfigv3` (`GET /api/v3/dynamic-scans/{id}`), keep that
   response, then write with `updateDynamicScanConfigv3` (`PUT /api/v3/dynamic-scans/{id}`) using an
   `UpdateDynamicScanConfigBody`.

6. **Run it.** `enableDynamicScanConfigv3` (`POST /api/v3/dynamic-scans/enable/{id}`). Watch
   `metrics.totalIPsScanned`, `metrics.totalScanRequests` and `lastSuccessfulScan` on subsequent
   reads to confirm it is doing work.

7. **Feed it, if you are the data source.** `submitDynamicScanDatav3`
   (`POST /api/v3/dynamic-scans/data`) takes a `DynamicScanSubmissionData` body and returns a
   `DynamicScanSubmissionResponse`. There is no reversal for a submission — the data lands.

8. **Retire.** `deleteDynamicScanConfigv3` (`DELETE /api/v3/dynamic-scans/{id}`). This is the reversal
   of a create, but there is no restore and no soft-delete surface, so a delete is final. Prefer
   `disableDynamicScanConfigv3` when the intent is "stop it for now" rather than "remove it".

## Error handling

Every operation declares only `200` and `400 Invalid input`, and the 400 has no response body schema.
Surface the raw status and your request context; there is no error code to branch on.
