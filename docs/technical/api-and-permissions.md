# Project Louie — API contracts and database permissions

Status: Implementation specification proposal, updated October 9, 2026 after final review. Nothing is deployed.

Companion: [data-model.md](data-model.md). The product documents win over this one.

## Architecture

- Next.js (same major version as Stack) serves pages and route handlers.
- Supabase Auth identifies creators. PostgreSQL functions own every multi-record change.
- Resend sends subscription confirmation emails. Cloudflare Turnstile protects sign-up, subscriptions and reports. Google Web Risk (the commercial version of Safe Browsing) checks links.
- The browser never chooses `owner_id` and never receives a privileged key. Route handlers take the user from the validated session.

## Common contract

Errors return:

```json
{ "error": { "code": "DRAFT_CONFLICT", "message": "This draft has newer saved changes." } }
```

| HTTP status | Code | Meaning |
| --- | --- | --- |
| 400 | INVALID_REQUEST | Malformed request |
| 401 | AUTH_REQUIRED | Missing or expired session |
| 404 | UNAVAILABLE | Missing, unauthorized, revoked, disabled or deleted |
| 409 | DRAFT_CONFLICT | Draft revision is stale or the draft no longer exists |
| 409 | CONTENT_CONFLICT | The published stack changed after the draft was opened |
| 409 | HANDLE_UNAVAILABLE | Handle taken or retired |
| 422 | CONTENT_INVALID | Content breaks a rule; names the field and limit |
| 422 | UNSAFE_LINK | A link failed the safety check; names the link |
| 429 | RATE_LIMITED | Includes `Retry-After`; also used for the 30-day handle rule |

Public failures always use one neutral `UNAVAILABLE`, so they never reveal whether hidden content exists.

Mutation requests use an `Idempotency-Key` UUID. Scope it to the authenticated actor and operation; store the payload fingerprint and successful safe result atomically with the write. The same key/payload returns its prior result, even after a later edit; different payload reuse returns `REQUEST_ID_REUSED`. This is a small receipt table, not an external queue or service. Retain receipts for at least 90 days; expired keys require reconciliation, never blind replay. Revisions remain mandatory for new actions. Initial save also checks permanent deleted-stack-ID reservations.

Additional errors: `ACCESS_CONFLICT` (409) for stale privacy state, `REQUEST_ID_REUSED` (409), and `TEMPORARILY_UNAVAILABLE` (503). Authenticated owners alone receive current revision details.

Cookie-authenticated writes require same-origin checks. Request bodies are size-limited and schema-validated.

## Public pages and reads

### `/{handle}` — creator page

Returns public profile fields, pinned public stacks, other public stacks (paginated, 50 at most, ordered by pin then `first_published_at` then ID), and the subscription form. A retired handle redirects permanently to the current one. A suspended creator shows "This page isn't available."

### `GET /api/creators/{handle}/stacks?q=&cursor=`

Search and pagination over the creator's live public publications only.

### `/{handle}/{slug}-{code}` — stack page

Resolves by code only. If the code is active and the publication is live, shareable, not disabled and owned by an active creator, it renders the stack. If the handle or slug is outdated, it uses a temporary no-store redirect to the canonical URL after validating current access. Otherwise it shows "This stack isn't available."

- Responses use `Cache-Control: no-store` so privacy changes and deletions take effect on the next request.
- The page sets Open Graph tags: title, description, creator and an image rendered from the first story page.
- Code lookups are rate-limited per IP.

## Creator reads

| Route | Returns |
| --- | --- |
| `GET /api/me/home` | Own profile and stack summaries for the private view |
| `GET /api/me/stacks?filter=all\|public\|unlisted\|private\|drafts&cursor=` | Own stacks with title, visibility, revisions, draft status and current URL |
| `GET /api/me/publications/{id}` | Own live content, visibility, revisions and current URL (none if private) |
| `GET /api/me/drafts/{id}` | Own draft content and revision, and the source publication's revisions for an edit draft |

## Creator writes

Each route calls exactly one database function.

| Route | Body | Function | Notes |
| --- | --- | --- | --- |
| `POST /api/me/stacks` | `stack_id`, `content`, `proposed_visibility` | `create_stack_draft` | First nonblank save |
| `POST /api/me/publications/{id}/edit-draft` | — | `open_edit_draft` | Returns the shared edit draft |
| `PATCH /api/me/drafts/{id}` | `expected_revision`, `content`, `proposed_visibility` (new stacks only) | `save_stack_draft` | Never creates a draft |
| `POST /api/me/drafts/{id}/publish` | `expected_revision`, `visibility` | `publish_stack_draft` | Runs the link safety check first |
| `POST /api/me/publications/{id}/republish` | `draft_id`, `expected_draft_revision`, `expected_content_revision`, `expected_access_revision` | `republish_stack` | Runs the link safety check first; never changes visibility |
| `PATCH /api/me/publications/{id}/privacy` | `expected_access_revision`, `visibility` | `change_visibility` | Immediate; returns the new URL or none |
| `POST /api/me/stacks/{id}/delete` | Expected publication content/access revisions and draft ID/revision (or null) | `delete_stack` | After the confirmation dialog |
| `POST /api/me/drafts/{id}/delete` | `expected_revision` | `delete_stack_draft` | After the confirmation dialog |
| `PATCH /api/me/profile` | `expected_revision` plus changed fields | `update_profile` | Profile links get the safety check |
| `POST /api/me/handle` | `expected_profile_revision`, `handle` | `change_handle` | 30-day rule; reserves the old handle |
| `PUT /api/me/pins` | Ordered publication IDs | `set_pins` | Pin count comes from the profile design |

