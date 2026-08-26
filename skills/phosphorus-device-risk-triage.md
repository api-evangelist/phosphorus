---
name: phosphorus-device-risk-triage
description: >-
  Find the riskiest xIoT devices in a Phosphorus tenant and assemble the full evidence for one of
  them — alerts, firmware, credentials, certificates, logs and scan history — before any remediation
  decision is made.
generated: '2026-08-26'
method: generated
source: openapi/phosphorus-api-openapi.yml
api: Phosphorus API
base_url: https://{tenant}.phosphorus.io
auth: X-API-KEY header
operations:
  - searchv2
  - getAlertsv2
  - getDeviceByUUIDv2
  - getDeviceAlertsByUUIDv2
  - getDeviceFirmwareByUUIDv2
  - getDeviceCredentialByUUIDv2
  - getDeviceCertificateByUUIDv2
  - getDeviceLogsByUUIDv2
  - getDeviceScansByUUIDv2
  - getSitev2
read_only: true
---

# Triage xIoT device risk in Phosphorus

Every operation in this skill is a GET. Nothing here changes state, so it is safe to run against a
production tenant.

## Before you start

- The base URL is the customer's own instance: `https://{tenant}.phosphorus.io`. There is no shared
  host — ask which tenant you are working in and never assume one.
- Send `X-API-KEY: <key>` on every request. There is no OAuth and there are no scopes, so the key
  carries whatever access the tenant granted it — you cannot narrow it per call.
- The contract documents only `200` and `400 Invalid input`. A `401` or `404` will happen and is not
  described anywhere, so handle non-200 by surfacing the raw status; do not try to parse an error
  body shape, because none is published.

## Steps

1. **Find candidate devices.** Call `searchv2` (`GET /api/v2/search`) with `q` written in the
   Phosphorus Search Query Syntax. The response carries `status`, `total`, `devices` and `facets` —
   use `facets` to see how the fleet distributes before drilling in. The query syntax is not
   published publicly, so if you do not already know it, work from the device attributes in the
   `device` schema (`manufacturer`, `model`, `type`, `vulnerable`, `hasDefaultCredentials`,
   `outOfDateFirmware`, `discontinued`, `invalidCredentials`) rather than inventing operators.

2. **Or start from what is already alerting.** Call `getAlertsv2` (`GET /api/v2/alerts`). With no
   parameters it returns active alerts from the last 24 hours — that default is stated in the
   contract, so you can rely on it. Widen with `startTime` and `endTime` (RFC 3339), narrow with
   `filter` and `severity`, and set `includeLostDevices` when you need devices the platform has
   stopped seeing.

3. **Read the device.** Call `getDeviceByUUIDv2` (`GET /api/v2/device/{uuid}`). The risk booleans on
   the `device` object are the fastest read: `vulnerable`, `hasDefaultCredentials`,
   `invalidCredentials`, `outOfDateFirmware`, `credentialsRequired`, `discontinued`, `manageable`,
   `canChangePassword`, `canUpdate`. Also note `siteId`, `lastInterrogation` and
   `lastInterrogationResult` — a stale interrogation means the risk flags are stale too.

   The `interrogate` query parameter on this operation triggers a live interrogation of the device.
   That is not a read: it puts traffic on an operational network. Leave it off unless a human has
   asked for it explicitly.

4. **Gather the evidence.** For the chosen uuid, call in whatever order suits the question:
   - `getDeviceAlertsByUUIDv2` — paginate with `limit`/`offset`, narrow with `name`, `subtype`,
     `severity`.
   - `getDeviceFirmwareByUUIDv2` — returns `firmwareObj` entries carrying `version`, `releaseDate`,
     `cves`, `severity`, `current`, `upgrade`, `downgrade`, `canUpdate` and checksums. The `cves`
     field is a raw CVE-ID list, not a CSAF or VEX advisory, so severity judgement is yours.
   - `getDeviceCredentialByUUIDv2` — what credentials exist for the device.
   - `getDeviceCertificateByUUIDv2` — certificates held by the device.
   - `getDeviceLogsByUUIDv2` — `logEntry` records with `message`, `created`, `device`, `user`.
   - `getDeviceScansByUUIDv2` — scan history.

5. **Place it in the estate.** Call `getSitev2` (`GET /api/v2/sites/{id}`) with the device's `siteId`
   to see whether the site is `online`, `acceptingScans`, and what `smVersion` it runs. A site that
   is offline or refusing scans explains a stale device record better than any device-level field.

6. **Check what is out of scope.** Call `getExcludedDevicesv3` (`GET /api/v3/devices/excluded`) to
   confirm the device is not deliberately excluded from management. Excluded devices are queryable by
   date and `jobId`.

## Pagination

`limit` and `offset` only. No response carries a `total`, `next` or cursor except `searchv2`, so on
every other list you must keep incrementing `offset` until you get a short page. Do not assume a
default page size — none is documented.
