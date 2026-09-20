---
generated: '2026-09-19'
method: generated
name: Publish or update a directory with a resumable publish session
description: Publish a multi-file bundle (or update an existing Temp) through temp.md's idempotent, resumable publish-session protocol, so a retry never creates a second URL and an interrupted upload can be resumed.
api: openapi/temp-md-openapi.yml
operations: [createPublishSession, getPublishSession, uploadPublishSessionFile, finalizePublishSession]
source: >-
  Grounded in openapi/temp-md-openapi.yml (operationIds verified verbatim), https://temp.md/docs#sessions and
  https://temp.md/llms.txt. Cross-cutting rules from conventions/temp-md-conventions.yml, errors/temp-md-problem-types.yml
  and authentication/temp-md-authentication.yml. Limits from https://temp.md/limits.json.
---

# Publish or update a directory with a resumable publish session

Use this instead of the single multipart `POST /temps` when the bundle has many files, is large (up to 50 MB /
100 files), or when you need a replay-safe publish. The session path is the ONLY public REST path with an
`Idempotency-Key`; see `conventions/temp-md-conventions.yml` (idempotency.coverage: partial).

## Auth
- Creating a NEW Temp: no credential needed (anonymous), OR `Authorization: Bearer tempmd_key_...` (account API key) so the Temp is owned immediately.
- Updating an EXISTING Temp: `Authorization: Bearer <updateToken>` (scoped capability) or an account API key, and put `tempId` in the manifest.
- Steps 2-4 use the session's own `uploadToken` (publishSessionToken scheme), not your account key. See `authentication/temp-md-authentication.yml`.

## Before you start
- Check for a `.tempmd` / `.tempmd.json` record in the project root. If this work already has a Temp, UPDATE it (send `tempId`) - never create a second link for the same work.
- Compute `size` (bytes) and lowercase SHA-256 `hash` for every file. Paths must be safe relative paths (no `..`, no backslashes, no duplicates, <=512 chars, depth <=20). The main page should be `index.html`.

## Steps
1. **Create or resume the session** - `createPublishSession` (`POST /publish-sessions`) with header `Idempotency-Key: <stable value, <=128 chars>` and JSON `{ "files": [{path,size,contentType,hash}...], "tempId"?: "...", "title"?: "...", "spaMode"?: false }`.
   - `201` = new session ready; `200` = an existing finalized session for the same key (you are done).
   - Read `sessionId`, `uploadToken`, `uploads[]` (files that need bytes: each has `fileId`, `method`, `url`, `headers`) and `skipped[]` (unchanged by hash). The session expires in 3600 s (`expiresAt`).
2. **Upload each declared file** - `uploadPublishSessionFile` (`PUT /publish-sessions/{sessionId}/files/{fileId}`) with `Authorization: Bearer <uploadToken>` and exactly the bytes declared. Size or hash mismatch is rejected (`400`/`409`). Re-uploading the same bytes is harmless.
3. **Resume after an interruption** - `getPublishSession` (`GET /publish-sessions/{sessionId}`) with the upload token returns the current per-file status; upload only what is still pending. Persist `sessionId` + `uploadToken` in protected storage before uploads begin.
4. **Finalize** - `finalizePublishSession` (`POST /publish-sessions/{sessionId}/finalize`). The server verifies every object and promotes the Version atomically; until then the live link is untouched. Finalize is safe to retry. Response: `canonicalUrl`, `tempId`, `versionId`, `expiresAt`, and for a NEW Temp `updateToken`, `claimToken`, `claimLink`.
5. Record `Temp ID | URL | Update Token | Expires | Claim Link` in `.tempmd`. Give the user only the canonical URL.

## Errors (see `errors/temp-md-problem-types.yml`)
- `400 invalid_idempotency_key` - you omitted or malformed `Idempotency-Key`.
- `409` on create - same key, different manifest: use a new key for a genuinely new bundle, or resume the existing session instead.
- `410` - the session TTL (1 h) or the Temp's restore window has lapsed; create a new session (or `restoreTemp` first).
- `413` - over 10 MB/file, 50 MB/bundle, 100 files, or 128 KB manifest.
- `422` on finalize - a declared file is missing or mismatched: re-upload it, finalize again.
- `429` - read `retry_after` / `Retry-After`, wait, retry. Limits: 60 anonymous sessions/hour/IP, 120 account sessions/hour, 120 updates/hour/Temp/IP.

## Reversibility (see `conventions/temp-md-conventions.yml`)
- A successful finalize replaces the live Version and there is no rollback endpoint; take `snapshotTemp` before an update if the previous state must stay addressable.
- Nothing is promoted until finalize succeeds, so abandoning a session is free.
