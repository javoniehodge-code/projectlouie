# Project Louie — Product rules

Status: Confirmed product rules, updated October 9, 2026 after final review. These rules override older notes and inherited Stack behavior.

## Visibility and publishing state

| Concept | Meaning |
| --- | --- |
| Public | Anyone can view it, and it appears on the creator's public page and in search on that page |
| Unlisted | Anyone with its link can view it; it does not appear on the creator's public page or in search |
| Private | Only the creator can view it |
| Draft | Unpublished work; separate from visibility and always owner-only |

- Anonymous viewers can view creator public pages, public stacks, and unlisted stacks with a valid link.
- New stacks default to public. The creator's home page defaults to the private view. These two defaults are unrelated.

## Stack URLs

- Every shareable stack has a readable URL: `/{handle}/{title-words}-{code}`, for example `/maya/three-days-in-tokyo-k7q2m9xd4p`.
- The **code** identifies the stack's current link. The handle and title words are for readability only.
- Any URL with a valid code works while the publication is shareable, enabled and its creator active. If its handle or title words are out of date, it redirects to the current form. Title edits and handle changes never break a stack link.
- The code is random and unguessable, so unlisted stacks can't be found by guessing their URL.
- A creator page lives at `/{handle}`.

## Privacy changes and links

- Privacy changes take effect immediately.
- Moving between **public and unlisted** keeps the same link.
- Moving to **private** revokes the link permanently. Anyone opening it sees "This stack isn't available." The owner can still open the stack from their own tools.
- Making a private stack public or unlisted again issues a **new** link. The old link never works again.
- A stack that leaves public is removed from the creator page and its search, and loses its pin.

## Creation and content

- The Title and Description steps can be skipped.
- Publishing requires a title and at least one body line. A link-only line counts. A section heading alone does not.
- A draft can be saved once it has a title, a description or a body line. Blank drafts are not saved. Nonblank drafts autosave.
- Each stack has a 6,000-character total limit. Field limits and counting rules: [technical/data-model.md](technical/data-model.md#limits-and-counting).
- No location and no tags.

## Editing and republishing

- Each published stack has at most one edit draft, shared across tabs and devices.
- Viewers keep seeing the published version until the creator republishes.
- Republishing replaces the published version. It keeps the same link, publication date and analytics, and updates the "Last updated" date.
- A tab with outdated work gets a conflict prompt. It never silently overwrites newer saved work. See [user-flows.md](user-flows.md#edit-a-published-stack).
- Deleting an edit draft leaves the published stack unchanged.

## Deleting a published stack

- Every deletion asks for confirmation, states that it's permanent, and offers Cancel.
- Deleting a published stack deletes the stack, its edit draft (if any) and its analytics.
- Its link stops working permanently and is never reused.
- If an edit draft exists, the confirmation says so: "Delete this stack? Its link will stop working and your unpublished edits will be deleted too. This can't be undone."

## Sharing and exports

- Creators can share published public/unlisted stacks as links, and can export any owned stack as images, including private stacks and drafts. Drafts must be published before they have a shareable link.
- Owners also have a separate full-length image download containing the entire stack, without the five-page story limit.
- Creator public pages can be shared by link. Disabled stacks and suspended creators cannot be shared publicly through Louie.
- Viewers can share any stack they can view as a link or story images, and save the images.
- Story images have at most five pages. They show the creator's name and, for shareable stacks, the live link. A longer stack ends with "See all N lines" and the link.
- Exporting a private stack or draft never creates a link or grants access.
- An exported image is a snapshot. Changing or deleting the stack can't recall images already shared.
- Shared links show a preview card (title, description, creator and image) when pasted into messaging and social apps.

## Creator page

- Shows the profile picture, name, links, pinned public stacks, other public stacks, search within this creator's public stacks, and the email subscription form.
- Pins apply only to public stacks. Pin count and layout come from the profile design.
- The creator's private view shows every stack (public, unlisted and private) plus drafts, and links to Manage stacks.

## Handles

- Handles use lowercase letters, numbers, periods and underscores, 2–30 characters (same as Stack).
- A creator can change their handle at most once every 30 days.
- Old handles redirect permanently to the current one and can never be claimed by anyone else, including the same creator's other accounts.

## Email subscriptions

- The subscription form appears only on the creator's public page.
- Double opt-in: submitting the form sends a confirmation email through Resend. The subscriber counts as active only after confirming.
- The form response never reveals whether an address was already subscribed.
- Creators see their subscriber list and export active subscribers as a CSV. Louie doesn't send newsletters; creators email subscribers from their own tool.
- Creators can delete addresses from their Louie list. The confirmation email includes a secure removal link so subscribers can remove their address from Louie. Removal deletes the address and its tokens and excludes it from future exports.
- Removal cannot recall prior CSV downloads or unsubscribe someone from a creator’s external email tool; those emails must provide their own unsubscribe mechanism. Rejoining Louie requires fresh confirmation.

## Analytics

- Owner-only, opened from Settings.
- Ranges: last 7 days and last 30 days.
- Metrics per stack and for the creator page: page views, measurable unique visitors and link clicks per URL. Repeat visits count as additional page views.
- Click history for removed URLs stays available within the selected range until the stack or account is deleted.
- The creator's own visits are excluded.
- Unique visitors are counted with a first-party cookie. Request permission only where required for the selected analytics setup and remember the viewer’s choice. Where consent is required, do not set the analytics cookie before acceptance.
- Stacks and links work when permission is declined. Count non-identifying views/clicks only where permitted; unique counts cover measurable visitors and disclose that limitation. Unknown location uses the conservative consent-required behavior. Cookie preferences remain accessible.
- Link clicks are recorded with a background beacon when a viewer opens a link.
- Republishing and privacy changes keep a stack's analytics. Deleting a stack deletes its analytics.
- Details: [user-flows.md](user-flows.md#analytics).

## Abuse protection

- Every link in a stack or profile is checked against Google's unsafe-site lists when it is published or saved. Shortened links are expanded and the final destination is checked. Unsafe links block publishing, with an explanation.
- Every outbound link uses `rel="nofollow noopener noreferrer ugc"`.
- Sign-up and the subscription form are protected by Cloudflare Turnstile and rate limits.
- Every stack and creator page has a "Report" option.
- An admin can manually disable a stack or suspend a creator, reverse that action and log who acted and when. A full moderation dashboard is outside v1. Public access, sharing, search and previews are blocked.
- The owner can still sign in, inspect, download and delete their content through owner tools. Publishing, republishing and public sharing of affected content remain blocked until restored.
- Details: [user-flows.md](user-flows.md#abuse-protection-reporting-and-takedown).

## Account deletion

- Requires a clear permanent-deletion confirmation and a validated creator session.
- Deletes the profile, stacks, drafts, analytics and subscriber addresses/tokens, and removes profile media.
- Revokes every viewer link permanently. Current and retired handles stay reserved without retaining the deleted profile; they no longer redirect to a live account.
- Invalidates sessions. Delayed saves cannot recreate deleted content.
- Copies already downloaded or shared externally cannot be recalled.

## Remaining design details

- Profile layout and pin count.
- Story-image overflow wording for private stacks and drafts; no link is printed.
