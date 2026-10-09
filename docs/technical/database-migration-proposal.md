# Project Louie — Database migration proposal

Status: Review proposal, not an executable/deployed migration. SQL excerpts specify the intended constraints; function bodies and database integration tests are the next implementation deliverable.

Companions: [data model](data-model.md), [API and permissions](api-and-permissions.md).

## Migration strategy

Build Louie in a new Supabase project/database rather than modifying Stack's production schema in place. This keeps Stack operational and prevents inherited feed/social behavior from entering Louie accidentally. This is a technical recommendation; no project has been provisioned.

Reuse reviewed content/editor logic where it fits. Do not copy Stack migrations wholesale. No Stack user/content migration is assumed.

Implement migrations in this order:

| Migration | Responsibility |
| --- | --- |
| 001_identity_and_content | Profiles, stacks, drafts, publications, links, receipts, validation |
| 002_transaction_functions | Save, open edit, publish, republish, privacy, deletion, surviving-draft publication |
| 003_access_surfaces | RLS, function grants, public projections, token resolver, owner reads |
| 004_subscriptions | Creator-specific subscriptions, protected collection and CSV reads |
| 005_search_and_pins | Derived creator search and atomic pin ordering |
| Later analytics migration | Metric/event schema after analytics specification |

During development, keep grants closed until functions and authorization tests are ready. Deploy the reviewed migration set together before exposing application routes. Apply migrations to a disposable/local database, then staging, then production; never test destructive concurrency scenarios on real creator data.

## Schema boundaries

Use:
- public.profiles for the explicitly public identity fields.
- louie_private for raw stack, draft, publication, link, subscriber, and receipt records.
- louie_api for deliberately callable database functions/projections, if configured for Supabase API exposure.

Do not expose louie_private through the Data API. Private-schema placement supplements grants/RLS; it does not replace them.

Keep service credentials server-only. Owner mutation functions validate the authenticated user; public read functions return a minimal projection. RLS and grants are tested with raw API calls, not only through Next.js.

## Record constraints

### Profiles

- id references auth.users(id).
- handle normalized to lowercase and unique.
- display_name is nonempty.
- revision is bigint >= 1.
- External link JSON shape is validated; account email/password never enter this public table.
- Handle format, media workflow, and final public fields must be reconciled with the updated profile design before profile migration is finalized.

Avoid an automatic cascading account-delete policy until account deletion is separately specified. Authentication-user deletion must not accidentally cascade through publication history or subscriber lists.

### Ownership throughout the schema

Store owner_id on the core private records and enforce ownership with composite foreign keys, not just function code.

SQL pattern:

```sql
-- profiles(id) is the authenticated creator identity.
CREATE TABLE louie_private.stacks (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id uuid NOT NULL REFERENCES public.profiles(id),
  lifecycle text NOT NULL DEFAULT 'active'
    CHECK (lifecycle IN ('active', 'retired')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (id, owner_id)
);

CREATE TABLE louie_private.publications (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  stack_id uuid NOT NULL UNIQUE,
  owner_id uuid NOT NULL,
  visibility text NOT NULL CHECK (
    visibility IN ('public', 'unlisted', 'private')
  ),
  content jsonb,
  schema_version integer NOT NULL DEFAULT 1 CHECK (schema_version >= 1),
  content_revision bigint NOT NULL DEFAULT 1 CHECK (content_revision >= 1),
  access_revision bigint NOT NULL DEFAULT 1 CHECK (access_revision >= 1),
  first_published_at timestamptz NOT NULL DEFAULT now(),
  content_updated_at timestamptz NOT NULL DEFAULT now(),
  deleted_at timestamptz,
  search_document tsvector,
  pin_position integer CHECK (pin_position >= 0),
  FOREIGN KEY (stack_id, owner_id)
    REFERENCES louie_private.stacks(id, owner_id),
  UNIQUE (id, stack_id, owner_id),
  UNIQUE (id, owner_id),
  CHECK (
    (deleted_at IS NULL AND content IS NOT NULL)
    OR
    (deleted_at IS NOT NULL AND content IS NULL
     AND search_document IS NULL AND pin_position IS NULL)
  ),
  CHECK (
    pin_position IS NULL
    OR (visibility = 'public' AND deleted_at IS NULL)
  )
);

CREATE TABLE louie_private.stack_drafts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  stack_id uuid NOT NULL UNIQUE,
  owner_id uuid NOT NULL,
  content jsonb NOT NULL,
  schema_version integer NOT NULL DEFAULT 1 CHECK (schema_version >= 1),
  revision bigint NOT NULL DEFAULT 1 CHECK (revision >= 1),
  proposed_visibility text NOT NULL DEFAULT 'public' CHECK (
    proposed_visibility IN ('public', 'unlisted', 'private')
  ),
  edit_of_publication_id uuid,
  base_content_revision bigint CHECK (base_content_revision >= 1),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  FOREIGN KEY (stack_id, owner_id)
    REFERENCES louie_private.stacks(id, owner_id),
  FOREIGN KEY (edit_of_publication_id, stack_id, owner_id)
    REFERENCES louie_private.publications(id, stack_id, owner_id),
  CHECK (
    (edit_of_publication_id IS NULL AND base_content_revision IS NULL)
    OR
    (edit_of_publication_id IS NOT NULL
     AND base_content_revision IS NOT NULL)
  )
);
```

