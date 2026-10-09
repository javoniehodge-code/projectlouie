# Project Louie — API contracts and database permissions

Status: Implementation specification proposal. No routes, policies, functions, or migrations have been deployed.

Companion: [data-model.md](data-model.md). Confirmed product decisions take precedence over implementation inherited from Stack.

## Architecture

Use Next.js server endpoints for public reads, creator operations, subscriptions, and CSV export. Supabase Auth identifies creators; PostgreSQL functions own multi-record transactions.

The browser never chooses owner_id or receives a privileged database key. Server endpoints derive the user from a validated authentication session. Do not authorize using a client-supplied user ID.

Routes below are proposed contracts, not claims about current code. Version them under /api/v1. Internal object IDs are UUIDs; draft/content/access revisions are decimal strings because database bigint values may exceed JavaScript's safe integer range.

## Common contract

Successful response:

```json
{
  "data": {},
  "request_id": "uuid"
}
```

Error response:

```json
{
  "error": {
    "code": "DRAFT_CONFLICT",
    "message": "This draft has newer saved changes.",
    "details": {"current_revision": "12"}
  },
  "request_id": "uuid"
}
```

| HTTP status | Code | Meaning |
| --- | --- | --- |
| 400 | INVALID_REQUEST | Malformed request |
| 401 | AUTH_REQUIRED | Missing/expired creator session |
| 404 | UNAVAILABLE | Missing, unauthorized, revoked, or deleted resource |
| 409 | DRAFT_CONFLICT | Draft revision or instance is stale |
| 409 | CONTENT_CONFLICT | Publication content changed since loading |
| 409 | ACCESS_CONFLICT | Privacy/link state changed |
| 409 | PUBLICATION_DELETED | Owner must explicitly publish a surviving draft as new |
| 409 | REQUEST_ID_REUSED | Idempotency key reused with different payload |
| 422 | CONTENT_INVALID | Draft/publication content violates rules |
| 429 | RATE_LIMITED | Request limit exceeded; include Retry-After |
| 503 | TEMPORARILY_UNAVAILABLE | Operation unavailable; retry may be possible |

Only authenticated owners receive revision/deletion details. Public resource failures use one neutral UNAVAILABLE response without exposing whether hidden content exists. Validation details name fields and limits without returning secrets.

Mutation requests use Idempotency-Key: UUID. A retry uses the same key and identical payload. Scope keys by actor and operation; store a payload fingerprint and safe response identifiers. Authenticate first, then check a completed receipt before resource preconditions, so a successful publish can be retried after consuming its draft.

Store successful outcomes atomically with writes. A failed validation/conflict does not become a successful receipt. Do not promise exactly-once delivery; the guarantee is no duplicate committed mutation for the same key.

Revision checks remain mandatory even with idempotency. A new action uses a new key. Do not silently retry conflicts as fresh writes.

Cookie-authenticated mutations require same-origin/CSRF protection. CORS must not allow arbitrary credentialed origins. Bound request size, validate JSON schemas, and rate-limit sensitive operations.

## Public reads

### GET /creators/:handle

Returns only public profile fields: id, handle, display_name, bio if retained, public avatar URL, validated external links.

No auth email, subscriber counts/list contents, private stack counts, or owner configuration.

### GET /creators/:handle/stacks?q=&cursor=&limit=

Returns paginated public published stack previews, current viewer links, and pin ordering. Initial maximum page size: 50. q searches this creator's current public publications only.

Filter ownership, live publication, and public visibility in the database before pagination/search. A public pin is ordering, not separate featured content. Cursor ordering must be deterministic, with an ID tie-breaker.

### GET /s/:token

Resolve a current nonrevoked link whose publication is live and public/unlisted. Return:
- publication_id, content_revision, current content.
- Public creator attribution and creator-page URL.
- Current viewer URL and permitted viewer actions.

Do not offer a public GET-by-publication-ID alternative that bypasses token revocation. Internal IDs in responses do not grant access.

Start with Cache-Control: no-store for token resolution and visibility-sensitive stack responses, including unavailable responses. Do not permanently redirect old tokens. Public profile collection caching is deferred until revocation/invalidation is verified.

An already-open page or downloaded image cannot be remotely erased. New requests must recheck current access; the app must not serve old protected data from a shared cache.

## Creator reads

### GET /me/home

Return own profile plus paginated management summaries for publications/drafts; default UI view is private. Do not return the full content of every stack in a large owner list.

### GET /me/stacks?filter=all|public|unlisted|private|drafts&cursor=&limit=

Return owned records only, with publication/draft IDs, titles, privacy, revisions, and edit-draft relationship. Include untitled but nonblank drafts. Deleted publications belong to historical analytics navigation, not the ordinary live stack list.

