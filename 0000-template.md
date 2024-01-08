**Note:** This process is loosely defined for now. When creating your RFC, feel free to skip some sections if you don't think they apply.

# Feature Name: (fill me in with a unique ident, `my-awesome-feature`)

## Summary

One paragraph explanation of the feature.

## Motivation

Why are we doing this? What use cases does it support?

## User-facing explanation

Explain the proposal as if it was already implemented and you were teaching it to a library user. That generally means:

- Introducing new named concepts.
- Explaining the feature, ideally through simple examples of solutions to concrete problems.
- Explaining how users should *think* about the feature, and how it should impact the way they code. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, explain how this feature compares to similar existing features, and in what situations the user would use each one.

## Implementation strategy

This is the technical portion of the RFC. Try to capture the broad implementation strategy, and then focus in on the tricky details so that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

When necessary, this section should return to the examples given in the previous section and explain the implementation details that make them work.

When writing this section be mindful of the following repo guidelines:

- **RFCs should be scoped:** Try to avoid creating RFCs for huge design spaces that span many features.
- **RFCs should avoid ambiguity:** Two developers implementing the same RFC should come up with nearly identical implementations.
- **RFCs should be "implementable":** Merged RFCs should only depend on features from other merged RFCs and existing features. RFC chains are allowed, but they should get merged in order.

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## \[Optional\] Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. In other similar processes, this often involves a list of links to other projects / libraries / programming languages implementing the same feature.

## Unresolved questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before the feature PR is merged?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## \[Optional\] Future possibilities

This is a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.

This section will often be less structured than the others.
