# Project Louie — User flows

Status: Behavior specs for every launch screen, updated October 9, 2026 after final review. Rules come from [product-rules.md](product-rules.md); if they disagree, product-rules.md wins.

- [Create Stack](#create-stack)
- [Edit a published stack](#edit-a-published-stack)
- [Change a stack's privacy](#change-a-stacks-privacy)
- [Delete a stack or draft](#delete-a-stack-or-draft)
- [View a stack](#view-a-stack)
- [Creator page: public page and private view](#creator-page-public-page-and-private-view)
- [Manage stacks](#manage-stacks)
- [Share a stack or creator page](#share-a-stack-or-creator-page)
- [Email subscription](#email-subscription)
- [Analytics](#analytics)
- [Change handle](#change-handle)
- [Sign up, sign in and sign out](#sign-up-sign-in-and-sign-out)
- [Abuse protection, reporting and takedown](#abuse-protection-reporting-and-takedown)

## Create Stack

Carried over from Stack as-is (Stack-App `main`, `web/src/app/create/CreateScreen.tsx`). Only the changes listed under "Differences from Stack" are made. The screens and interactions are not redesigned.

### Entry points

- **Create** on the creator's home page.
- A signed-out visitor who opens `/create` sees the sign-in sheet. After signing in, they land on the Title step.

### Flow

1. **Title.** "What would you like to call this Stack?" Up to 60 characters. Return or Next advances; Skip appears while empty. The title then floats into its final position.
2. **Description.** Optional, up to 180 characters, with a visible counter. Shift+Return adds a new line. Next or Skip advances. Tapping the title goes back to it.
3. **Body.**
   - The first line starts as a numbered line with the cursor in it.
   - Title and description stay editable here.
   - Under the selected line, a toolbar offers Section, Paragraph, Numbered, Bulleted, Link and inline Bold.
   - Return continues a list. Return on an empty list line turns it into a paragraph. Return on an empty paragraph ends writing.
   - Links can be typed into text (they become link pills) or attached with the link sheet, which accepts an optional name.
   - **Edit** opens Reorder or Delete. Delete puts a delete button on each line and heading; deletions offer Undo for about five seconds.
   - The footer hides while writing and returns when done.
   - Finalize needs at least one line with text or a link.
4. **Finalize.** Choose visibility: Public (default), Unlisted or Private, with a one-line explanation of each. Open Preview.
5. **Preview.** The stack as viewers will see it, with Back and Publish in the footer.
6. **Publish.** With no title, a "Name your Stack to publish" sheet appears. On success: "Your Stack is live," with **Share your Stack**, **View your Stack** and **Exit**.

At any point: **Save draft** saves and stays; **Exit** saves and returns to the home page. Exit never discards work.

### Rules

1. Only signed-in creators can create stacks.
2. Publishing requires a title and at least one line with text or a link. A heading alone doesn't count.
3. A draft is saved once it has a title, a description or a line. Blank work is never saved.
4. Nonblank drafts autosave about 1.2 seconds after typing stops.
5. Limits: [data-model.md](technical/data-model.md#limits-and-counting). Over-limit content shows what to shorten and blocks saving and publishing.
6. Links that fail the safety check block publishing and name the problem link.
7. Drafts never appear on public surfaces.

### Differences from Stack

| Stack today | Louie |
| --- | --- |
| Finalize asks for a location | No location |
| "Invite Only" | "Unlisted", with no mention of a feed |
| "Back to the feed" on the sign-in gate | "Back" to the previous page |
| Fork and feed-update code paths | Removed |
| A failed autosave isn't retried | Failed saves retry automatically, and the status shows "Not saved — retrying" until one succeeds |
| No warning when closing or reloading with unsaved changes | The browser warns before leaving with unsaved changes |
| `/create` keeps its URL after the first save, so reloading opens a blank stack | After the first save the URL becomes `/create?draft={id}`, so reloading reopens the draft |
| A missing draft opens a blank editor silently | "This draft isn't available," with a link home |
| A retried first save can create a duplicate | The client chooses the stack ID, so retries reach the same draft |

### Failure cases

1. A stack without a title or line is published.
2. Work is lost after a failed save, a reload or a lost connection.
3. A retry creates a duplicate stack.
4. The app reports a stack as published when it wasn't saved.
5. A signed-out visitor can create or publish.
6. A draft shows up publicly.

### Expected results

- A published stack persists after reload and opens at its URL.
- Sections, lines, formats, bold words, links and link names are saved in order.
- The chosen visibility is applied, and a public or unlisted stack has a working link immediately.
- Drafts reopen at Body with their content.

---

## Edit a published stack

Uses the Create Stack screens from Body onward.

### Entry points

- **Edit** on a stack in the creator's private view.
- **Edit** in a stack's menu on Manage stacks.
- **Edit** in the owner menu on the stack's own page.

### Flow

1. The editor opens at **Body** with the stack's shared edit draft. If none exists, one is created from the published version.
2. The creator changes the title, description, sections, lines or links. Changes autosave to the edit draft; viewers still see the published version.
3. **Finalize** shows the current visibility. Changing it applies immediately, before republishing (see [change-privacy.md](#change-a-stacks-privacy)).
4. **Preview** is optional.
5. **Publish** replaces the published version. The link, publication date and analytics stay; "Last updated" changes. The creator lands on the stack page.

**Exit** saves the edit draft and returns. The draft appears under the stack in the private view and on Manage stacks until it's published or deleted.

### Conflicts

- **Another tab saved newer changes to this draft.** The stale tab shows: "This draft was changed in another tab." Options: **Load latest** (the stale tab's unsaved changes are discarded after one more confirmation) or **Keep mine** (overwrites the newer draft after confirmation).
- **The published stack changed after this draft was opened.** This happens if another draft was published from a different tab first. Publish shows: "This stack was updated after you started editing." Options: **Replace it with my version** or **Discard my draft and edit the latest**.
- **The draft or stack was deleted elsewhere.** "This stack isn't available anymore," with a link home. Work is never silently recreated.

### Rules

1. Each published stack has at most one edit draft.
2. Saving the edit draft never changes the published stack, its position, its link or its visibility.
3. Publishing requires a title and at least one line with text or a link.
4. Publishing never creates a second stack.
5. Deleting the edit draft (with confirmation) leaves the published stack unchanged.

### Failure cases

1. An edit draft becomes visible to viewers.
2. A stale tab silently overwrites newer work.
3. Republishing changes the link or resets analytics.
4. Republishing undoes a privacy change made during editing.
5. An autosave recreates a draft after it was published or deleted.

### Expected results

- Viewers see the new version on their next load.
- The URL keeps its code; if the title changed, the title words update and old URLs redirect.
- The edit draft disappears after publishing.

---

## Change a stack's privacy

### Entry points

- Finalize, while creating or editing.
- The owner menu on the stack's page.
- A stack's menu on Manage stacks.

### Flow

1. The creator opens the visibility sheet (adapted from Stack's `components/Visibility.tsx`), which shows Public, Unlisted and Private with a one-line explanation of each.
2. Choosing a new option applies it immediately. A toast confirms the change and explains its effect on the link:

| Change | Link | Toast |
| --- | --- | --- |
| Public → Unlisted | Same link | "Unlisted. It's off your page, and anyone with the link can still open it." |
| Unlisted → Public | Same link | "Public. It's on your page." |
| Public or Unlisted → Private | Link stops working for good | "Private. Links you've shared won't work anymore, even if you make it public again." |
| Private → Public or Unlisted | New link | "Public with a new link. Old links stay off." |

3. Moving to Private asks for confirmation first, because it permanently breaks links already posted.

### Rules

1. Only the owner can change privacy.
2. Changes apply immediately and don't touch the content or the edit draft.
3. Leaving Public removes the stack from the creator page and its search, and clears its pin.
4. A revoked link never works again.
5. Analytics continue through every privacy change.

### Failure cases

1. A private stack opens from an old link.
2. Public ↔ Unlisted changes the link.
3. An unlisted stack appears on the creator page or in search.
4. Republishing a draft reverts a privacy change.

### Expected results

- The next request to any affected URL reflects the new privacy setting.
- The share sheet always shows the current link, or explains that private stacks have none.

---

## Delete a stack or draft

### Entry points

- A stack's menu on Manage stacks.
- The owner menu on the stack's page.
- A draft's **Delete** in the private view or on Manage stacks.

### Flow

1. The creator chooses **Delete**.
2. A confirmation dialog appears with **Cancel** and a red **Delete**:
   - Published stack without an edit draft: "Delete this stack? Its link will stop working and its analytics will be deleted. This can't be undone."
   - Published stack with an edit draft: "Delete this stack? Its link will stop working, and your unpublished edits and its analytics will be deleted too. This can't be undone."
   - Draft of a new stack: "Delete this draft? This can't be undone."
   - Edit draft: "Delete your unpublished edits? The published stack won't change."
3. After deletion, the item disappears from every owner surface and a toast confirms it.

### Rules

1. Only the owner can delete.
2. Every deletion needs confirmation (Stack currently deletes drafts with one click; Louie doesn't).
3. Deleting a published stack deletes its publication, edit draft and analytics, and revokes its link permanently.
4. Deleting an edit draft leaves the published stack unchanged.
5. Deletion is all or nothing. Repeating it is harmless.
6. If deletion fails, nothing is deleted, an error explains it, and the creator can retry.

### Failure cases

1. Deletion happens without confirmation, or after Cancel.
2. A deleted stack still opens from its URL or appears on the creator page.
3. Deleting an edit draft changes the published stack.
4. A stale tab recreates a deleted stack or draft.
5. The app reports success when deletion failed.

### Expected results

- The stack's URL shows "This stack isn't available."
- Its link code is never reused.
- A stale tab that tries to save gets "This stack isn't available anymore."

---

## View a stack

### Entry points

- A link shared on social media or in a message: `/{handle}/{title-words}-{code}`.
- A stack on the creator's public page.

### Flow

1. The viewer opens the stack. No account is needed.
2. They see the title, description, the creator's name and picture (linking to the creator page), "Last updated," and the sections and lines.
3. Links open in a new tab. Each click is recorded with a background beacon (see [analytics.md](#analytics)).
4. A prompt invites them to see the creator's other stacks and opens the creator page. Its exact placement comes from the design.
5. **Share** opens the share sheet (see [share-and-export.md](#share-a-stack-or-creator-page)).
6. **Report** is available from a menu (see [report-and-takedown.md](#abuse-protection-reporting-and-takedown)).

The owner viewing their own stack also sees **Edit**, **Privacy** and **Delete** in an owner menu.

### Rules

1. Public and unlisted stacks open for anyone with a valid link.
2. Private stacks, drafts, disabled stacks and stacks of suspended creators are unavailable publicly. Owners inspect/download/delete through authenticated owner tools; those routes do not expose viewer links.
3. A URL with an old handle or old title words redirects to the current URL.
4. Outbound links use `rel="nofollow noopener noreferrer ugc"`.
5. Pasting the URL into messaging or social apps shows a preview card with the title, description, creator and an image.

### Failure cases

1. A private, deleted or revoked stack's content is visible.
2. An outdated URL with a valid code shows an error instead of redirecting.
3. The owner's visit is counted in analytics.

### Expected results

- The page is readable on a phone without zooming and works on slow connections.
- The unavailable page links to the creator's page if the creator is active.

---

## Creator page: public page and private view

The layout comes from the latest Claude Design profile, which hasn't been folded in yet. It will include the subscription form and pins. This spec covers behavior only; it does not use Stack's current profile code as the design source.

### Public page

#### Entry points

- The creator's link in bio: `/{handle}`.
- The creator's name or picture on any of their stacks.
- The "see other stacks" prompt on a stack.

#### Contents

- Profile picture, name, optional bio, and social and website links.
- The email subscription form (see [email-subscription.md](#email-subscription)).
- Pinned public stacks, in the creator's order, then other public stacks, newest first.
- Search across this creator's public stacks.
- **Share** for the page and **Report** in a menu.

#### Rules

1. Anyone can view it without an account.
2. Only public stacks appear, and only public stacks can be pinned.
3. Search covers only this creator's public stacks.
4. `/{old handle}` redirects permanently to `/{current handle}`.
5. A suspended creator's page shows "This page isn't available."
6. Pasting the page URL into messaging or social apps shows a preview card with the name, picture and bio.

### Private view (home)

#### Entry points

- After sign-in.
- `/{handle}` while signed in as that creator; it opens in the private view.

#### Contents

- A toggle between **Private view** (default) and **Public view**, which shows exactly what visitors see.
- Every stack (public, unlisted and private) with a badge showing its visibility, plus drafts and edit drafts.
- **Create**, **Manage stacks**, **Edit profile**, **Settings**.
- **Edit** on each stack.

#### Rules

1. Only the creator sees the private view.
2. Drafts and edit drafts appear only here and on Manage stacks.
3. The public view matches what a signed-out visitor sees.

### Failure cases

1. An unlisted, private or draft stack appears on the public page.
2. A visitor reaches the private view.
3. An old handle shows an error instead of redirecting.

### Expected results

- A creator can put `/{handle}` in their bio and it shows their public stacks and subscription form.
- Visibility changes show up on the page on the next load.

---

## Manage stacks

Adapted from Stack's Manage screen (`web/src/app/settings/stacks/ManageScreen.tsx`), moved out of Settings.

### Entry point

**Manage stacks** in the creator's private view.

### Contents

- A list of every stack, with filters: All, Public, Unlisted, Private, Drafts.
- Each row shows the title (or "Untitled draft"), a visibility badge, "Last updated," and whether it has unpublished edits.
- A link button that opens the share sheet for shareable stacks.

### Actions

| Item | Menu |
| --- | --- |
| Published stack | Edit, Privacy, Share, Delete |
| Draft of a new stack | Edit, Delete |
| Stack with unpublished edits | Also: Delete edits |

Deletion uses the confirmations in [delete-stack.md](#delete-a-stack-or-draft). Privacy uses [change-privacy.md](#change-a-stacks-privacy).

### Rules

1. Only the owner sees this screen.
2. Every destructive action asks for confirmation.
3. The list updates immediately after any change.

### Failure cases

1. Another creator's stacks appear.
2. A destructive action runs without confirmation.

### Expected results

- A creator can find, edit, change the privacy of, share or delete any stack in a few taps.
- Empty states explain how to create a first stack.

---

## Share a stack or creator page

Adapted from Stack's share sheet and story images (`web/src/components/Share.tsx`, `web/src/components/StoryPages.ts`).

### Entry points

- **Share** on a stack page (creator and viewers).
- **Share your Stack** on the "Your Stack is live" screen.
- The link button on Manage stacks.
- **Share** on the creator's public page.

### Flow

1. The share sheet shows the story images in a swipeable row.
2. Actions: **Copy link**, **Save image(s)**, and **Share…** (the device's share menu, with the images and link).
3. For a private published stack, the sheet explains that it has no viewer link and offers images only. **Make it shareable** opens the privacy sheet.
4. A draft has image export only. **Publish to share a link** opens its editor at Body, then Finalize; changing proposed privacy alone never creates a link.
5. Owners also see **Download full-length image**, containing the entire stack separately from the story images. Export failures preserve the content and offer Retry.
6. Creator-page sharing offers its public link and device sharing. Its image/card layout follows the updated profile design.
7. For disabled content or a suspended creator, owner tools offer download only; public sharing stays blocked.

### Story images

- 9:16 pages (1080 × 1920) with the platform's top and bottom areas left clear, as in Stack.
- Page 1 shows the creator's name and picture, title and description; later pages show a slim header with the title and "2 / 4".
- Lines are never split across pages, and a section heading never ends a page.
- At most five pages. A longer stack fades into "See all N lines" with the live link.
- The last page shows the creator's name and the live link.
- Removed from Stack's version: like, save and repost counts, and "Make your own stack →".
- Content area is taller than Stack's, so more fits per page. The exact layout comes from the design.

### Rules

1. Creators can export any owned stack, including private stacks and drafts. Only live public/unlisted publications have shareable viewer links. Owners have a separate full-length image download containing every line.
2. Viewers can share and export any stack they can open.
3. Exporting never creates a link or grants access.
4. Images are snapshots; changing or deleting the stack can't recall shared copies.
5. Private stacks and drafts have no link to print. Their overflow text is still to be decided.

### Failure cases

1. An image shows a link for a private stack or draft.
2. An image exposes content the requester can't view.
3. Text overflows or is clipped on any page.

### Expected results

- A creator can post a stack to Instagram or TikTok stories in a few taps.
- Every image shows who made it and, when shareable, where to find the full stack.

---

## Email subscription

Louie collects subscribers for creators. It does not send newsletters: creators export a CSV and email subscribers from their own tool.

### Viewer: subscribe

#### Entry point

The subscription form on the creator's public page. It appears nowhere else.

#### Flow

1. The viewer enters their email and taps **Subscribe**. Cloudflare Turnstile runs in the background.
2. The form shows: "Check your inbox to confirm your subscription to {name}."
3. Louie sends a confirmation email through Resend:
   - From a Louie address, naming the creator.
   - One **Confirm subscription** button.
   - "If you didn't sign up, ignore this email."
4. Tapping the email button opens a confirmation page. The viewer taps **Confirm subscription** there, then sees "You’re subscribed to {name}" and a link back. Opening the email URL alone never confirms, so email scanners cannot subscribe someone.
5. The email also contains a secure **Remove my email from Louie** link. It opens a confirmation page; confirming removes the address and associated tokens from Louie’s list. It explains that earlier exports and external mailing lists are unaffected.

#### Rules

1. The subscriber becomes active only after confirming.
2. The form's response is always the same, whether the address is new, pending or already active, so no one can check who subscribes to whom.
3. A confirmation email goes to the same address for the same creator at most once every 24 hours.
4. Confirmation links expire after 7 days. An expired or used link opens a page offering to send a new one.
5. The form includes a short consent line: "{name} will get your email address. Louie's privacy policy applies."
6. Submissions are rate-limited per IP.

### Creator: subscribers and export

#### Entry point

**Subscribers** in Settings.

#### Flow

1. The creator sees the count of active subscribers and a list (email, signup date, confirmed date, status).
2. **Export CSV** downloads active subscribers: email, subscribed_at, confirmed_at.
3. **Delete address** asks for confirmation, then removes the address and associated tokens. It disappears from the list and future exports; previously exported copies are unaffected.

#### Rules

1. Creators see only their own subscribers.
2. Pending (unconfirmed) addresses are listed but not exported.
3. The CSV is generated on demand and never stored.
4. Removed addresses are not retained as subscriber records. A later signup requires a new confirmation. Repeating signup for an active address never downgrades it to pending.
5. Failed email delivery provides retryable feedback and does not mark a failed delivery as successfully sent.

### Out of scope for this release

- Sending emails to subscribers from Louie.
- Integrations with email providers.
- Unsubscribing from the creator’s external mailing tool. Louie supports removal from its own list only.

### Failure cases

1. An address becomes active without confirming.
2. The form reveals that an address is already subscribed.
3. Someone floods an address with confirmation emails.
4. A creator sees or exports another creator's list.

### Expected results

- A viewer can subscribe in one step plus one confirmation tap.
- A creator can export a clean list of confirmed subscribers at any time.

---

## Analytics

### Entry point

**Analytics** in Settings.

### What creators see

- Ranges: last 7 days and last 30 days.
- For the creator page and for each stack: page views, measurable unique visitors and link clicks.
- For each stack: clicks per URL, most-clicked first, showing the link's name if it has one.

### Definitions

| Metric | Definition |
| --- | --- |
| Visit | One page load of a stack or the creator page by someone other than the owner |
| Page views | Number of visits, including repeat visits |
| Unique visitors | Number of distinct visitor cookies with at least one visit in the range |
| Link click | One beacon sent when a viewer opens a link in a stack |

### How it's collected

- **Visitor cookie.** A first-party cookie holding a random ID, kept for one year. It isn't shared with other sites.
- **Regional consent.** Ask permission only where required for the chosen analytics setup, including applicable EU/EEA and UK requirements. Region rules must be verified before launch rather than assuming EU-only coverage. Remember the choice and offer Cookie preferences. Unknown location follows the consent-required path. Before acceptance or after refusal, no analytics identifier is set where consent is required. Non-identifying views and clicks are counted only where permitted.
- **Measurement coverage.** Explain that unique visitors cover measurable browsers, not every person; one person using multiple devices may count more than once.
- **Link clicks.** When a viewer opens a link, `navigator.sendBeacon` records the URL. The viewer goes straight to the destination, with no redirect.
- **Excluding the creator.** Nothing is recorded when the signed-in owner views their own stack or page. The owner's browser also gets an "owner device" flag at sign-in, so their signed-out visits on that browser aren't counted either.
- **Bots.** Requests from known crawlers and preview bots are ignored.
- Events are accepted only for stacks the viewer can open.

### Storage

- Raw events are retained for 90 days with a scheduled cleanup. V1 queries those events for the 7/30-day ranges; no indefinite daily rollup is required. Unique visitors use distinct IDs across the whole selected range, never the sum of daily unique counts.
- Republishing and privacy changes keep a stack's analytics.
- Deleting a stack deletes its analytics.
- Clicks are counted by normalized URL within a stack. The same URL twice is combined; a changed URL starts from zero. Previously clicked URLs remain visible within the selected range after removal from the content.

### Failure cases

1. The owner's visits are counted.
2. An analytics cookie is set before consent where consent is required, or a remembered refusal is ignored.
3. Events are recorded for a private, deleted or revoked stack.
4. One creator sees another's analytics.

### Expected results

- A creator can see which stacks and links get attention in the last 7 or 30 days, including clicks on URLs since removed.
- Numbers update within a few minutes. Unique counts deduplicate across the selected range.

### Launch verification

Verify the regional consent configuration for the chosen analytics implementation; do not display a prompt to every viewer by default.

---

## Change handle

### Entry point

**Handle** in Settings → Edit profile.

### Flow

1. The creator types a new handle. Availability is checked live: format, taken or retired.
2. A confirmation explains: "Your page moves to /{new}. Links using @{old} will redirect. You can't change your handle again until {date 30 days from now}. No one can ever use @{old}."
3. On confirm, the handle changes immediately.

### Rules

1. Format: lowercase letters, numbers, periods and underscores, 2–30 characters.
2. At most one change every 30 days. Before then, the field shows when the next change is allowed.
3. The old handle is reserved forever. No one, including the same creator later, can claim it.
4. `/{old handle}` redirects permanently to `/{current handle}`, including after several changes.
5. Stack links keep working because they're looked up by their code; outdated handles in them redirect.

### Failure cases

1. A second change within 30 days succeeds.
2. Another creator claims a retired handle.
3. An old link to the creator page or a stack stops working.

### Expected results

- Every link the creator has ever posted keeps working after any number of handle changes.

---

## Sign up, sign in and sign out

Adapted from Stack (`web/src/components/AuthSheet.tsx`, `web/src/app/auth/*`, `web/supabase/templates/*`, Stack-App `docs/user-flows/auth.md`).

### Entry points

- **Get started** or **Sign in** on Louie's home page.
- Opening `/create` or any creator-only page while signed out.

### Sign up

1. The creator enters name, handle, email and password.
2. The handle is checked live: format, taken, or retired.
3. Cloudflare Turnstile runs in the background.
4. A 6-digit code is emailed. Entering it confirms the account.
5. The creator lands where they were heading, or on their home page.

### Sign in

1. Email or handle, plus password.
2. The creator returns to the action that asked them to sign in.

**Forgot password** emails a 6-digit code, then the creator sets a new password.

### Sign out

**Sign out** in Settings. Only in Settings, unlike Stack, which also shows it under Edit profile.

### Rules

1. Anyone can create a creator account.
2. Viewers never need an account to read stacks or subscribe.
3. A retired handle can't be chosen at sign-up.
4. Sign-up and sign-in are rate-limited.
5. Sign-in emails (codes and password resets) go through Resend, set up as Supabase Auth's custom SMTP, so every Louie email comes from the same sender. Supabase's built-in email service isn't meant for production.

### Failure cases

1. Wrong password, expired code, email or handle already taken: clear message, and the form keeps what was typed.
2. Losing connection mid-sign-up: retry without creating two accounts.
3. A signed-out visitor reaches a creator-only page.

### Expected results

- Exactly one account per sign-up.
- The session survives reloads until sign-out or expiry.
- After sign-out, creator pages require signing in again.

---

## Abuse protection, reporting and takedown

A link-in-bio product with open sign-up and outbound links is an obvious target for phishing and spam. These protections ship in the first release. Based on Stack's safety notes (`13. SAFETY.md`, Phase 1).

### Automatic protection

| Protection | Where |
| --- | --- |
| Link safety check with Google Web Risk (the commercial version of Safe Browsing) | Every link in a stack at publish and republish, and every profile link on save |
| Short-link expansion: follow redirects on the server (up to 5) and check the final destination | Same |
| `rel="nofollow noopener noreferrer ugc"` on every outbound link | Stack pages and creator pages |
| Cloudflare Turnstile | Sign-up, the subscription form and the report form |
| Rate limits | Sign-in, sign-up, publishing, subscriptions, reports and code lookups |

An unsafe link blocks publishing or saving with: "We can't publish this link because it's flagged as unsafe: {link}."

### Viewer: report

1. **Report** in the menu on a stack page or creator page.
2. Choose a reason: Malicious link, Spam, Adult or inappropriate, Other. Optional details and email.
3. "Thanks. We'll review this."

### Admin: review and takedown

- Admins use a minimal protected manual workflow to inspect reports and take action. A dedicated moderation dashboard is outside v1. Access uses a server-side allowlist.
- Actions: **Disable stack**, **Suspend creator**, **Dismiss**, and reversing either takedown.
- A disabled stack is unavailable publicly. Its owner sees the reason in their private view and can inspect, download or delete it through owner tools; republishing and public sharing are blocked until restored.
- A suspended creator’s page and stacks are unavailable publicly. They can still sign in, inspect, download and delete owned content, but cannot publish, republish or share publicly until restored. Restoring access never reactivates already revoked links.
- Every admin action is logged with who did it and when.

### Rules

1. Reports never reveal the reporter to the creator.
2. Admin tools are never reachable by creators.

### Failure cases

1. A flagged link gets published.
2. A disabled stack is still reachable.
3. A bot creates accounts or subscriptions at volume.

### Later (not this release)

- Automatic image moderation, automatic quarantine after several reports, and periodic rechecks of already-published links.

---

## Delete account

Entry: Settings → Delete account. Explain that the profile, stacks, drafts, analytics and subscriber list will be permanently deleted and all viewer links stopped. Cancel keeps everything. Confirm requires a validated session; completion signs the creator out and invalidates sessions. Failure never reports completion and offers Retry. Owners can use this action while suspended.

Acceptance: content is unavailable on every public/owner surface after completion; subscriber addresses and tokens are removed; current and retired handles remain reserved but show unavailable; delayed saves cannot recreate any deleted work. External copies cannot be recalled.
