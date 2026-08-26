---
name: phosphorus-vault-secret-handling
description: >-
  Store, read, list and rotate credentials in the Phosphorus Vault, with the specific care the API's
  missing delete operation and missing version-restore path demand.
generated: '2026-08-26'
method: generated
source: openapi/phosphorus-api-openapi.yml
api: Phosphorus API
base_url: https://{tenant}.phosphorus.io
auth: X-API-KEY header
operations:
  - listSecretsv3
  - getSecretv3
  - createSecretv3
  - updateSecretv3
  - updateNativeVaultToDefaultv3
  - getProvidersv3
  - listProviderCredentialsv3
read_only: false
---

# Handle secrets in the Phosphorus Vault

This skill touches live device credentials on an operational network. Read the two warnings before
the steps; they are the whole reason this skill exists as a separate document.

## Two irreversible edges

1. **There is no delete.** `createSecretv3` has no counterpart operation in the published contract —
   no `DELETE /api/v3/vault/{id}`. A secret you create through this API cannot be removed through this
   API. Confirm with `listSecretsv3` that the secret does not already exist before creating one, and
   never create speculatively.
2. **A rotation cannot be rolled back through the API.** `updateSecretv3` is documented as updating a
   secret *by creating a new version*, so history exists server-side — but the contract publishes no
   operation to read or restore a prior version, and states no retention window. If you rotate a
   device credential to a wrong value, the API gives you no way back. Capture the value you are
   replacing through whatever channel you legitimately hold it in, out of band, before you write.

There is also no idempotency key, so a `createSecretv3` that times out must never be blindly retried
— re-list first.

## Steps

1. **See what is there.** `listSecretsv3` (`GET /api/v3/vault`) returns `name` and `externalId` only —
   never secret material. `externalId` is the handle you carry forward.

2. **Read one.** `getSecretv3` (`GET /api/v3/vault/{id}`). Treat the response as secret material:
   do not log it, do not echo it into a transcript, and do not place it in an error message.

3. **Create.** `createSecretv3` (`POST /api/v3/vault`) with a `createSecretBody` — `name`, `login`,
   `password`, `serialNumber`. Returns a `createSecretResponse`. Record the identifier it returns;
   with no delete operation available, an unrecorded secret is an orphan you cannot clean up.

4. **Rotate.** `updateSecretv3` (`PUT /api/v3/vault/{id}`) with an `updateSecretBody`. This creates a
   new version. Get a human decision before this call when the credential is bound to a device that
   is currently in service.

5. **Set the tenant default vault — carefully.** `updateNativeVaultToDefaultv3` (`PUT /api/v3/vault`)
   with an `updateVaultToDefaultBody` makes the native Phosphorus Vault the tenant default. This is a
   tenant-wide configuration change with no published inverse operation. It changes where the whole
   platform looks for credentials, so it is not a step to take as part of routine automation.

6. **Check the alternative first.** If an external credential store is already registered, its
   credentials may be assignable without creating anything new: `getProvidersv3`
   (`GET /api/v3/providers`) lists registered providers and `listProviderCredentialsv3`
   (`GET /api/v3/providers/{id}/credentials`) returns the credentials that provider can assign. Using
   an existing provider credential avoids an undeletable Vault entry entirely.

## Handling secret material

Transport is HTTPS with the key in an `X-API-KEY` header. Beyond that the API offers no envelope
encryption, no write-only field marking and no redaction guidance in the contract. Assume every value
you send or receive is plaintext at your end of the connection and handle it accordingly.
