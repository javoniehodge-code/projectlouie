# Project Louie — Database migration proposal

Status: Review proposal, updated October 9, 2026 after final review. Not an executable migration. Function bodies and database tests are the next deliverable.

Companions: [data model](data-model.md), [API and permissions](api-and-permissions.md).

## Strategy

- Build Louie in a new Supabase project. Stack's database stays untouched, and none of Stack's social features can leak in.
- Don't replay Stack's migrations. Write Louie's schema fresh, reusing Stack's final content rules (`20261011000000_link_only_lines.sql`) and limits (`web/src/lib/format.ts`).
- No data moves from Stack to Louie.
- Apply to a local database first, then staging, then production. Never run concurrency or deletion tests on real creator data.

| Migration | Contents |
| --- | --- |
| 001_identity | private schema, profiles, handle_registry and reserved-handle checks |
| 002_content | content validators first, then stacks, stack_drafts, publications, link_codes, deleted_stack_ids and mutation receipts |
| 003_operations | All transaction functions from the data model |
| 004_access | grants, RLS, public read functions (no client access exists before this migration) |
| 005_subscriptions | email_subscriptions, purpose-scoped tokens, submit/confirm/removal functions |
| 006_search_and_pins | Search vector, creator search function, pin ordering |
| 007_analytics | analytics_events, 7/30-day queries, 90-day retention cleanup |
| 008_safety | reports, profile status, publication disabled state |

## Key constraints

```sql
CREATE SCHEMA louie_private;

CREATE TABLE public.profiles (
  id uuid PRIMARY KEY REFERENCES auth.users(id),
  handle text NOT NULL UNIQUE CHECK (handle ~ '^[a-z0-9._]{2,30}$'),
  handle_changed_at timestamptz,
  display_name text NOT NULL CHECK (char_length(btrim(display_name)) BETWEEN 1 AND 50),
  bio text NOT NULL DEFAULT '' CHECK (char_length(bio) <= 160),
  avatar_path text,
  links jsonb NOT NULL DEFAULT '{}'::jsonb,
  status text NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'suspended')),
  revision integer NOT NULL DEFAULT 1,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE louie_private.handle_registry (
  handle text PRIMARY KEY CHECK (handle ~ '^[a-z0-9._]{2,30}$'),
  profile_id uuid REFERENCES public.profiles(id) ON DELETE SET NULL,
  state text NOT NULL CHECK (state IN ('current', 'retired', 'reserved')),
  reserved_at timestamptz NOT NULL DEFAULT now(),
  retired_at timestamptz
);
CREATE UNIQUE INDEX one_current_handle_per_profile
  ON louie_private.handle_registry (profile_id) WHERE state = 'current';
-- All handle claims use this one registry; profile.handle is a transactional projection.
-- Account deletion changes its handles to reserved and clears profile_id.

CREATE TABLE louie_private.deleted_stack_ids (
  id uuid PRIMARY KEY,
  deleted_at timestamptz NOT NULL DEFAULT now()
);
-- Permanent identifier reservation only; delayed create requests must reject these IDs.
-- Define immutable content validators before the tables whose CHECKs invoke them.


CREATE TABLE louie_private.stacks (
  id uuid PRIMARY KEY,               -- supplied by the client on first save
  owner_id uuid NOT NULL REFERENCES public.profiles(id),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id, owner_id)
);

CREATE TABLE louie_private.publications (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  stack_id uuid NOT NULL UNIQUE,
  owner_id uuid NOT NULL,
  visibility text NOT NULL CHECK (visibility IN ('public', 'unlisted', 'private')),
  content jsonb NOT NULL CHECK (louie_private.can_publish_content_v1(content)),
  content_revision integer NOT NULL DEFAULT 1,
  access_revision integer NOT NULL DEFAULT 1,
  last_draft_id uuid NOT NULL,
  first_published_at timestamptz NOT NULL DEFAULT now(),
  content_updated_at timestamptz NOT NULL DEFAULT now(),
  search_document tsvector,
  pin_position integer CHECK (pin_position >= 0),
  disabled_at timestamptz,
  FOREIGN KEY (stack_id, owner_id)
    REFERENCES louie_private.stacks(id, owner_id) ON DELETE CASCADE,
  CHECK (pin_position IS NULL OR visibility = 'public')
);

CREATE TABLE louie_private.stack_drafts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  stack_id uuid NOT NULL UNIQUE,
  owner_id uuid NOT NULL,
  content jsonb NOT NULL CHECK (louie_private.has_draft_content_v1(content)),
  revision integer NOT NULL DEFAULT 1,
  proposed_visibility text DEFAULT 'public'
    CHECK (proposed_visibility IN ('public', 'unlisted', 'private')),
  base_content_revision integer,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (stack_id, owner_id)
    REFERENCES louie_private.stacks(id, owner_id) ON DELETE CASCADE
);

CREATE TABLE louie_private.link_codes (
  code text PRIMARY KEY CHECK (code ~ '^[a-hjkmnp-z2-9]{10}$'),
  publication_id uuid REFERENCES louie_private.publications(id) ON DELETE SET NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  revoked_at timestamptz
);
-- Revoked codes are kept forever so they are never reissued.

CREATE UNIQUE INDEX one_active_code_per_publication
  ON louie_private.link_codes (publication_id)
  WHERE revoked_at IS NULL;
```