### GET /me/publications/:id

Return owned live content, effective privacy, content/access revisions, and current valid viewer URL if shareable. Private content is read through this owner route, without minting a viewer link.

### GET /me/drafts/:id

Return owned saved content, draft revision, source publication state, and current source content/access revisions when live. A surviving draft is labeled as requiring publication as a new stack.

## Creator writes

Each route invokes one transactional function. No browser sequence of independent inserts/updates is allowed.

### POST /me/stacks

Body: client_stack_id, client_draft_id, content, proposed_visibility (default public).

Require nonblank draft content. Atomically create stack + draft revision 1. Return IDs/revision. This is first nonblank save, not opening an empty editor.

### POST /me/publications/:id/edit-draft

No content payload required. Lock/check ownership and return the existing shared draft or create it from the current publication. Return draft ID/revision plus source content/access revisions. Concurrent calls return the same draft.

If publication has been deleted, do not recreate an edit draft. A previously surviving draft is accessed by draft ID.

### PATCH /me/drafts/:id

Body: expected_revision, content; proposed_visibility permitted only for standalone new drafts or surviving drafts.

Validate nonblank content and limits, lock/check the exact draft instance and revision, then save/increment revision. Return saved content/revision. Never upsert a missing draft ID.

No publication/privacy mutation. If source content has changed since draft creation, surface the conflict before replacement publication; do not rebase automatically.

A save with lost acknowledgement can be retried with its original key. The UI serializes autosaves per tab and ignores late acknowledgements older than its current revision.

### POST /me/drafts/:id/publish

Body: expected_draft_revision, visibility.

Only for a never-published stack. Validate title plus at least one body item and content limits. Create publication, active viewer link if public/unlisted, and consume draft atomically.

Return publication_id, stack_id, content_revision, access_revision, visibility, viewer_url (null if private). A source associated with a deleted publication must use publish-as-new instead.

### POST /me/publications/:id/republish

Body: draft_id, expected_draft_revision, expected_content_revision, expected_access_revision.

Validate latest saved draft and publication under lock, replace published content, and consume draft atomically. Keep publication/analytics identity and current link.

This endpoint does not accept visibility copied from a draft. If the creator changes privacy in Finalize, use the privacy endpoint first, acknowledge its immediate effect, and refresh access_revision. Failure of later content publication does not roll back an already confirmed privacy change.

Return the updated published snapshot and current link. Concurrent/stale publishing returns conflict rather than overwriting.

### PATCH /me/publications/:id/privacy

Body: expected_access_revision, visibility.

Unchanged selection returns current state without rotating the link. A real visibility change revokes the current link, increments access_revision, changes visibility, and creates a fresh link if the destination is public/unlisted.

Return effective visibility/access_revision and new viewer_url if applicable. Preserve draft and published content; remove pinning when leaving public. Revoked links remain inactive.

### POST /me/publications/:id/delete

Body: expected_content_revision, expected_access_revision.

After UI confirmation, delete published content/search/pin data, revoke all active links, and set deleted_at. Retain metadata and existing draft. Return deleted publication ID and surviving_draft_id if present.

Repeated identical requests return the original outcome; they never delete a surviving draft.

### POST /me/drafts/:id/delete

Body: expected_revision.

Delete the exact owned draft after confirmation. Leave publication untouched. Late saves targeting this draft fail rather than recreating it.

### POST /me/drafts/:id/publish-as-new

Body: expected_draft_revision, visibility (required explicit selection), client_stack_id.

Only for a surviving draft whose source publication is deleted. Validate saved content, create new stack/publication/link, consume old draft, and retire old stack container atomically.

Return fresh identifiers and current viewer URL. Do not copy historical analytics or redirect deleted URLs. A concurrent repeated attempt is handled through the same request identity.

### PATCH /me/profile

Body: expected_revision plus allowed changed fields.

Update only creator-owned public profile fields after validation, with revision conflict detection. Reject unknown fields, owner ID, auth email, and arbitrary privilege fields. Avatar and handle-redirect workflows remain deferred.

### PUT /me/pins

Proposed body: ordered_publication_ids representing the creator's public pin list.

Validate all IDs are owned live public publications; apply positions atomically. Pin count policy must be finalized before this endpoint is implemented. Draft/private/unlisted/deleted publications cannot be pinned publicly.

## Subscription and export operations

### POST /creators/:handle/subscriptions

Body: email. Derive creator and signup_source=creator_page on the server.

Validate email; enforce per-creator/address uniqueness; apply rate limits. Return a generic accepted response without disclosing whether the address is already active, pending, or unsubscribed.

