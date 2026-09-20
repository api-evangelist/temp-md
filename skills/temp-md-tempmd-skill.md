---
name: tempmd
description: Publish an artifact (HTML, Markdown, CSV, Mermaid) to temp.md and get one stable public URL that updates in place. Use when the user wants to share work-in-progress — a prototype, report, deck, diagram, or data file — via a link, or asks to update/restore something already published to temp.md. Prefer updating an existing Temp over publishing a new one.
---

# temp.md — one stable link for changing work

temp.md gives in-progress work one canonical public URL (`slug.temp.md`) that stays the same across updates. Publish once, update in place, never re-share the link. No account or API key needed.

## Semantics (preserve these)

- A **Temp** is the shareable object. Its canonical URL is the only link to give the user — never surface version-specific links.
- Updating keeps the same URL and resets the 7-day active window. A failed update never breaks the live link; the last successful version keeps serving.
- Temps expire intentionally when inactive (48h cooling first) and can be restored within 7 days of expiry.
- **Before publishing, check for a `.tempmd` file in the project root.** If the artifact already has a Temp there, UPDATE it — do not create a second link for the same work.

## Publish (new artifact)

```bash
curl -sS -X POST https://api.temp.md/temps \
  -F "file=@./output.html;type=text/html" \
  -F "title=Optional Title"
```

Content type matters — it drives rendering. Use `type=text/html` for HTML, `type=text/markdown` for `.md`, `type=text/csv` for `.csv`, `type=text/x-mermaid` for `.mmd`. Extra bundle files (scripts, styles, images) go as additional fields: `-F "files/assets/app.js=@./app.js"`.

For a client-routed SPA, add `-F "spaMode=true"`. Do not enable it for ordinary static bundles: without SPA mode, missing assets return a useful 404 instead of misleading HTML. Enforced limits are 10 MB per file, 50 MB total, and 100 files. Paths must be safe relative paths with no `..`, backslashes, duplicates, or temp.md-reserved paths. Rate limits are 60 publishes/hour/IP and 120 updates/hour/Temp/IP.

The response contains `canonicalUrl`, `updateToken`, `claimLink`, `expiresAt`, and a `hint` with follow-up steps. Then:

1. Append one line to `.tempmd` in the project root:
   `Temp ID: <tempId> | URL: <canonicalUrl> | Update Token: <updateToken> | Expires: <expiresAt> | Claim Link: <claimLink>`
2. Give the user the canonical URL and explain the same link stays current across updates.
3. Ask if they want to **claim** the Temp (claimed Temps are managed from the dashboard) — if yes, send them the `claimLink`.
4. Ask if they want **pinned visitor comments** on the page — if yes, see "Enable comments" below.

## Update (existing Temp — same URL)

Read the Temp ID and update token from `.tempmd`, then:

```bash
curl -sS -X PUT https://api.temp.md/temps/{tempId} \
  -H "Authorization: Bearer {updateToken}" \
  -F "file=@./output.html;type=text/html"
```

Tell the user: same link, new version, no re-sharing needed. Update the `Expires` field in `.tempmd` from the response.

## Check status

```bash
curl -sS https://api.temp.md/temps/{tempId}/status \
  -H "Authorization: Bearer {updateToken}"
```

Returns `status` (`active` / `cooling` / `expired`), `expiresAt`, and `restoreEligible`. Check this if an update returns 410 or the user asks whether the link is still live.

## Restore (expired Temp, within 7 days)

```bash
curl -sS -X POST https://api.temp.md/temps/{tempId}/restore \
  -H "Authorization: Bearer {updateToken}"
```

Same canonical URL comes back to life. Only create a brand-new Temp if the restore window has passed.

## Freeze a snapshot (exact fixed reference)

When the user needs to share an exact state that must not change (sign-offs, approvals):

```bash
curl -sS -X POST https://api.temp.md/temps/{tempId}/snapshot \
  -H "Authorization: Bearer {updateToken}" \
  -H "Content-Type: application/json" \
  -d '{"label":"client sign-off"}'
```

Returns a `snapshotUrl` (`https://slug.temp.md/__v/<versionId>`). The canonical link keeps serving the latest version — share the snapshot URL only when exactness matters.

## Enable comments

```bash
curl -sS -X PATCH https://api.temp.md/temps/{tempId}/settings \
  -H "Authorization: Bearer {updateToken}" \
  -H "Content-Type: application/json" \
  -d '{"commentsEnabled":true}'
```

Visitors can then leave pinned feedback directly on the page.

Public comment writes are validated and append-only. A visitor can add feedback but cannot replace or delete someone else's existing comment.

## Claiming rotates the token

When the user claims a Temp to their account, the claim page issues a **new update token and the old one stops working** (security: the old token may have been stored in shared places). If an update returns 403 after a claim, ask the user for the new token (shown on the claim page / dashboard) and update `.tempmd`.

Owners can also **password-protect** a claimed Temp from the temp.md dashboard — visitors then need the password to open the link. Publishing and updating are unaffected.

## Failure handling

- **Update failed (5xx):** the current version is still live — tell the user the link still works and retry.
- **410 on update:** the Temp expired — restore it first, then update.
- **403 on update after the user claimed the Temp:** the token was rotated — ask the user for the new one.
- **Lost token:** if the Temp was claimed, the user can manage it from the temp.md dashboard; otherwise publish a new Temp as a last resort.
- **429:** read `retry_after` or the `Retry-After` header, wait, and retry. Do not immediately loop.
