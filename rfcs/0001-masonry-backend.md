# Feature Name: masonry-backend

## Summary

Use the codebase of the crate [masonry-rs](https://github.com/PoignardAzur/masonry-rs) as the new backend for Xilem.

## Motivation

Masonry and Xilem were both forked from Druid, and currently share a lot of code and design.

Masonry was written as an exploration into the concept of a GUI backend, with a Widget tree that could serve as a the retained component of a reactive architecture.

Xilem was written as an exploration into reactive GUI, and thus more effort was spent on its frontend and on how add developers would describe their interfaces.

As a result, Xilem's native backend is in a poor state:

- There is [code commented out](https://github.com/linebender/xilem/blob/ea45b9f8c14e3708f0fcbe0a0e1c760f59146323/src/widget/widget.rs#L113-L120).
- There are [*entire modules* commented out](https://github.com/linebender/xilem/blob/ea45b9f8c14e3708f0fcbe0a0e1c760f59146323/src/widget/mod.rs#L19-L20).
- There is [documentation referring to items from Druid that no longer exist](https://github.com/linebender/xilem/blob/ea45b9f8c14e3708f0fcbe0a0e1c760f59146323/src/widget/widget.rs#L51-L71).
- There are [TODOs without an associated issue](https://github.com/linebender/xilem/blob/ea45b9f8c14e3708f0fcbe0a0e1c760f59146323/src/widget/core.rs#L569C5-L569C66).

While Masonry's codebase hasn't been updated in a while, I made sure to address the above problems before I stepped away from it. I would also argue it has a tighter vision and more helpful documentation, thought that is more subjective.

Masonry also comes with some built-in perks, like powerful unit tests and a structured widget graph.

For these reasons, I think a new backend should be added to Xilem using either the Masonry crate or one forked from it. (We will refer to that new backend as "Masonry" in the rest of this RFC, although it might have a different name in the end.)


## User-facing explanation

### Xilem architecture

Xilem is roughly separated into two layers:

- **The reactive layer** handles the app logic and centralized state. Your application's code will call functions from this layer.
- **The platform layer** handles the document state, depending on the platform.

The reactive layer defines the `View` trait as well as other items for sequences, type-erasure and memoization. A `View` represents a transient description of one or multiple graphical items, which is usually returned by a declarative function that is often called a "component". An example of component could be:

```rust
fn app_logic(data: &mut u32) -> impl View<u32, (), Element = impl Widget> {
    Column::new((
        Button::new(format!("count: {}", data), |data| *data += 1),
        Button::new("reset", |data| *data = 0),
    ))
}
```

In that example, `app_logic()` returns a `Column` view which contains two `Button` views. These store the minimum information to represent the concept of two buttons stored in a column.

The `View` trait has an associated type `Element` which represents a persistent UI entity for the underlying platform (eg DOM node, Widget, etc). Every time app data changes, a new View is produced. That View is diffed against the app's previous View, and all changes are applied to matching elements: if the new view has a child view that wasn't in the previous version, a new element is added; if the new view no longer has a child view that was in the previous version, the matching element is removed, and so on.

(In web frameworks that use a virtual DOM, this process is sometimes known as "reconciliation".)

Xilem has multiple platform layers. `xilem_web` is based on the web DOM, whereas the main `xilem` crate is based on Masonry, which provides a native widget tree and some Developer Experience features.

Note that, while the platform layer is sometimes referred to as a "backend", it's somewhat tightly coupled with the reactive layer: `xilem_web`, `xilem` and any other platforms will each have their own version of the `View` trait, the `ViewSequence` trait, the `AnyView` trait, the `BoxedView` type, and so on.

### `xilem`

The element type in `xilem` is bound to the `masonry::Widget` trait. Different views defined in Xilem have matching Widget types defined in Masonry, though the correspondence isn't always one-to-one.

`Widget` in Masonry is a trait that represent units of graphical composition, things like text and buttons and boxes and containers. To implement Widget, a type must:

- Be `'static`.
- Provide info about its children.
- Handle user events.
- Compute its layout.
- Draw itself in a paint target.

See the [Widget trait definition](https://github.com/PoignardAzur/masonry-rs/blob/3cfc2cddec05d48a31a9812bdda27792d336f03f/src/widget/widget.rs#L65-L199) for more details.


### Directory structure

The `https://github.com/linebender/xilem` repository has the following crates in its `crates/` directory:

- `xilem_core/`
- `xilem_web/`
- `masonry/`

The main `xilem/` crate is defined at the project root.

## Implementation strategy

The implementation will require several steps:

- Refactor Masonry to use the same dependencies Xilem currently uses, especially Glazier, Vello, and Parley insted of druid-shell and Piet.
- Integrate AccessKit support.
- Move the Masonry codebase to the `xilem` repository, ideally in a way that preserves commit history.
- Port the list of Github Issues in the `masonry-rs` repository to the `xilem` repository.
- Create a temporary `xilem_masonry` crate, aiming for parity with the current Xilem crate.
- Once parity is reached, replace the current `src/` folder with the contents of `xilem_masonry`. Remove `xilem_masonry/`.
- Publish `xilem`.


## Drawbacks

### Layer separation

This move decouples the reactive layer of xilem from its platform layer by putting them in separate crates.

This means some features in `xilem` may be harder to implement because they won't have access to the backend's private code.

### Jumping to Masonry

By using the Masonry codebase, the Xilem project will move towards concepts Masonry was implementing that Xilem maintainers weren't necessarily considering for Xilem.

Some PRs intended for Xilem may become obsolete once the move to Masonry is complete, because Masonry will go in a direction that these PRs didn't account for.


## Rationale and alternatives

### Separating at the crate level vs the repository level

The initial suggestion was to use Masonry as it currently exists, as a separate repository.

However, people have brought up that it would make the Xilem development process a lot more complex. Features that reached across crates (`xilem_core`, `xilem`, `masonry`) would need to be rolled out in coordinated pull requests between repositories.

### Separating at the crate level vs the module level

Masonry and Xilem's code could simply be merged in a single crate, under different modules.

The consensus was mostly that having Masonry as a separate crate would help keep Xilem "honest" and enforce separation of concerns.

More importantly, Masonry was built as a potential foundation for *multiple* native GUI crates, and it would be valuable to conserve that; therefore, while Masonry would still depend on crates from the Linebender ecosystem, it shouldn't be too tied to Xilem.

### Porting Xilem to Masonry vs Masonry to Xilem

In any merge between two projects, one has to be the base the other is grafted on.

This RFC was written under the assumption that Masonry would be mostly unchanged and Xilem would be reworked to accomodate it, but it would also be possible to take changes from Masonry and graft them to Xilem's current backend code.

That said, the Motivation section explains why Masonry is a healthier codebase to start from.

## Unresolved questions

### Do we want to keep the Masonry brand?

The name Masonry was chosen when that crate wasn't officially part of the Linebender project. Even if we want the codebase and the crate to be added to the Linebender umbrella, its name and general branding are up in the air.

Arguments for:

- "Masonry" is a catchy name, easy to pronounce in most european languages.
- The name already exists and the Rust community is vaguely aware of it, to the point is comes up (rarely) in discussions.
- The name evokes brickwork and building stuff out of sturdy rectangle things. It can evoke having a solid foundation that things are built on top of.
- I have [the coolest logo idea ever](https://github.com/PoignardAzur/masonry-rs/issues/7).

Arguments against:

- "A masonry layout" is a thing, especially in web development.
  - People unfamiliar with Rust might come upon this crate and misunderstand what it's for.
  - When talking about how Masonry does layout, discussion could get confusing.


## Future possibilities

### Performance features

Xilem has added features over Druid that Masonry doesn't support, notably virtual lists and async support.

I think they're currently under-used in Xilem, so Masonry doesn't need to implement them right away for parity.

In the long term, these features will be very important for eventual performance work.

### Directory structure

Xilem's current directory structure is a bit awkward. It looks like:

- `crates/`
  - `xilem_core/`
  - `xilem_web/`
  - (soon) `masonry/`
- `src/`

In other words, all crates are in a `crates/` directory, except for `xilem/` which is in the root. This makes it seem like eg `xilem_web` is a dependency of `xilem`, when they're disjoint libraries.

An earlier draft of this RFC proposed renaming `xilem` to `xilem_native`, creating a placeholder `xilem` crate to inform about the multiple backends, and moving all of them into the `crates/` folder.

However, these changes were deemed out of scope for this RFC.

### Ownership story

We need to agree on a general procedure for publishing and updating crates, including Masonry, which is currently owned only by Olivier Faure.

At the very least, if Masonry is integrated, its ownership should probably be shared with Raph Levien as a stopgap.
