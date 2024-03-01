# Feature Name: version-matrix

## Summary

Rust projects depend on direct dependencies on other packages, indirect dependencies introduced by those packages, and the Rust toolchain itself.
As the range of versions a project wants to support increases, so does the complexity of doing so and the tenability of testing them.
The goal here is to settle on a version matrix strategy that is reasonable and would be widely applicable across Linebender projects.


## Motivation

### Testing ensures the quality of user experience

If we don't actually test the versions that we claim to support, then that will inevitably lead to situations where a user tries to use our project and something fails.
Having CI testing is already a longstanding tradition of Linebender projects, so the general idea of testing is not controversial.
I posit that we should test as many version combinations as reasonable and not claim to support anything beyond what we test.

### Using old versions of software generally leads to more issues

Although not always true, in most cases the newer version of a project has fewer issues per feature, including in terms of security.
Thus I posit that we don't want to support using software that has known issues when there is an improvement available.

### Contributor experience should be as frictionless as possible

No contributor is forever and as such making the experience pleasing for contributors will help with their retention.
This will have an especially large impact on new contributors making their first PR.
When they submit a simple PR we don't want them being hit with unrelated error messages in the CI.
Such a smooth experience will no doubt also be appreciated by veteran contributors.
Of the numerous times we've been hit by this, take for example the case where Raph had to start [debugging an unrelated CI error](https://github.com/linebender/vello/pull/401/commits/9861e506dc89924cb6feb30610264f7266e467e3) during his numerical robustness work.
These scenarios ramp up in frequency very fast in our low traffic projects like Druid or Piet.
In cases where a [simple one line fix PR](https://github.com/linebender/piet/pull/561) could be accepted we're instead greeted with the task of having to deal with the accumulated decay of the last 6 months.

In similar vein it would be great if contributors could run various tests locally in a manner that approximates the CI, to cut down on surprising errors later.

### Historic build reproduction

Sometimes you want to check out an older revision of a project and compile it.
For example if there is a newly found bug, but it was introduced a while ago and you want to debug it in the version with the least changes.
It would be very useful indeed to be able to compile this historic revision without having to start dealing with the fact that some unknown sub-dependency's newer version isn't compatible and causes unexpected further issues.

### We want to support more than just the latest stable Rust toolchain

Not everyone can or will upgrade their Rust toolchain the moment a new version is published.
This includes potential new contributors, to whom we want to let write their patch without having to start updating their toolchain.
Beyond that, it will be even more common among application developers who use our libraries.
It's a question of balance of course, but we want to support toolchains as old as possible.

### We want a widely applicable strategy

In the absence of a well thought out general strategy we will have (potentially ephemeral) ad hoc approaches for every distinct project.
This means an increased maintenance burden, as problems solved with one approach won't necessarily carry over to another.
Also there will be more problems to solve, because a mix of strategies will bring us the union of their respective problems.


## User-facing explanation

We test and support only the latest versions of all our package dependencies.

Our minimum supported Rust version (MSRV) is largely decided by our dependencies and we just document the fact.


## Implementation strategy

### Keep the existing general Linebender CI as a base and extend it

Our current general CI script is the result of years of experience.
It already has good techniques to avoid some sources of dependency contamination, e.g. feature activations caused by dev dependencies.
It also already pins the stable Rust toolchain version, allowing us to choose when to update it.
Those benefits should move forward and as such this RFC doesn't see the need to remove anything and all changes are additions.

### `Cargo.lock` is committed to source control

Having a one true `Cargo.lock` solves a lot of the contributor and CI issues.
Contributors can easily use the same dependency versions as the CI does.
The CI won't suddenly start producing errors just because a new dependency version was published that causes issues.
We get reproducible historic builds.
We are in control of the timing when dependency updates are introduced.

### `Cargo.lock` is updated regularly

Given that a `Cargo.lock` of a library is used only by the developers of said library, we should not get comfortable with one set of versions for too long.
`Cargo.lock` should be updated to contain the most recent versions of dependencies on a regular basis.
Roughly as often as we update the stable Rust toolchain pin in the CI script.
Waiting longer than that creates too large of a time window for bad user experience issues to remain hidden.
Updating it too often, e.g. every week just because, will cut down on the benefits of contributor's local environment matching the CI and increase merge conflicts.
Critically, an update should always happen just before publishing to ensure that we test the unlocked experience that users will have.

### `Cargo.toml` matches `Cargo.lock`

We don't want to claim that our project works with dependency versions that we don't actually test.
Thus we want to make sure that our `Cargo.toml` has the minimum versions specified exactly as we test them with `Cargo.lock`.

An easy way to do this is to use the `cargo upgrade` tool available via [`cargo-edit`](https://crates.io/crates/cargo-edit).

```sh
cargo install cargo-edit
```

Running the following command will update all the `Cargo.toml` files in the workspace and also `Cargo.lock`, in a SemVer compatible way.

```sh
cargo upgrade
```

### `rust-version` (MSRV) is a formalized entry of de facto reality

Instead of striving for a specific MSRV target we just specify the oldest Rust toolchain that is capable of dealing with our code and its dependency tree.
Projects with few dependencies like Kurbo can continue to enjoy a relatively old MSRV, while larger projects like Vello will specify a much newer MSRV due to the fast moving dependencies demanding it.

For now the only way to know when this needs to be bumped is by failing a compilation attempt with the MSRV toolchain, either locally or via CI.
In the future Cargo will receive additional powers to inform us more directly under some circumstances.

Additionally, we will accept reasonable PRs even if they contain code which directly uses Rust features that require the MSRV to be bumped all the way to stable.
What exactly *reasonable* means will be left undefined by this RFC and will remain at the discretion of reviewers, as has been the case thus far.

### MSRV changes are non-breaking

We join the larger Rust community in their choice of declaring MSRV changes as non-breaking.
Currently this can cause some pain due to the surprise compilation failure, though less than a breaking version would with its manual opt-in wheel of churn.
In the future Cargo will be smarter and prevent most of this pain for our users.

### CI gets an additional MSRV job

In addition to the existing stable Rust toolchain jobs we also add a cut down copy as a MSRV job.
Specifically we need to avoid doing a Clippy pass as different Clippy versions disagree on goals and would result in a deadlock.
The MSRV job only needs to verify the crates that need to satisfy the MSRV.
Which is to say that the MSRV job doesn't need to verify examples, tests, benchmarks, or various auxiliary packages in the workspace.
Crucially the MSRV job can use our one true `Cargo.lock`.


## Drawbacks

Committing `Cargo.lock` will generate a bunch of noise and more importantly it will be an everlasting source of merge conflicts.

On the noise front, it will greatly inflate the *lines changed* stats on PRs.
I know I personally use that stat to determine whether to quickly review or postpone.
In a world with `Cargo.lock` I will need to click deeper to see just how much `Cargo.lock` inflated that stat.
Then that `Cargo.lock` change should at least be skimmed to see that nothing obviously incorrect is happening, increasing the review burden.

Any time someone adds, removes, or updates a dependency we will get a `Cargo.lock` change.
Having multiple of these changes in flight concurrently is pretty much guaranteed to cause a merge conflict.

Not supporting ancient Rust toolchains as our MSRV deprives people who depend on their OS (Debian, Red Hat) to supply their toolchain.

Not treating MSRV bumps as breaking changes will cause surprise compilation failures for the few people on very old toolchains.
That is until already planned Cargo changes land in the future and trickle down to these people, preventing those failures.

Not supporting old untested dependencies can cause issues for users whose app has some `misguided_dependency` which either pins `sub_dependency` to a specific version, or sets an upper limit to its version.
Well now if our project requires the latest version of `sub_dependency` then Cargo will fail to compile at all.
So supporting a wider range of versions would allow our packages to co-exist with misguided packages for longer periods of time.


## Rationale and alternatives

### Deciding how to handle the dependency version matrix

I've been spending a lot of time thinking and researching how to improve the CI experience.
Although the CI itself is just the messenger, so the focus is actually more on how to handle dependencies in a robust way.

For Druid we had a fairly straightforward strategy.
We only supported the latest versions of everything.
It worked well enough, although not perfectly.
Last year we started [manually specifying the stable Rust toolchain version](https://github.com/linebender/glazier/pull/76) in the CI to prevent sudden breakage.
That has been a clear improvement over having the toolchain version unlocked.
We no longer have an automatically broken CI every 6 weeks due to Clippy changes.
Following that concept I've been looking into the idea of also locking other dependencies.

Adjacent to all of this is the question of old versions.
Should we support them, how far back, and why?

### The basics

Rust projects depend on a specified version range of other Rust projects.
These other projects include direct dependencies on other packages, indirect dependencies introduced by those packages, and the Rust toolchain itself.

The simplest thing to do here would be to only depend on the latest stable version of all of these projects.
That would mean there is only one very well tested dependency tree and anything older is unsupported.
However there are some reasons to also support older versions.
The latest versions might behave in ways that a specific user might want to avoid.
In the case of the toolchain the user might not have gotten around to updating just yet, or they might be using an older toolchain version provided to them by someone else, e.g. their administrator or OS.

Unfortunately properly supporting older versions is difficult for a myriad of reasons.
As you increase the supported version range of some dependency, the chance of it being compatible with the ranges of all the other dependencies decreases.
To make matters worse the Rust toolchain provides very little to help with finding a compatible set.
There is [some progress](https://github.com/rust-lang/cargo/pull/12560) towards automating these tasks in the nightly version, but at this point it's at the level of temporary partial hacks and any stabilization is likely years away.
A lot of packages are also buggy in the sense that they provide incorrect minimum specifications for their sub-dependencies and never test them.
There has been some community work to clean up the ecosystem, but there's still plenty to do.
Specifically, when I last checked our stack in September 2023, we had a deep dependency on `gpu-allocator` which ended up bringing us an incorrect dependency specification that doesn't actually work.
Looking at the [relevant issue](https://github.com/Traverse-Research/gpu-allocator/pull/174) now, it doesn't look like it has been fully solved.
Functional tooling would certainly help propel that work forward.
Additionally, even if we know all the combinations that could work, we are not going to realistically be able to test them.
Not via the CI and even less so manually.
The problem space is just too large.
Thus the question becomes whether we can find a good balance point where we support more than just the latest versions but also actually test these support claims to some reasonable degree.

### Minimum supported rust version (MSRV)

The Rust toolchain itself is an interesting dependency.
We depend on it directly, but also indirectly through every other direct and indirect dependency.
We might choose some toolchain version range to support, but so will all our direct and indirect dependencies.
This creates an immense pressure towards stable.
Because it takes just one indirect dependency to bump their MSRV for it to cascade all the way to our project having an effective MSRV bump too.

Even if our project and all our direct and indirect dependencies at their latest versions work at our advertised MSRV at our release time, some dependency might bump their MSRV after our release.
Given that Cargo dependency resolution favors the latest versions and is not MSRV-aware, this broken MSRV experience will be the default for anyone adding our project as a dependency.

There is some work ongoing to add [MSRV-aware dependency resolution to Cargo](https://github.com/rust-lang/cargo/issues/9930) with a [prototype hack](https://github.com/rust-lang/cargo/pull/12560) being available in nightly.
However a proper solution in stable Rust seems years away.

> [We (the cargo team) are concerned that the different costs associated with making the resolver rust-version aware are more than the benefit and we put this in the pile of things waiting on the new resolver implementation which has stagnated.](https://github.com/time-rs/time/discussions/535#discussioncomment-4444626)
> -- epage

That said, there is a lot of active discussion going on in [rfc#3537](https://github.com/rust-lang/rfcs/pull/3537) and it seems to be on track to be accepted.
That RFC introduces the idea of a 3rd revision of the Cargo resolver that would be very accommodating to MSRV at the cost of making it harder to get the latest versions of dependencies.

Even so, until such work is completed, stabilized, and propagated enough to become our MSRV, it is only possible to guarantee MSRV by having the user craft their own application's `Cargo.lock` in a way that is compatible with MSRV.
Either manually or e.g. by using the nightly prototype hack.
(I do wonder how many instances there are where someone is stuck on an old Rust toolchain, but also has access to a nightly toolchain.)

Indeed even that Rust RFC states:

> The sooner we improve the status quo, the better, as it can take years for these changes to percolate out

#### How old are we talking?

The Rust team itself only supports the latest version.
Unlike other languages, there are no patches for older than the current stable version.

Given the mess that is MSRV-aware dependency resolution, it would be very difficult and painful to support a very old version unless the project has no dependencies, or just a few that have a compatible MSRV policy themselves.

Especially as the Cargo MSRV-aware resolver RFC also introduces a new default mechanism for `cargo new`.
Specifically that the `rust-version` property which defines a package's MSRV will be, by default, automatically set to the toolchain version used to publish it.
This will super accelerate the already massive pressure towards stable being the computed MSRV of a large dependency tree.

In the broader ecosystem we have extremely foundational projects like [`libc` considering adopting a MSRV policy as low as stable and one before that.](https://github.com/rust-lang/libs-team/issues/72)
Although the policy hasn't been finalized yet, seemingly pending Cargo improvements.

Others are moving forward with tightening their MSRV policies, e.g.:
* [`winit` has a stable minus three policy](https://github.com/rust-windowing/winit/tree/e41f0eabb1d0bd4583dce8035e86abaf38427a99?tab=readme-ov-file#msrv-policy)
* [`time` adopted a stable minus two policy](https://github.com/time-rs/time/discussions/535)
* [`once_cell` adopted a stable minus seven policy](https://github.com/matklad/once_cell/issues/201)
* [`tokio` has a 6 months policy, which is roughly stable minus four](https://github.com/tokio-rs/tokio/blob/1b8ebfcffb10beadda709ea4edfc1078a9897936/README.md#supported-rust-versions)

It's also worth noting that some users who want to use their OS provided Rust can be using versions that are years old.
I don't think catering to such outliers is worth the cost at this time in Rust's life.

Another interesting angle to consider is that if one of our dependencies has an important bug fix available only with a very recent MSRV, do we stubbornly delay the fix and/or invest the time into making a MSRV-compatible fork until our own MSRV policy allows for direct dependence?

Receiving the latest bug fixes and features from our dependencies has more value than providing ancient Rust toolchain support for a small minority.

#### Are MSRV increases a SemVer breaking change?

Previously I have leaned towards considering increasing MSRV breaking, however this is swimming against the stream.
The Rust ecosystem as a whole is leaning towards the opposite.
Combine that with how Cargo resolves dependencies and the user experience will still be one of MSRV change.

> [Yeah I'm very opposed to bumping semver for an MSRV bump.](https://github.com/rust-lang/libs-team/issues/72#issuecomment-1192890449)
> -- BurntSushi

> [Treating MSRV bumps as semver breaking changes is absolutely untenable.](https://github.com/matklad/once_cell/issues/201#issuecomment-1260615454)
> -- BurntSushi

There is [some discussion over this at the Rust API guidelines forum.](https://github.com/rust-lang/api-guidelines/discussions/231)
Although a bunch of this discussion has happened in other random threads (*including the two linked earlier via the quotes*) and that API guidelines thread is more of a summary.

The most interesting argument made is that considering MSRV bumps as breaking will harm people on older toolchains.
Consider `deep_dep` bumping MSRV and then their own major version.
Then `middle_dep` will upgrade to this new major version of `deep_dep`.
Finally `user_app` that depends on `middle_dep` will be forced to adopt the `deep_dep` MSRV if they want to get `middle_dep` updates.
However if `deep_dep` did not increase their major version, then `middle_dep` doesn't need to update their `Cargo.toml`, and `user_app` can still manually craft a `Cargo.lock` that uses an old version of `deep_dep` with the latest `middle_dep`.

Another one is that MSRV bumps (which people want to do regularly) being breaking would mean immense stagnation of code quality, as bug fixes would no longer arrive with a simple `cargo update`.
People would have to manually upgrade, which is unlikely to happen as frequently as `cargo update`.

That thread and others contain some additional arguments too, but I posit that the most important argument is just the fact that a lot of people in the ecosystem are sold on this idea.
So our dependencies will casually bump MSRV whether we like it or not.

#### Lazy or disciplined?

Would we consistently increase `rust-version` based on our policy to prevent people getting surprised by the policy?
To make our contributors more comfortable with using new Rust features.
Or do we lazily increase `rust-version` when there is an actual new feature we want to use.
Which can make contributors argue over the triviality of the feature in situations where we haven't updated `rust-version` in a long while.
Think syntactic sugar to lose 6 months of support.

We could try to reduce the chance of arguments by making the policy clear that a contributor can bump `rust-version` in accordance with our MSRV policy regardless of the triviality of the feature.
Even so, there are two issues that spring to mind.

First, bumping MSRV in all the configuration files and documentation can be needlessly challenging for drive-by contributors who would otherwise submit a reasonable small patch that just happens to use a fairly recent Rust feature.
Not to mention that these drive-by contributors might not even know about our MSRV policy allowing them to do it without argument.

Second, the longer we let `rust-version` lag behind the MSRV policy, the larger the chance of CI breaking for unrelated PRs due to a dependency bumping their MSRV above the Rust toolchain version we use in our CI.
Although this second issue could be mitigated with `Cargo.lock`, which I'll cover a bit further down.

#### A green path forward

The more I researched and thought about MSRV, the more I started favoring the evergreen approach of Druid.
Having access to the latest stable toolchain brings a lot of benefits.
Both directly from the toolchain itself, and also by being able to effortlessly update our whole dependency tree.

We could still have a lazily updated `rust-version` in order to usually provide at least a little backwards compatibility, but with a clear policy that it might be bumped all the way to stable at any point.
Although as I wrote above, having `rust-version` lag behind will cause contributor friction because they need to know that they can even bump it and also how.
Then again if the policy allows for bumping to stable, then the MSRV CI job could be optional for PRs, allowing new contributors to ignore errors.
Perhaps even better if there's no MSRV CI job for PRs at all - to save resources and avoid confusion, and instead there's some sort of pre-release job to ensure that the `rust-version` field is correct for actual releases.

Such an aggressive evergreen policy wouldn't be too different than how most of our packages currently operate, i.e. without a policy and no MSRV tests. So in a sense it's just formalizing what we already do.

The policy could also be revisited once stable Rust receives a MSRV-aware dependency resolver or one of our packages reaches 1.0.

Additionally, more limited scope and foundational packages like `kurbo` could adopt a somewhat different policy.
Although like I mentioned earlier, even `libc` and `time` are aiming for quite evergreen approaches and it takes just one package in the whole dependency tree to bump the requirement for the whole application.

### Minimum supported packages

How many different versions of dependencies should we support and why would we even do so?
Using the latest versions have plenty of clear benefits.
They contain the most bug fixes, have more features, and match nicely with the Cargo design principle of always preferring the latest - so tools work well.
On the other hand the latest versions might behave in ways that a specific user might want to avoid.

It's also possible that the user's app has some `misguided_dependency` which either pins `sub_dependency` to a specific version, or sets an upper limit to its version.
Something that a library should never do, but sometimes does anyway, e.g. as a hack to ensure MSRV compatibility.
Well now if our project requires the latest version of `sub_dependency` then Cargo will fail to compile at all.
So supporting a wider range of versions allows our packages to co-exist with misguided packages for longer periods of time.

These reasons aren't super strong motivation to support older packages.
Even so, if the cost of supporting older versions isn't too much it might be worth doing.

#### Making sure what we claim to support actually works

It's one thing to claim in `Cargo.toml` that some old version of a dependency is sufficient.
That's common in the ecosystem and as life has shown, often incorrect.
If we were to claim support for old versions then we need to also test that.
At least on the CI level, i.e. automated tests.
That would at least guarantee that the combination will compile and not behave incorrectly in some ways.
However there is a ton of stuff that automated tests will never cover.
Especially in the GUI domain, where so much behavior is hard to automatically test but comparatively easy to see as a human.
Human testing is in extremely limited supply and thus highly unlikely to scale to many different dependency version combinations.

Okay, but what would it take to test multiple combinations in CI?
Well this is an exercise in cutting scope.
It is completely unachievable to test every actual combination, especially if we claim to support like a 100 different patch releases of some dependency.

The following version matrix seems like a good starting point:

| MSRV toolchain   | `Minimum.lock` | `Current-MSRV-compatible.lock` | `Unlocked` |
|------------------|----------------|--------------------------------|------------|
| Stable toolchain | `Minimum.lock` | `Current.lock`                 | `Unlocked` |

Even that would mean a 6x increase in compilation configurations!
Not necessarily a linear increase in time due to parallelism and shared dependencies, but a big increase nonetheless.

Thus the question continues, can we significantly reduce this workload while still achieving similar test coverage?

Unlocked testing is the most important test for library users, as that is the scenario that new users will immediately encounter.
However users aren't necessarily depending on `main` and if they are, they can be warned about the potential issues.
Instead most users are depending on a specific release of a library.
Also, updating a past release is by definition a new release, so testing Unlocked dependencies for a release can mean no more recent than their state at the time of release.
Thus we can achieve the closest possible approximation of `Unlocked` test coverage by updating our `Current.lock` as a preparation step just before release.
Thus we can omit the `Unlocked` testing with both toolchains.

With the remaining four configurations I think quite a nice compromise would be to only test `Minimum.lock` with the MSRV toolchain, and only `Current.lock` with the stable toolchain.
If `Minimum.lock` works with the older MSRV toolchain, then there is an extremely high chance that it will also work with the stable toolchain.
Skipping `Current-MSRV-compatible.lock` is a somewhat larger sacrifice as we don't really have a proxy to guess its success rate.
However testing `Minimum.lock` with MSRV would be guaranteeing that there is some configuration that works with MSRV.
`Current.lock` wouldn't probably work anyway (as some dependency probably has a more recent MSRV) and so not testing `Current-MSRV-compatible.lock` would basically mean that we don't determine the freshest dependency tree that works with MSRV, only promising that `Minimum.lock` works and anything beyond that is untested.

Doing the above, we would end up with a much simpler test scenario.
`Current.lock` with the stable toolchain and `Minimum.lock` with the MSRV toolchain.

#### Compiling with minimum dependency versions is possible only when stars align

Cargo always favors the most recent versions.
There has been some work done for the nightly toolchain to generate a `Cargo.lock` with minimal versions.
Unfortunately there are now two different attempts at it and neither of them actually work.

The initial attempt was the [`minimal-versions` flag](https://doc.rust-lang.org/cargo/reference/unstable.html#minimal-versions) which generates `Cargo.lock` with the minimum versions specified by direct and indirect dependencies.
Seems great on the surface, but it runs into two big issues.

The first issue is that a lot of projects don't have correct minimal specifications to begin with, e.g. they added `foo = "0.1"` to `Cargo.toml` when in reality they depend on `foo = "0.1.2"`.
Even those that are initially precise, tend to not actually test the minimal versions and so with time they end up depending on newer features without updating the minimum requirement.
Cargo behavior makes sure that the issue won't be caught.

The second issue is both more subtle and more fundamental.
There can only be a single `Cargo.lock` per workspace.
That means that if `package_a` depends on `foo = "0.1.1"` and `package_b` depends on `foo = "0.1.2"` then even with `-Z minimal-versions` you will end up with a `Cargo.lock` that has `foo = "0.1.2"`.
What's more, a similar issue exists even with a single package.
Cargo will generate `Cargo.lock` with both dev dependencies and `--all-features` enabled and there is no way to disable this.
That means once again that the package itself might specify `foo = "0.1.1"`, but if a dev dependency or a feature either increases the `foo` requirement directly to `0.1.2` or adds a dependency that indirectly requires `foo = "0.1.2"`, well then once again `Cargo.lock` has `foo = "0.1.2"` even though the package directly specified a lower version.
Note that you can't manually override this to a lower version even with `cargo update -p foo --precise 0.1.1`.
When a user will depend on the library they won't have the library's dev dependencies enabled, and probably also won't enable all the features.
Which means that if they use `-Z minimal-versions` with their own app they will get `Cargo.lock` with `foo = "0.1.1"` that was never tested and is broken.

Later there was a second attempt by adding the [`direct-minimal-versions` flag](https://doc.rust-lang.org/cargo/reference/unstable.html#direct-minimal-versions) which generates `Cargo.lock` with the minimum versions specified for direct dependencies but maximum versions for indirect dependencies.
The idea being to deal with the problem of the ecosystem being full of packages that have incorrect minimum specifications, which was the first issue I described for `minimal-versions`.
Unfortunately it doesn't sufficiently address the second more fundamental issue.
This flag has a new restriction that all packages in a workspace need to manually specify the exact same version of matching dependencies.
So if you have `package_a` depend on `foo = "0.1.1"` and `package_b` depend on `foo = "0.1.2"`, well then `-Z direct-minimal-versions` will just error out.
That is an unfortunate limitation because `package_a` could easily be fully functional with just `foo = "0.1.1"`, but at least it removes one surprising case of `Cargo.lock` having `foo = "0.1.2"`, as would happen with `-Z minimal-versions`.
However the issue remains with dev dependencies and optional features.

To summarize, Cargo has been designed to always prefer maximum versions of dependencies and despite multiple attempts to add support for compiling with minimum versions, none of them have been successful, unless you have a really simple package.

Doing minimum version testing sounds like a good idea in theory.
In reality, in Rust land, it's simply not possible with reasonable effort.
(One could do all testing in a separate non-workspace package with zero dependencies other than the test target, which would ensure that neither the workspace nor dev dependencies ruin the `Cargo.lock` generation.
However I think the maintenance burden, loss of development ergonomics, and confusion among new contributors caused by such an effort goes way beyond what the benefits can justify.)

Thus the idea of having a `Minimum.lock` is not really practical.
It might work for smaller projects, but as the workspace size increases so does the probability of `Minimum.lock` being wrong.
That is on top of the fact that generating such a `Minimum.lock` would at a minimum require contributors to fumble around with unstable features of the nightly toolchain, which is a lot of extra friction.

#### Should we commit `Cargo.lock`?

Multiple different `Cargo.lock` variants is a no-go, but what about a simple single `Cargo.lock`?

The `Cargo.lock` in question would aim to mimic the unlocked experience while preventing sudden CI breakage due to dependency updates.
It's essentially the same strategy as manually defining the stable Rust toolchain version in the CI, which has proven successful for us.
With this lock file, we don't want to be behind the update curve too much.
Instead, we want to choose when to deal with the issues that come with the updates.

The upside of having `Cargo.lock` committed would be to help prevent unexpected CI breakage.
Also local development testing would more closely match the CI environment.

The downside would be merge conflicts with branches that modify dependency requirements.
Also PRs with `Cargo.lock` changes would be noisier and with an inflated number of lines changed.
Not exactly an insurmountable issue, but does remove the usefulness of the line change summary metric.

This is also a question of what to optimize for?
No `Cargo.lock` prevents merge conflicts between branches that modify dependencies, but having the lock file prevents CI breakage due to a bad dependency update.
Which is more common, concurrent PRs that update dependencies or problematic dependency releases?
Likelyhood of both will increase with project growth.

Generally we want to incentivize smaller PRs over giant ones, because they are easier to review and thus much more likely to get reviewed in a timely manner.
Smaller PRs with their shorter review cycles also mean that there is a smaller window of opportunity for multiple branches with dependency updates to appear.
Thus we have some control over reducing the chance of the concurrent PR issue.
Bad dependency updates are much more out of our control.

Pinning the Rust toolchain in our CI is very ephemeral without also having `Cargo.lock`.
Because if a dependency bumps their MSRV beyond our pinned version, our CI breaks.
We like the benefits of pinning the toolchain and choosing our own update schedule.
`Cargo.lock` would make sure that the choice remains in our hands.

Committing `Cargo.lock` gives better build reproduction, so it's possible to go back in time to a historic revision and see what Rust toolchain version succeeded, and have the correct `Cargo.lock` ready to reproduce that success.

##### When to update `Cargo.lock`?

What's a good choice for the update cadence?
One logical choice would be to combine it with the Rust toolchain update every 6 weeks.
Additionally, an update should occur just before releasing a package to verify that the unlocked experience is not broken.

Also, when I say _update_ I basically mean `cargo update` which will only perform a SemVer compatible update.
Major version updates are a separate issue and should continue to be done on a case-by-case basis.

In a related note, `Cargo.lock` will get updated outside of the regular schedule whenever the dependency requirements in `Cargo.toml` get updated.
For example when we specify that we need a specific bugfix patch of a dependency.

### Putting it all together

We can get a lot of benefits with a fairly simple strategy if we're willing to sacrifice support for a good chunk of old versions of our dependencies.

A single `Cargo.lock` that also matches our `Cargo.toml`.
No fancy MSRV policy to fight for, but instead just documenting the facts of reality.
The oldest toolchain version that can compile our code plus our dependencies.
Which can actually end up up being a lot older for simpler packages than a more conventional calendar based strategy would provide.


## Future possibilities

Once Rust [rfc#3537](https://github.com/rust-lang/rfcs/pull/3537) gets implemented and stabilized we'll have a much more capable MSRV aware Cargo with its resolver at version 3.
At that point it might be worth considering adopting a longer lasting MSRV policy, or maybe having an LTS release of our projects for such a purpose.
As such an improved Cargo would finally make it much less painful to compile things with an old toolchain, due to its ability to generate a working `Cargo.lock`.
Contrast that with the current situation where any old toolchain user would have to manually craft such a lock file.

At that point it will also become important that all packages inside a single workspace have the same MSRV.
Because the proposed way that the new resolver works is by resolving dependency versions so that they work with the oldest MSRV in the workspace.
That means that if packages in a workspace share dependencies, then the package with the older MSRV will prevent the package with the newer MSRV from receiving dependency updates it otherwise would receive.
Critically this is the behavior only for the developers in the workspace.
As external users of the package wouldn't be affected by the workspace dynamics.
Which would mean that the developers / CI would never be able to test the dependency tree that actual users experience.