The code alphabet leaves out `0`, `1`, `i`, `l` and `o`, which are easy to misread. That gives 31 symbols and about 8 × 10^14 possible codes.

A deferred constraint trigger checks the final state of each transaction:
- a live public or unlisted publication has exactly one active code;
- a private publication has none;
- a code with `publication_id` set to null is revoked.

Deleting a stack cascades to its publication and draft. The `delete_stack` function revokes the stack's code first, and the foreign key then clears `publication_id`, so the revoked code stays reserved.

Validators used in CHECK constraints must be immutable and depend only on their input. If content rules change, add a v2 validator and revalidate existing rows explicitly.

## Subscriptions

- Unique (creator_id, email). Status is `pending` or `active`; `confirmed_at` is required when active.
- Confirmation tokens are stored hashed with a 7-day expiry, in a table the Data API can't reach.
- Submission is one function, called from a rate-limited route after the Turnstile check. It never reveals existing status. It sends at most one confirmation per address per creator per 24 hours.
- Explicit confirmation/removal POSTs mutate; token GETs are read-only. Owner removal deletes the address and tokens. Repeat signup preserves active status.
- Account deletion cascades to subscriber addresses and tokens. Delivery to Resend is outside SQL; track rate-limited attempts and successful sends accurately, with safe retry handling.

## Analytics

- Raw `analytics_events` are retained for 90 days; a scheduled cleanup deletes older events. No daily rollup is required for v1’s 7/30-day ranges.
- Unique visitors use distinct IDs across the whole range, not summed daily uniques. Click history for removed URLs remains queryable within the range.
- Events cascade on publication and account deletion. Index `(publication_id, occurred_at)` and `(creator_id, occurred_at)`.

## Retry receipts and account deletion

- Create a private mutation-receipt table unique on actor/action ID/operation, with payload fingerprint and safe committed result. Receipts commit in the same transaction as mutations. Keep at least 90 days; expired IDs require reconciliation rather than replay.
- Privacy changes require access revision; deletion checks confirmed content/access/draft revisions. Permanently reserved deleted stack IDs prevent resurrection even after receipts expire.
- Retire/revoke codes before deleting publications. Account deletion preserves handle/code/stack-ID reservations while deleting content, events and subscriber data; media/auth cleanup must succeed before reporting completion. Failed cleanup remains publicly inaccessible and retryable.
- Suspension checks apply to every public resolver, search, image/preview, subscription and event path. Owner inspect/download/delete routes stay available; publish/republish/public sharing are blocked.
- Migration ordering must define functions only after all referenced tables/types exist. Implement subscription/report/account functions after their dependencies; do not interpret 003 as defining functions that reference later tables.

## Grants and RLS

- RLS is enabled on every table. Owner reads use `owner_id = auth.uid()` (`creator_id` for subscriptions and analytics).
- No client insert, update or delete grants on core tables, even for owners, because raw writes could skip revision checks and code rules.
- Mutation functions are `SECURITY DEFINER`, owned by a dedicated non-login role with only the table privileges it needs. Each has a fixed `search_path`, schema-qualified references and explicit execute grants. Default execute for `PUBLIC` is revoked.
- Public read functions return only live public content, or one published stack for a valid code.
- The service-role key stays on the server and is used only by the admin tools and scheduled jobs.

## Indexes

- `stacks (owner_id, updated_at, id)` for Manage.
- `publications (owner_id, first_published_at, id) WHERE visibility = 'public' AND disabled_at IS NULL` for creator pages.
- GIN on `publications.search_document` with the same filter.
- `stack_drafts (owner_id, updated_at, id)`.
- `email_subscriptions (creator_id, status, subscribed_at, id)`.
- Unique pin positions per owner among live public publications. `set_pins` clears affected positions before writing the new order in the same transaction.

## Required database verification

Run the verification table in [data-model.md](data-model.md#verification-required-before-acceptance) against real roles, grants, functions and concurrent sessions. Also verify:
- a stack can't be attached to another owner's records;
- a revoked code is never reissued after stack deletion;
- 7/30-day unique counts deduplicate across the entire range and retention cleanup preserves those ranges;
- old retries cannot undo later privacy/publication changes or recreate deleted IDs;
- account deletion preserves handle/code reservations and removes addresses/tokens;
- token GETs never confirm or remove subscriptions;
- one creator can't read another's subscriptions, analytics or reports.

## Deployment

1. Implement the function bodies and grants.
2. Apply to a fresh local database and run the verification matrix.
3. Repeat on staging with synthetic content.
4. Verify the Next.js routes, redirects and revocation behavior before launch.

Fix deployed migration problems with forward migrations; never drop live tables to roll back. A database restore must preserve revoked codes and retired handles before viewer traffic resumes.

References:
- [Supabase database functions](https://supabase.com/docs/guides/database/functions)
- [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Supabase Cron](https://supabase.com/docs/guides/cron)
- [PostgreSQL row locks](https://www.postgresql.org/docs/17/explicit-locking.html)