The proposed_visibility column is used only for new/surviving drafts. Live edit publication operations ignore it and use current publication privacy. Function validation enforces that distinction.

A publication has one permanent identity within its original stack, including after deletion. A surviving draft is consumed while creating a different stack/publication; it is not reassigned to an existing unrelated publication.

### Share links

```sql
CREATE TABLE louie_private.share_links (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  publication_id uuid NOT NULL,
  owner_id uuid NOT NULL,
  token text NOT NULL UNIQUE,
  issued_access_revision bigint NOT NULL CHECK (issued_access_revision >= 1),
  created_at timestamptz NOT NULL DEFAULT now(),
  revoked_at timestamptz,
  FOREIGN KEY (publication_id, owner_id)
    REFERENCES louie_private.publications(id, owner_id),
  CHECK (token ~ '^[A-Za-z0-9_-]{43}$'),
  CHECK (revoked_at IS NULL OR revoked_at >= created_at)
);

CREATE UNIQUE INDEX one_active_link_per_publication
  ON louie_private.share_links(publication_id)
  WHERE revoked_at IS NULL;
```

Generate 32 random bytes and encode as URL-safe base64 without padding, producing 43 characters. The shape check does not establish randomness; the issuance function does.

The partial unique index prevents duplicate active links. A deferred constraint trigger additionally enforces:
- A live public/unlisted publication has exactly one active link.
- A private/deleted publication has no active link.
- The active link's issued_access_revision equals its publication's access_revision.

A deferred check allows revoke/update/issue steps within a transaction while rejecting invalid final states. Raw client updates cannot bypass this.

Revocation is permanent. Token, target publication, and issued generation are immutable after issuance. A trigger rejects attempts to clear revoked_at or move an old token to another publication.

### Mutation receipts

Primary key: owner_id + operation + request_id. Required fields: canonical payload fingerprint, safe result identifiers, completion timestamp.

Receipt writes commit with their mutation. Authorization is checked before lookup. Reject a duplicate request ID with different payload; replay the committed result for the same payload.

Use a canonical typed serialization before computing payload fingerprints; do not hash arbitrary JSON key order. Publish/delete receipts must not retain draft content or subscriber addresses. Retention duration needs an operational policy; callers cannot assume retry keys remain available indefinitely.

### Email subscriptions

- creator_id references profiles(id).
- normalized email, status, signup source, signup date, status-change timestamps.
- Unique (creator_id, normalized_email).
- status is pending, active, or unsubscribed.
- signup_source is server-controlled.
- An unsubscribed record requires unsubscribed_at.
- For active records, confirmation timestamps depend on the still-unresolved confirmation policy.
- Do not assume every pending/active subscription is also a Louie account.
- No cascade deletion from auth.users until account deletion is designed.

Anonymous submission executes one constrained function through a rate-limited server endpoint. It cannot read the list or activate a previously unsubscribed address by a generic upsert.

## Content validation functions

Define versioned validator functions before attaching content constraints:

| Function | Responsibility |
| --- | --- |
| validate_content_v1 | Shape, allowed types, stable IDs, supported formatting, URLs and limits |
| has_draft_content_v1 | Nonempty title, description, or valid body item |
| can_publish_content_v1 | Valid title plus at least one valid body item |
| content_text_count_v1 | Canonical text budget |
| publication_search_text_v1 | Derived searchable published text |

Do not accept arbitrary HTML or client-supplied search vectors. Duplicate item IDs and dangling links fail validation. A link-only item may count as a body item if valid under the content schema.

Known product limits: 6,000 text characters, no location/tags, no blank saved drafts. The exact field limits and definition of text counting still need final specification. Treat this as a migration implementation gate, not an invitation to copy inconsistent Stack counters.

Validator functions used in CHECK constraints must depend only on their input and be immutable. When semantics change, introduce a new schema/validator version and explicitly revalidate existing rows; changing a function alone does not recheck existing constraint data.

Drafts and live publications have separate validity requirements. Deleted publications have no content to validate.

## Transaction function design

All creator operations lock in the same order: stack, publication if present, draft if present. Authenticate, derive owner, verify request receipt, lock records, recheck revisions/state, validate, mutate, record receipt, return.

Rechecking after acquiring locks is mandatory. Owner IDs and timestamps are assigned by the database, not accepted from browser input.