If the creator changes privacy on the Finalize screen while editing, the privacy route runs first and takes effect immediately; republishing then changes only content.

## Email subscriptions

| Route | Who | Behavior |
| --- | --- | --- |
| `POST /api/creators/{handle}/subscriptions` | Anyone | Body: `email`, Turnstile token. Always answers "Check your inbox to confirm." Sends a confirmation through Resend unless one was sent to that address for this creator in the last 24 hours. |
| `GET /subscribe/confirm?token=` | Subscriber | Read-only confirmation page; no mutation on link opening. |
| `POST /api/subscriptions/confirm` | Subscriber | Valid unexpired confirmation token → active. |
| `GET /subscribe/remove?token=` | Subscriber | Read-only removal confirmation page, explaining external exports are unaffected. |
| `POST /api/subscriptions/remove` | Subscriber | Valid removal token → delete address and all associated tokens. |
| `POST /api/me/subscriptions/{id}/delete` | Owner | Confirmed deletion of own subscriber address and tokens. |
| `GET /api/me/subscriptions?cursor=` | Owner | Own list: email, status, signup and confirmation dates |
| `GET /api/me/subscriptions/export` | Owner | CSV of active subscribers: email, subscribed_at, confirmed_at. Cells are escaped against spreadsheet formulas. Sent with `no-store` as an attachment and never stored. |

The confirmation email comes from a Louie address, names the creator, includes a removal link and says to ignore it if the reader didn’t sign up. Repeat signup never downgrades active status. Delivery failure does not mark a successful send. Expired/used tokens return neutral feedback; resends require a fresh form submission with rate limits. Removal does not recall CSVs or unsubscribe external platforms.

## Analytics

| Route | Who | Behavior |
| --- | --- | --- |
| `POST /api/events` | Viewers' browsers | Batch of view and link_click events. Ignored for owners, bots, suspended creators and stacks the viewer can’t open. Validate the clicked URL against authorized current content; apply consent rules and rate limits even when no visitor cookie exists. |
| `GET /api/me/analytics?range=7d\|30d` | Owner | Totals per stack and for the creator page; clicks per URL |

Link clicks are sent with `navigator.sendBeacon` as the link opens, so the viewer goes straight to the destination. Cookie and consent rules: [user-flows.md](../user-flows.md#analytics).

## Reports and admin

| Route | Who | Behavior |
| --- | --- | --- |
| `POST /api/reports` | Anyone | Reason, optional details and email, Turnstile token |
| Minimal protected manual operations | Admins only | Inspect reports; disable/re-enable a stack; suspend/restore a creator; dismiss reports; log actions. No dedicated dashboard required in v1. |

Admin access uses a separate allowlist of admin user IDs, checked on the server. It is never exposed to creators.

## Image exports

Story images are rendered in the browser from content the requester is allowed to see (Stack's `StoryPages.ts`). Owners can export drafts and private stacks; viewers can export what they can open. Owners additionally have a separate full-length image download containing the entire stack. Drafts must publish to obtain links. No link is created for an export. Open Graph images render on the server only after checking current shareable access, active creator status and disabled state; use no-store. Already downloaded images cannot be recalled.

## Database permissions

- Core records live in a private schema that the Supabase Data API doesn't expose.
- The browser has no direct insert, update or delete on core tables. Every change goes through the functions above.
- Functions run as a dedicated limited role. Each one checks `auth.uid()` and ownership itself, uses a fixed `search_path`, and has explicit execute grants.
- Public read functions return only live public content, or a single published stack for a valid code.
- No public function exposes unlisted listings, codes in bulk, drafts, subscribers, analytics or reports.

## Verification before acceptance

- Try raw table reads and writes as anonymous, owner and non-owner users; all denied except approved functions.
- Race saves, publishes, privacy changes and deletions against the real database.
- Lose a publish response and retry; same result.
- Confirm owner visits and bot visits create no analytics events.
- Confirm one creator can't read or export another's subscribers or analytics.

References:
- [Data model](data-model.md)
- [Resend](https://resend.com/docs)
- [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/)
- [Google Web Risk](https://cloud.google.com/web-risk/docs)

## Account deletion and moderation access

`POST /api/me/account/delete` requires a validated session and explicit permanent-deletion confirmation. A retry-safe transaction removes profile, owned stacks/drafts/events and subscriptions/tokens, revokes codes and reserves current/retired handles and deleted stack IDs. Server cleanup removes media and deletes the auth identity/invalidates sessions. Do not report completed deletion before required cleanup succeeds; failed stages remain blocked from public access and can be retried safely.

Suspended owners retain sign-in, authenticated reads, downloads, stack/address deletion and account deletion. Publishing, republishing and public sharing remain denied. Every public page, search, preview/image, subscription and event path checks suspension/disable state independently. Admin reversals never reactivate permanently revoked codes.
