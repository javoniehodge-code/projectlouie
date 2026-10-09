# Project Louie

Status: Product scope for the initial release. Updated October 9, 2026 after final review. Where this document and older notes disagree, this document wins. Detailed behavior lives in [product-rules.md](product-rules.md) and [user-flows.md](user-flows.md).

## Product purpose

Louie is a link-in-bio product with email capture, built as a slimmed-down spin-off of Stack. Creators organize information into stacks (itineraries, guides, curated lists), share individual stacks or their creator page, collect email subscribers, and see how viewers use their links.

Louie is not a social platform. There is no feed, discovery, following or engagement layer.

## Who Louie serves

Social media creators whose content benefits from an organized, reusable destination beyond a single post. Examples:

- Travel creators sharing multiple itineraries.
- AI creators sharing multiple how-to guides.
- Art creators sharing event and gallery recommendations.

Anyone can sign up as a creator. Viewers never need an account.

## Primary journeys

### Viewer

- Social link → individual stack → creator's public page → another stack.
- Social link → creator's public page → individual stack. This is the link-in-bio journey.
- Creator's public page → subscribe by email → confirm from the inbox.

### Creator

Home page → create stack → publish → share → manage stacks → check analytics → export subscribers.

## Launch quality bar

Carried over from Stack: anything shipped must feel like a finished product, not a prototype. A feature that is not fully specified, designed and verified is cut from the release rather than shipped partially. Every screen needs its loading, empty, error and unavailable states.

## Initial release screens

| Screen | Who | Spec |
| --- | --- | --- |
| Individual stack | Anyone with access | [user-flows.md](user-flows.md#view-a-stack) |
| Creator's public page | Anyone | [user-flows.md](user-flows.md#creator-page-public-page-and-private-view) |
| Creator's private view (home) | Creator | [user-flows.md](user-flows.md#creator-page-public-page-and-private-view) |
| Manage stacks | Creator | [user-flows.md](user-flows.md#manage-stacks) |
| Sign up / sign in | Creators | [user-flows.md](user-flows.md#sign-up-sign-in-and-sign-out) |
| Create stack | Creator | [user-flows.md](user-flows.md#create-stack) |
| Edit stack | Creator | [user-flows.md](user-flows.md#edit-a-published-stack) |
| Share sheet and image export | Creator and viewers | [user-flows.md](user-flows.md#share-a-stack-or-creator-page) |
| Email subscription and confirmation | Viewers | [user-flows.md](user-flows.md#email-subscription) |
| Subscribers and CSV export | Creator | [user-flows.md](user-flows.md#email-subscription) |
| Analytics (in Settings) | Creator | [user-flows.md](user-flows.md#analytics) |
| Settings, including handle change | Creator | [user-flows.md](user-flows.md#change-handle) |
| Report a stack or creator | Anyone | [user-flows.md](user-flows.md#abuse-protection-reporting-and-takedown) |

Privacy changes and deletion are covered in [user-flows.md](user-flows.md#change-a-stacks-privacy) and [user-flows.md](user-flows.md#delete-a-stack-or-draft).

## Scope summary

### In the initial release

- Stacks with public, unlisted and private visibility, plus drafts.
- The Create Stack flow carried over from Stack as-is: Title → Description → Body → Finalize → Preview. Editing starts at Body.
- Readable stack URLs: `/{handle}/{title-words}-{code}`.
- Creator public page with profile picture, name, links, pinned and public stacks, search within the creator's stacks, and the email subscription form.
- Creator private view and Manage stacks.
- Share sheet with links and story images for creators and viewers. Owners also have a separate full-length image download; drafts must be published before a shareable link is available.
- Creator-page sharing.
- Account deletion with removal of content, analytics and subscribers; links and handles remain permanently reserved.
- Email capture with double opt-in. Louie sends the confirmation email through Resend. Creators export a CSV and email subscribers from their own tool. Creators can delete addresses, and subscribers can remove themselves from Louie’s list.
- Analytics: page views, measurable unique visitors and link clicks, for the last 7 or 30 days. Consent prompts appear only where required for the chosen analytics setup, and choices are remembered.
- Handle changes, at most once every 30 days, with permanent redirects from old handles.
- Abuse protection: link safety checks, bot protection, reporting and admin takedown.

### Outside the initial scope

- Comments, likes, saves, reposts and forks.
- Explore, discovery, global search and feeds.
- Followers and following, notifications and mentions.
- Sending newsletters or emails to subscribers from Louie, and email-provider integrations.
- Paid or locked stacks, pricing and commerce.
- Location and tags on stacks.
- Stack's "Publish & Share Update", update notes and feed cooldowns.

## Reuse from Stack

The Create Stack flow and its content format are reused as-is (see [technical/data-model.md](technical/data-model.md#content-format)). Stack's sharing sheet, story-image export, visibility sheet, sign-in sheet and Manage screen are adapted. The profile page comes from the latest Claude Design version, not Stack's current code. Reused code doesn't set Louie's product rules: the rules in this repository do.

## Waiting on the profile design

The updated profile design from Claude Design hasn't been folded in yet. It will include the subscription form and pins. Until it lands, these stay open: exact profile fields, avatar upload and cropping, pin count, the placement of creator-page search, and the placement of the prompt on a stack that leads to the creator's other stacks.

## Remaining design details

- Profile layout and fields, pin count, search placement and creator-page sharing appearance remain dependent on the updated design.
- Exact story-image overflow wording for private stacks and drafts; it must never suggest a working link.

## V1 decision principle

Louie is an untested product and this release tests demand. Build the smallest complete creator and viewer experience. Use manual report review and takedown; do not add a dedicated moderation dashboard, newsletter sending, provider integrations or advanced analytics.