| Function | Required transaction behavior |
| --- | --- |
| create_stack_draft | Reject blank content; create one stack/draft; retries converge |
| open_edit_draft | Return existing draft or copy current content once; no draft for deleted publication |
| save_stack_draft | Match exact draft ID + expected revision; increment revision; no upsert |
| publish_stack_draft | Validate publish requirements; create publication/link; consume draft |
| republish_stack | Check draft/content/access revisions; replace content/search; preserve publication/link identity; consume draft |
| change_publication_visibility | Apply immediately; revoke old link; issue new link if shareable; remove nonpublic pin |
| delete_publication | Check confirmed revisions; clear content/search/pin; revoke links; preserve draft/metadata |
| delete_stack_draft | Delete exact revision of draft; leave publication unchanged |
| publish_surviving_draft | Require deleted source and explicit privacy; new stack/publication/link; consume draft; retire old container |
| submit_creator_subscription | Validate target/address; deduplicate; do not expose status or silently resubscribe |
| set_publication_pins | Validate owned live public IDs; replace ordering atomically |

Hard failures roll back the entire operation. Do not catch and suppress errors that leave partial writes. A failed content publication after a previously committed privacy change leaves that privacy change in effect.

No automatic draft recreation after publish/delete. An old tab receives a conflict/unavailable result. No client-supplied visibility is applied implicitly during republishing.

## Grants and RLS proposal

Enable RLS on private records and explicitly restrict schema/table access. For any approved owner read, policy predicate is owner_id = auth.uid(); subscriptions use creator_id = auth.uid().

Do not grant client INSERT/UPDATE/DELETE for core tables, even with owner RLS. Such grants permit bypassing draft revision checks and transaction invariants.

Use narrowly owned SECURITY DEFINER mutation functions where client write grants are absent:
- Dedicated non-login function role, not a general administration role.
- Fixed search_path and schema-qualified tables/functions.
- Explicit auth.uid() checks and composite ownership relationships.
- REVOKE default function execution from PUBLIC; grant only approved entry points.
- Row/table privileges limited to what the function needs.
- If RLS applies to the function-owner role, configure deliberate policies; do not accidentally assume definer means automatic RLS bypass.

Separate public readers from creator mutation entry points. A token resolver is allowed to resolve a valid token and return one minimal published snapshot, not enumerate raw unlisted records. Public collection readers filter live public publications before search/pagination.

Service-role access must stay server-only. Every server operation using privileged access still enforces ownership or viewer token validity. Never rely on a hidden UI button for authorization.

## Search and pin indexes

Initial indexes:
- stacks(owner_id, updated_at, id) for management cursors.
- publications(owner_id, first_published_at, id) filtered to live public records.
- GIN on publication search_document, filtered to live public records.
- Unique active share token lookup through share_links(token).
- drafts(owner_id, updated_at, id) for owner draft lists.
- subscriptions(creator_id, status, subscribed_at, id) for owner list/export.

Derive publication search text during publish/republish; privacy filters govern whether it can be queried. Keep deleted search_document null.

Public pin ordering needs deterministic tie-breaking. A unique pin position per owner among live public publications can enforce ordering, but swapping positions must first clear affected positions within the same transaction before setting the replacement order. Maximum pin count is deferred to the profile design.

## Required database verification

| Test | Outcome |
| --- | --- |
| Cross-owner foreign key attachment | Database rejects |
| Two concurrent edit opens | Exactly one draft |
| Two concurrent saves of revision N | One new revision; one conflict |
| Blank draft / missing publish title or body | Validation rejects |
| Old save after consuming draft | No recreated draft |
| Duplicate publish request after lost response | Same committed result |
| Duplicate request with changed payload | Rejected |
| Public → unlisted/private | Old token revoked; correct new access |
| Privacy change during content editing | Draft survives; stale publication blocked |
| Link issuance failure | No partial privacy/publication commit |
| Delete publication with draft | Content gone, draft retained, link inactive |
| Publish surviving draft | New publication/link; old publication remains deleted |
| Raw anonymous/owner writes | Denied outside approved functions |
| Token enumeration/internal-ID access | No hidden content disclosure |
| Other creator's subscriber export | Denied |
| Five-page export | No schema-driven creation of a public draft/private link |

Tests must run against real database roles, grants, functions, and concurrent sessions. Mocked endpoint tests alone do not establish these guarantees.

## Deployment and rollback

Before applying to production:
1. Confirm content schema/counting and reconcile updated product docs.
2. Implement all function bodies and permission grants.
3. Apply to a fresh development database; run the verification matrix.
4. Repeat against staging with representative synthetic content.
5. Validate Next.js endpoints and revocation behavior before launch.

Schema rollout should be additive until verified. Fix failed deployed migrations with forward migrations; do not drop live tables to revert a release. Backup/recovery planning is separate from application deletion: restoring a database must preserve revoked/deleted access state before reopening viewer traffic.

## What this proposal does not claim

These SQL excerpts have not been executed against Supabase/PostgreSQL. They are constraint designs, not a tested ready-to-run migration. No database project, secret, deployment, or production change has been created.

The next implementation artifact is the actual migration set plus database integration tests. Email confirmation, exact profile changes, account deletion, and analytics storage remain separate decisions; core publication/draft/link implementation can proceed once content validation is settled.

Official references:
- [Supabase database functions](https://supabase.com/docs/guides/database/functions)
- [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [PostgreSQL row locks](https://www.postgresql.org/docs/17/explicit-locking.html)
- [PostgreSQL unique and partial indexes](https://www.postgresql.org/docs/current/sql-createindex.html)
