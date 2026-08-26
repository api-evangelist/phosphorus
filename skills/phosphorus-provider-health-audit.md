---
name: phosphorus-provider-health-audit
description: >-
  Audit every integration provider registered in a Phosphorus tenant, run health checks, and tell
  whether a stale device record is a Phosphorus problem or a broken upstream integration.
generated: '2026-08-26'
method: generated
source: openapi/phosphorus-api-openapi.yml
api: Phosphorus API
base_url: https://{tenant}.phosphorus.io
auth: X-API-KEY header
operations:
  - getProvidersv3
  - getProviderByIdv3
  - performHealthCheckv3
  - performAllHealthChecksv3
  - listProviderCredentialsv3
  - listDynamicScanConfigsv3
  - getSitev2
read_only: false
---

# Audit Phosphorus integration provider health

Providers are the external systems a Phosphorus tenant depends on — credential vaults, CMDBs,
scanners. When device data goes stale, the provider layer is where to look first. The health-check
operations are POSTs: they cause work upstream, but they create no persistent state you would need to
undo.

## Steps

1. **List them.** `getProvidersv3` (`GET /api/v3/providers`) returns every registered provider.

2. **Read one.** `getProviderByIdv3` (`GET /api/v3/providers/{id}`) for the detail on a specific
   provider.

3. **Check one.** `performHealthCheckv3` (`POST /api/v3/providers/{id}/health-check`) runs a health
   check against a single provider. Start here when you already suspect a specific integration.

4. **Check them all — and mind the phrasing.** `performAllHealthChecksv3`
   (`POST /api/v3/providers/health-check`) is documented as performing "a health check for every
   provider with a low concurrency". Low concurrency means this call is slow by design, not that it
   is cheap: it reaches out to every registered upstream system in turn. Run it deliberately, not on
   a tight schedule, and do not fire a second one while the first may still be running — there is no
   idempotency key and no job handle to poll.

5. **Check what a provider can hand out.** `listProviderCredentialsv3`
   (`GET /api/v3/providers/{id}/credentials`) returns the credentials that provider can assign. A
   provider that is healthy but returns nothing assignable is a different failure from one that is
   unreachable.

6. **Connect it to the scan surface.** `listDynamicScanConfigsv3` (`GET /api/v3/dynamic-scans`)
   carries `providerId` and `providerName` on each config, plus `lastSuccessfulScan`. Cross-referencing
   a failing provider against the configs bound to it tells you the blast radius. Confirm the site
   side too with `getSitev2` (`GET /api/v2/sites/{id}`) — check `online` and `acceptingScans` before
   blaming the provider.

## Interpreting the result

The contract declares only `200` and `400 Invalid input` on these operations and publishes no schema
for the health-check response, so read the returned body as-is rather than expecting a documented
status enum. Report what came back verbatim alongside the provider id and name.
