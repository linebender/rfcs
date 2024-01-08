# rfcs

This repository holds suggestions for major changes to the Linebender ecosystem, notably Xilem, Glazier and Vello. Some of the wording is lifted from the Bevy project.

RFCs (requests for comments) provide a way for contributors to propose design changes in a specific way. Creating a RFC starts a collaborative process and invites community members to offer feedback and suggest changes or alternative designs.

The process is loosely-defined for now; this repository has been created to document some of the large changes that are expected to occur starting 2024. Hopefully this README should be rewritten before the end of January 2024.

Unlike other process of the same name, RFCs in Linebender are expected to cover both external features and internal refactorings, at least at first.

Some rules of thumb for now:

- **Most changes don't need RFCs:** The majority of changes (bug fixes, small tweaks, and iterative improvements) should not go through the RFC process. Just use the normal contributing process in the matching repo.
- **RFCs are not feature requests:** RFCs are for developers who already have a specific design in mind (including technical details) and want feedback on it. If you want to request a feature, please create an issue in the matching repo.
- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features. Try to pick a specific feature slice or architectural change and describe it in as much detail as possible. Feel free to create multiple RFCs if you need multiple features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with very similar implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing features. It is ok to create multiple dependent RFCs, but they should either be merged at the same time or have a clear merge order that ensures the "implementable" rule is respected. (Note: This one may be relaxed for the first few months.)
- **Don't create RFCs before you have a design:** If you want to explore design spaces with the Linebender community, consider finding or creating a discussion on [the Zulip server](https://xi.zulipchat.com/). If at any point during the discussion you discover a design you believe in enough to bring to completion, create an RFC. You don't need to have all of the details sorted out ahead of time, but Draft RFCs (RFCs created as "Draft PRs" on Github) should have more than just a few sentences describing the feature. An initial Draft RFC should at a bare minimum describe the design and goals from a high level, draw out a first draft of the public apis, and prescribe a good portion of the technical details. RFCs are a platform for community members to share detailed technical designs for major changes, not for people to open discussions with the intent to eventually find a design.

If you are uncertain if you should create an RFC for your change, don't hesitate to ask us in the [#general stream](https://xi.zulipchat.com/#narrow/stream/147921-general) on the Zulip server.


## Why create an RFC?

RFCs are intended to be a tool for collaboration, not a burden for contributors.

RFCs protect Linebender contributors from wasting time implementing features that never had a chance of getting merged. This could be due to things like misalignment with project vision, missing a key requirement, forgetting a technical detail, or failing to consider alternative designs.

RFCs also serve as a form of documentation. They describe how a feature should work and why it should work that way. They serve as a trace of how the project got to its current state. The accompanying pull request(s) record how the Linebender community came to that conclusion.

They don't need to be perfect, complete, or even very good when you submit them. The goal is to move the discussion into a format where we can give each part of the design the focus it deserves in a collaborative fashion.


## The Process

- Fork this repository and create a new branch for your new RFC.
- Copy 0000-template.md into the rfcs folder and rename it to 0000-my-feature.md, where my-feature is a unique identifier for your feature.
- Fill out the RFC template with your vision for my-feature. For now, the structure is somewhat loose, and you can skip sections if you feel they don't apply.
- Create a pull request in this repo. The first comment should include:
    - A one-sentence description of what the RFC is about.
    - A link to the "rendered" form of the my-feature.md file. To do so, link directly to the file on your own branch, so then the link stays up to date as the file's contents changes. See #1 for an example of what this looks like.
- With your PR created, note the PR number assigned to your RFC. Rename your file to PR_NUMBER-my-feature.md (ex: 0007-foo-bar.md), and make sure to update the link to your rendered file.
- Help us discuss and refine the RFC. Linebender contributors will leave comments and suggestions. Ideally at some point relative consensus will be reached. Your RFC is "accepted" if your pull request is merged. If your RFC is accepted, move on to the next step. A closed RFC indicates that the design cannot be accepted in its current form.
- Linebender contributors are now free to implement (or resume implementing) the RFC in a PR in the matching repo, and an associated tracking issue is created in that repo. You are not required to provide an implementation for your RFC, nor are you entitled to be the one that implements the RFC.


## Collaborating

First, make sure you always abide by [the rust code of conduct](https://www.rust-lang.org/policies/code-of-conduct) when participating in the RFC process.

Additionally, here are some suggestions to help make collaborating on RFCs easier:

- The [insert a suggestion](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/reviewing-changes-in-pull-requests/commenting-on-a-pull-request#adding-line-comments-to-a-pull-request) feature of GitHub is extremely convenient for making and accepting quick changes.
- If you want to make significant changes to someone else's RFC, consider creating a pull request in their fork/branch. This gives them a chance to review the changes. If they merge them, your commits will show up in the original RFC PR (and they will retain your authorship).
- Try to have design discussions inside the RFC PR. If any significant discussion happens in other repositories or communities, leave a comment with a link to it in the RFC PR (ideally with a summary of the discussion).
