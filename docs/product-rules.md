# Project Louie — Product rules

Status: Confirmed product rules. Implementation details requiring further specification are listed at the end.

## Visibility and publishing state

| Concept | Meaning |
| --- | --- |
| Public | Accessible to anyone and displayed on the creator's public page |
| Unlisted | Accessible to anyone with a valid link, but omitted from the creator's public page |
| Private | Accessible only to the creator |
| Draft | Work that has not been published; separate from the privacy setting |

Anonymous viewers can view creator public profiles and public stacks. They can also view unlisted stacks with a valid link. Private stacks and drafts are owner-only.

## Creation and content limits

- Title and description steps can be skipped.
- Publishing requires a nonempty title; a description is optional.
- A saved draft must contain at least one nonempty element: a title, description, or body line item.
- Completely blank drafts cannot be saved.
- Each stack has a total limit of 6,000 characters. Detailed field limits and character counting rules must be specified explicitly before implementation.

## Editing and republishing

- Each published stack has one shared edit draft, separate from the published version.
- Editing the same stack in multiple tabs operates on that shared draft rather than creating competing edit drafts.
- A stale tab attempting to save receives a conflict prompt; it must not silently overwrite newer saved work.
- The edit draft remains until the creator publishes it or deletes it.
- Viewers continue seeing the existing published version until the creator republishes.
- Republishing replaces the existing published version; it does not create an additional published stack.
- Deleting an edit draft leaves the existing published stack unchanged.

## Privacy changes and links

- Changing a stack from public to unlisted removes it from the public creator page, invalidates its old public link, and generates a new unlisted link.
- Anyone with the new unlisted link can view the unlisted stack.
- Changing a stack from public to private blocks viewer access through the old link. The owner retains access to the private stack.
- A private stack link remains accessible only to the owner.

## Sharing and exports

- Shared images or exports can be generated from any type of stack, including public, unlisted, private, and draft stacks.
- Generated images can be shared externally or saved as pictures.
- Exporting a private stack does not grant viewers access to its stack link.
- An exported image is a snapshot; changing or deleting the source stack cannot revoke copies already shared externally.

## Deleting a published stack

- Deletion removes only the published version.
- Its old published link stops working and remains inactive.
- Its associated edit draft survives with its saved content.
- Historical analytics remain attached to the deleted publication.
- If the creator publishes the surviving draft, they must choose its privacy setting.
- Publishing that surviving draft creates a new published stack with a new link and fresh analytics.
- The new publication does not reactivate the deleted publication's link or combine its new analytics with historical analytics.

Deletion confirmation:

> Delete this published stack? Its link will stop working. Your edit draft and historical analytics will remain.

The confirmation text must reflect whether an edit draft actually exists.

## Analytics

- Analytics is owner-only and accessed through Settings.
- Initial metrics include total visitors, unique visitors, and link clicks.
- Historical analytics remain after deletion of a published stack.
- A new publication created from a surviving draft starts with fresh analytics.

## Viewer email subscriptions

- Viewers can subscribe to a creator's email list.
- This is distinct from creator account signup.
- This capability is included in the initial release product scope.
- Form placement, subscription management, and email delivery or export integrations will be specified separately.

## Details to define in supporting specifications

- Privacy transitions other than public → unlisted and public → private, including whether previously revoked links ever reactivate.
- The conflict prompt and how creators resolve edits from stale tabs.
- Character counting rules, treatment of link URLs, and individual field limits.
- Visitor and unique visitor definitions, reporting periods, owner and bot traffic, and link click attribution.
- Where creators access analytics for deleted publications.
- The email subscription flow, list management, and any integrations.
