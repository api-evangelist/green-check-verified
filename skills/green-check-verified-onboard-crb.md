---
name: green-check-verified-onboard-crb
description: Onboard a cannabis-related business (CRB) onto Green Check on behalf of a service provider, either by inviting the business to complete a full due-diligence application or by connecting its point-of-sale system directly.
api: Green Check Access
generated: '2026-08-22'
method: generated
source: openapi/green-check-verified-access-openapi.yaml + https://developer.greencheckverified.com/guides/next-steps + https://developer.greencheckverified.com/guides/create-crb-connect-pos
operations:
  - get-token
  - get-service-provider-onboarding-templates
  - get-crbs-by-ein
  - get-service-provider-pos-credentials-schema
  - create-crb-for-service-provider
  - connect-crb-by-id
  - get-service-provider-crbs
  - get-crb-info
---

# Onboard a CRB onto Green Check

Creates a Cannabis Related Business under your service provider account. Two published paths, and
choosing between them is the real decision — they produce different compliance coverage.

## Before you start

- You need `client_id` / `client_secret` issued by Green Check. There is no self-service signup.
- You need your `sp_id`. Every call is scoped to it.
- Work against `https://sandbox-api.greencheckverified.com` first. Production is
  `https://prod-api.greencheckverified.com`.
- **Confirm the data-sharing language in your contract with the CRB before connecting them.** Green
  Check's docs state that its own terms-of-service acceptance is enforced only through the UI invite
  path — on the API paths, that consent is your responsibility.

## Step 1 — Get a token (`get-token`)

`POST /auth/token` with `{client_id, client_secret, grant_type: "client_credentials"}`.

Returns `access_token`, `scope[]`, `expires_at`. Send `Authorization: Bearer <access_token>` on every
later call. **Tokens live 3600 seconds** — check `expires_at` (unix) and re-authenticate before it
passes, or you will get 401s that look like permission failures.

## Step 2 — Check the business is not already here (`get-crbs-by-ein`)

`GET /service-providers/{sp_id}/ein-search`

Do this first. **There is no idempotency key on CRB creation and no delete operation.** The only
duplicate guard is a `409 Conflict` on an exact organization-name match, which will not catch
"Example Dispensary LLC" against "Example Dispensary". An EIN check is your real guard.

If the CRB exists already, use `connect-crb-by-id` (`POST /service-providers/{sp_id}/connect-crb`)
instead of creating a new one.

## Step 3 — Choose the path

| | Invite path | Connect-POS path |
|---|---|---|
| Operation | `create-crb-for-service-provider` with `users[]` | `create-crb-for-service-provider` with `pos_info` |
| CRB effort | Completes a full application | Self-service, low |
| License verification | Yes | **No** |
| Negative-news alerts | Yes | **No** |
| Due-diligence documents | Yes | **No** |

You must supply **either** `pos_info` **or** `users[]` with at least one entry. You may supply both.

## Step 4a — Invite path

1. `get-service-provider-onboarding-templates` — `GET /service-providers/{sp_id}/onboarding-templates`.
   Note the `template_id` you want.
2. `create-crb-for-service-provider` — `POST /service-providers/{sp_id}/crbs`:

```json
{
  "org": { "name": "Example Dispensary", "state": "CO", "business_type": "retail",
           "template_id": "<template_id>" },
  "users": [ { "firstName": "Jane", "lastName": "Doe", "email": "jane@example.com" } ],
  "options": { "onboarding_required": true }
}
```

This **sends an email to a named human**. It cannot be recalled. Do not retry this call blindly on a
timeout — re-run the EIN search first and check `get-service-provider-crbs` for the record.

## Step 4b — Connect-POS path

1. `get-service-provider-pos-credentials-schema` — `GET /service-providers/{sp_id}/pos-credentials-schema`.
   **Fetch this at runtime; do not hardcode or cache provider fields.** The schema changes as Green
   Check adds integrations, and it is what tells you which fields each POS requires.
2. Present the provider list, collect credentials, then `create-crb-for-service-provider` with
   `pos_info: {pos_type, pos_credentials, location_id}`.

Green Check makes a live test call to the POS during creation, so credential errors surface
synchronously as `422`.

## Step 5 — Confirm

`get-service-provider-crbs` then `get-crb-info`. Watch `due_diligence_status` move through
`gcv_pending` → `gcv_in_progress` → `bank_awaiting_review` → `bank_review_in_progress` →
`bank_approved`.

**There is no webhook.** Green Check sends email notifications, and its own docs say to treat those
as a convenience and not as a workflow trigger. Poll this status field.

## Errors

| Status | Cause | Do |
|---|---|---|
| 401 | Expired/invalid token | Re-run step 1 |
| 403 | Wrong `sp_id`, CRB not connected, or token scope too narrow | Check `scope[]` from step 1 |
| 409 | Organization name already exists | Search by EIN; connect instead of create |
| 422 | Validation failure | Read `details` — it names each failing field |

`422` also carries domain rules: `business_type` must match `pos_type` (a wholesale org needs a
wholesale POS), and an invalid or missing `location_id` returns the list of valid options in the
response body — use it rather than guessing.

## Sandbox rehearsal

Green Check publishes deterministic simulation values. With `pos_type: "Greenbits"`,
`username: "testing"`: password `testing` simulates success, `invalid_testing` simulates rejection.
`location_id: "testing"` simulates a successful connection with no real POS call; `invalidLocation`
or an empty value returns the available-options error. Rehearse every branch here before production.
