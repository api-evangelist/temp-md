---
generated: '2026-09-19'
method: generated
name: Account-owned publishing and update-token recovery
description: Create a temp.md account and a named API key so published Temps are owned (no expiry, manageable from the dashboard), and recover a lost scoped update token without changing the public URL.
api: openapi/temp-md-openapi.yml
operations: [signup, login, createApiKey, listApiKeys, createPublishSession, finalizePublishSession, rotateUpdateToken, revokeApiKey]
source: >-
  Grounded in openapi/temp-md-openapi.yml (operationIds verified verbatim), https://temp.md/docs#agents,
  https://temp.md/llms.txt and https://temp.md/.well-known/agent.json (authentication.accountApiKeys). Rules from
  conventions/, errors/ and authentication/.
---

# Account-owned publishing and update-token recovery

Anonymous Temps expire after 7 idle days and their only credential is the scoped update token. Owning them
through an account removes the expiry (privacy policy: "Claimed files are retained until the owner deletes
them") and adds a recovery path when the token is lost. Accounts are optional; do not create one unless the user
wants persistence or recovery.

## Auth
- `signup` / `login` are anonymous. `login` returns `LoginResult.token` (account JWT).
- Everything under `/me/...` takes `Authorization: Bearer <JWT or tempmd_key_...>` (accountBearer). See `authentication/temp-md-authentication.yml`.
- Rate limits: 10 signups and 30 logins per hour per IP; 60 key creations per hour per account (`rate-limits/temp-md-rate-limits.yml`).

## Steps - set up a key once
1. **Create the account** - `signup` (`POST /auth/signup`, body per `SignupInput`; `409` if the email exists) then **sign in** - `login` (`POST /auth/login`, `{ "email", "password" }`) and keep the returned `token` in memory only.
2. **Create a named API key** - `createApiKey` (`POST /me/api-keys`, `{ "name": "CI publisher" }`, max 64 chars). The secret (`tempmd_key_...`) is returned ONCE and stored hashed server-side - put it in protected storage immediately. Max 20 active keys. `409` if the name is taken.
3. **Verify** - `listApiKeys` (`GET /me/api-keys`) shows `id`, `name`, `suffix`, `status`, `createdAt`, `lastUsedAt` - never the secret.

## Steps - publish as the account
4. Publish with the key on the connection: REST `createPublishSession` + `finalizePublishSession` with `Authorization: Bearer tempmd_key_...` ("New Temps created with an account key are owned immediately"), or configure the same header on the MCP connection (`https://api.temp.md/mcp`) / run `npx @tempmd/cli login`. Owned Temps have `ownershipState: claimed` in `getTempStatus`.

## Steps - recover a lost update token
5. **Rotate** - `rotateUpdateToken` (`POST /me/temps/{tempId}/update-token`, account credential). This atomically invalidates EVERY prior scoped update token for that Temp and returns exactly one replacement; the canonical URL does not change. Limit 20 rotations/hour/account/Temp. Update the `.tempmd` record. (Same operation behind MCP `recover_update_token` and CLI `tempmd recover`.)
6. If the Temp was never claimed and the token is lost, there is no recovery: publish a new Temp (provider guidance in skill.md).

## Hygiene
- **Revoke** a leaked key immediately - `revokeApiKey` (`DELETE /me/api-keys/{keyId}`); revocation is irreversible and takes effect at once. Create a new key rather than trying to un-revoke.
- Claiming an anonymous Temp into the account (dashboard / claim link) ROTATES its update token; a `403` on the next update means you hold the old one.
- Never put account keys, update tokens or upload tokens in URLs, logs, analytics or renderer output (docs#browser-integrations).

## Errors (see `errors/temp-md-problem-types.yml`)
- `401 unauthorized` - missing/invalid bearer. `403` - valid credential, wrong Temp (rotated token). `404` - unknown tempId/keyId. `429` - wait `retry_after`.
