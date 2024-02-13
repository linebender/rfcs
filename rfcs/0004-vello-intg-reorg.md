# Feature Name: `vello-intg-reorg`

## Summary

This RFC has a simple, targeted goal to move integrations (`vello/integrations/vello_svg`) out of the `vello` crate, similar to `velato`. This would be implemented **after** 1.0 of vello.

In addition, I propose deprecation of `vello_svg` and `velato` in favor of bringing [`vello-svg`](https://github.com/vectorgameexperts/vello-svg) and [`vellottie`](https://github.com/vectorgameexperts/vellottie) into the GitHub linebender organization as the official integrations.

Lastly, the vello docs would make a mention of integrations and where to find them in their respective repos.

## Motivation

- Simplify the publish cycle for `vello` - Only 1 crate (vello) to crates.io in CI/CD.
- Standard treatment of integrations (currently we have vello_svg in the vello crate, but velato as a standalone)
- Keep PRs focused, rather than needing changes to `vello_svg`

In favor of deprecating `vello_svg` and `velato` for `vello-svg` and `vellottie`:

- vello-svg is a 1:1 mirror, but has additions like a [nursery](https://vectorgameexperts.github.io/vello-svg/) and tests.
- vellottie is nearly a 1:1 mirror with velato's runtime, except it might have some cleanup and rendering fixes. It has 2 parsers, a fast, new serde parser, and a step-through parser used on parse error to identify lottie file issues with granularity. It also has a [nursery](https://vectorgameexperts.github.io/vellottie) and tests.

## User-facing explanation

- Need SVG? Go to the vello-svg README.
- Need Lottie? Go to the vellottie README.

## Implementation strategy

Tasks:

- Wait for [the vello 0.1 publish](https://github.com/linebender/vello/issues/302) - Vello should not publish `vello_svg`.
- Option 1: Deprecate `vello_svg` and `velato`
  - Move `vello-svg` and `vellottie` into the linebender organization
  - Pin `vello-svg` and `vellottie` to vello 0.1 instead of git refs in the Cargo.toml.
- Option 2: Ignore forks
  - Build up `vello_svg` and `velato` to parity (this could take longer).
  - Pin `vello-svg` and `velato` to vello 0.1 instead of git refs in the Cargo.toml.
- Publish `vello-svg` and `vellottie`/`velato`.
- Update the `vello` docs to reflect the official integrations

## Drawbacks

Can't think of any

## Rationale and alternatives

- The impact of not doing this is that we drag along a fragmented ecosystem of forks and integrations around vello, which isn't what we want prior to a 0.1 launch. We want a unified vision of where integrations will live, how we can create new integrations (e.g. "vello-pdf")

## \[Optional\] Prior art

N/A

## Unresolved questions

- N/A

## \[Optional\] Future possibilities

N/A
