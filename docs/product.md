# Project Louie

Status: Product scope baseline. Remaining details are listed below.

## Product purpose

Louie is a streamlined version of Stack that serves as a link in bio destination for social media creators with substantial information and links to share. Creators organize that information into stacks, share individual stacks or their creator page, and measure viewer activity.

## Who Louie serves

Louie serves social media creators whose content benefits from an organized, reusable destination beyond a social post. Examples include:

- Travel creators sharing multiple itineraries.
- AI creators sharing multiple how-to guides.
- Art creators sharing event and gallery recommendations.

Viewers can read accessible stacks and browse a creator's public page without creating an account.

## Primary journeys

### Viewer

- Social media link → individual stack → creator's public page → another stack.
- Social media link → creator's public page → individual stack.

The second journey supports Louie becoming the creator's main link in bio destination.

### Creator

Creator's home page → create stack → publish → share → manage stacks from home page → measure activity through analytics.

The creator's home page has a toggle between public and private views. It defaults to the private view for the creator. Manage stacks is a separate screen accessible from the private view. Visitors see only the public page.

## Privacy and drafts

Stacks support public, unlisted, and private visibility:

- Public: Anyone can view the stack, and it appears on the creator's public page.
- Unlisted: Anyone with its link can view the stack, but it does not appear on the creator's public page.
- Private: Only the creator can view the stack.

Drafts are unpublished work, distinct from a stack's privacy setting. The creator can access drafts from the private view and manage stacks screen.

### Draft requirements

Creators can skip the title and description steps. A saved draft must contain at least one nonempty element: a title, a description, or a body line item. Completely blank drafts cannot be saved. A title is required before publishing; a description is optional.

### Editing published stacks

Editing a published stack creates a separate draft copy associated with that published stack. Edits stay in the draft until the creator publishes the changes. Viewers continue seeing the existing published version throughout editing. Publishing the draft replaces the existing published version; it does not create an additional published stack.

### Private stack exports

Creators can export private stacks as images and share those images externally. The stack itself remains private, and its link remains accessible only to the owner.

## Initial release screen inventory

### Individual stack

Viewers can read content and open links. A prompt invites them to see the creator's other stacks and directs them to the creator's public page. The prompt may appear at the bottom of the stack; its exact placement will be determined during design.

Access is subject to the stack's privacy setting.

### Creator's public page

Anyone can view this page. It includes the creator's profile picture, name, social media and website links, and all public stacks. It serves as a destination the creator can use as their main link in bio.

### Creator's private view

Only the creator can access this view, which is the default view of their home page. It includes public, private, and unlisted stacks, plus drafts. The creator can edit a stack directly from this view or open the separate manage stacks screen to quickly manage multiple stacks.

### Manage stacks

Only the creator can access this screen. All stacks and drafts appear in a list format.

- Published stacks have an Edit dropdown with actions to edit content, change privacy, or delete.
- Drafts have actions to edit or delete.

### Sign up / sign in

Creators use this screen to create an account or sign in to access creator tools.

### Create / edit stack flow

Creation follows five screens:

1. Title, which can be skipped.
2. Description, which can be skipped.
3. Body, with the ability to edit the title and description here as well.
4. Finalize.
5. Preview: a separate screen displaying the stack, with only Publish and Back as additional controls.

The detailed controls on the finalize screen will be defined in the create flow specification. Publishing requires a title. Saving a draft requires at least one nonempty title, description, or body line item.

Editing an existing stack begins at the body screen, where its title and description can also be edited.

### Analytics

Only the creator can access analytics, through Settings. Analytics covers individual stack activity and link clicks. Exact metrics will be defined in a separate specification.

### Sharing

Creators can share stacks as links or social media friendly images. Generated images can also be saved as pictures. Relevant sharing behavior and designs from the original Stack project will be reviewed for reuse.

## Outside the initial scope

- Comments.
- Reposts.
- Explore and discovery.
- Feeds.
- Followers and following.
- Paid or locked stacks, pricing, and commerce features.

## Reuse from Stack

Existing Stack designs can be reused or adapted where they support Louie's scope. Selected Stack behavior, particularly sharing, will be specified explicitly before implementation. Existing designs do not automatically establish Louie's product rules.

## Remaining details

- Whether search within a creator's public stacks is included at launch. Navigation from an individual stack to the creator's page is confirmed; a search feature is not yet confirmed.
- The exact controls on the finalize screen and placement of the prompt to view other stacks.
- Detailed sharing options, analytics metrics, and deletion outcomes.
- Detailed publishing behavior, including privacy changes and what happens if a published stack is deleted while an associated edit draft exists.

Detailed access rules, editing behavior, deletion outcomes, sharing options, and analytics definitions belong in the supporting product rules and flow specifications.