No anonymous subscriber SELECT or direct INSERT/UPDATE grants. Duplicate handling must not silently resubscribe an unsubscribed address. Confirmation/resubscription requirements remain unresolved; do not treat generic accepted feedback as proof an address became active.

### GET /me/subscriptions?status=&cursor=&limit=

Owned creator list only. Return address, signup date, source, and status. Paginated; no cross-creator filters accepted from the client.

### GET /me/subscriptions/export?status=active

Owner-authenticated CSV download. Default active-only; broader explicitly selected statuses remain labeled. Include email, signup date, source, and status; escape delimiters and formula-like cells. Use no-store and attachment headers; do not persist export files in publicly accessible storage.

Confirmation and unsubscribe token endpoints will be specified once their flow is settled. Do not invent newsletter sending or provider sync endpoints.

## Image exports

Prefer client rendering from an already authorized content snapshot; it requires no dedicated persistent image database model. A viewer exports published content; an owner can export a draft/private snapshot.

Generate at most five story pages. Show the current valid viewer URL for shareable overflow, preserve attribution, and avoid creating access links for drafts/private content. If server generation is later required, it must independently recheck token/ownership rather than trusting submitted IDs.

## Database permissions

Use private storage schemas for publication/draft/link/subscriber/receipt records and explicit safe projections. Enable RLS wherever owner-readable records are exposed through Supabase.

| Role/surface | Read | Write |
| --- | --- | --- |
| anon | Safe public projections and valid-token resolver only | Subscription submission through constrained server endpoint only |
| authenticated owner | Own draft/publication management through approved routes | Approved transactional functions only |
| authenticated nonowner | Same viewer access as anon | No writes to other creators' objects |
| controlled server reader | Minimal public/token lookup access | None for a read endpoint |
| controlled subscription writer | Necessary subscription upsert fields | Constrained subscription operation only |
| maintenance/admin | Operational scope, isolated credentials | Never exposed to client |

Revoke direct client INSERT/UPDATE/DELETE on tables whose invariants depend on transactions. Owner RLS alone does not prevent an owner bypassing revision checks or link rotation through raw writes.

Because client DML is denied, transactional functions may require a carefully restricted definer role. That role should have only necessary table privileges. Every function:
- Derives/authenticates the caller and checks ownership internally.
- Has a fixed safe search_path and schema-qualified object references.
- Has explicit execute grants; no accidental public execution.
- Rejects unauthenticated callers, unknown states, invalid content, and stale revisions.
- Uses consistent locking and revision checks.
- Does not return other creators' data or arbitrary SQL results.

Invoker functions remain preferred where their required grants do not permit bypassing invariants. A service-role bypass is not a substitute for explicit ownership/access validation.

Public projections must use current publication visibility/deletion state. No public projection exposes unlisted rows, draft content, raw share-link inventory, subscriber addresses, or mutation receipts.

## Operation-to-function mapping

| API operation | Proposed database function |
| --- | --- |
| First draft save | create_stack_draft |
| Open shared edit draft | open_edit_draft |
| Autosave | save_stack_draft |
| Initial publish | publish_stack_draft |
| Republish | republish_stack |
| Privacy change | change_publication_visibility |
| Publication deletion | delete_publication |
| Draft deletion | delete_stack_draft |
| Surviving draft publication | publish_surviving_draft |
| Subscriber submission | submit_creator_subscription |

Functions must commit their related records/receipts together. A server error after committing does not justify issuing a new request identity on retry.

## Verification before implementation acceptance

- Attempt raw table writes as anon, owner, and nonowner; denied paths must remain denied even when the UI is bypassed.
- Enumerate public/token APIs and verify no hidden collection or subscriber disclosure.
- Execute competing saves and publishes against the real database.
- Delay autosave until after draft consumption/deletion; no draft reappears.
- Simulate a committed mutation followed by a lost response; retry returns the same result.
- Race privacy changes, publishing, deletion, and link resolution; no stale request restores access.
- Verify creator search and exports cannot cross ownership boundaries.
- Verify expired sessions, invalid payloads, and failed writes preserve recoverable work without false success.

## Implementation gates and next step

This document is ready to drive a SQL migration proposal and integration tests. Before deployment, finalize content validation/counting, email confirmation/resubscription, pin count, and profile media/handle rules. These do not block writing the core draft/publication/link transactions.

Next: propose the schema constraints, function definitions, grants, and RLS policies in a reviewed migration. Synchronize latest confirmed flow decisions into product documentation alongside implementation.

References:
- [Data model](data-model.md)
- [Supabase database functions](https://supabase.com/docs/guides/database/functions)
- [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
