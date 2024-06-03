# Feature Name: `widgets-in-arenas`

## Summary

Change Masonry's architecture so that all widgets are stored in a per-application arena.
Container widgets won't store child widgets anymore, and will instead store keys/indices into that arena.


## Motivation

Currently, if you want to declare a container widget, that container must store its children inside `WidgetPod`s.
For instance, the `SizedBox` widget is declared roughly like this:

```rust
pub struct SizedBox {
    child: WidgetPod<Box<dyn Widget>>,
    width: Option<f64>,
    height: Option<f64>,
    background: Option<BackgroundBrush>,
}
```

The WidgetPod struct stores a widget with some associated data, notably for layout and pass invalidation.

This architecture, where each widget directly owns its children, means that any event or feature that targets a specific widget must nevertheless be passed through the widget hierarchy.
Leaving aside performance, this is requires a lot of boilerplate code.

For instance, Masonry has a `RouteFocusChanged` event that widgets aren't supposed to react to, but are supposed to pass down to all their children.
Accessibility events include a target widget id and are similarly passed through.

To avoid sending events to the entire tree needlessly, widgets use bloom filters as a heuristic to estimate whether a given widget id is within their subtree.
That heuristic is cumbersome.

Besides existing features, having widgets own their children mean that any new functionality in the widget tree requires adding new event variants and special cases in WidgetPod methods to handle them.
You can't add a feature that directly mutates a widget's sub-child, you have to route a request for the sub-child to be mutated through the entire hierarchy.


## User-facing explanation

Widgets should be stored in an arena owned by `RenderRoot`.

(That arena could be a hashmap, a slotmap, an ECS store, or something else entirely.)

**The implementations should take pains to makes this invisible to users.**

This means parent Widgets should still own `WidgetPod`, but `WidgetPod` should no longer store a widget.
Instead, it should only store a key for the arena.

That means the following methods on WidgetPod can no longer exist:

```rust
    pub fn widget(&self) -> &W;
    pub fn as_ref(&self) -> WidgetRef<'_, W>;
    pub fn as_dyn(&self) -> WidgetRef<'_, dyn Widget>;
    pub fn is_initialized(&self) -> bool;
    pub fn has_focus(&self) -> bool;
    pub fn is_active(&self) -> bool;
    pub fn has_active(&self) -> bool;
    pub fn is_hot(&self) -> bool;
    pub fn id(&self) -> WidgetId;
    pub fn layout_rect(&self) -> Rect;
    pub fn paint_rect(&self) -> Rect;
    pub fn paint_insets(&self) -> Insets;
    pub fn compute_parent_paint_insets(&self, parent_size: Size) -> Insets;
    pub fn baseline_offset(&self) -> f64;
    pub fn widget_mut(&mut self) -> &mut W;
```

In exchange, RenderRoot and TestHarness both acquire a new method:

```rust
pub fn edit_widget<R>(&mut self, id: WidgetId, f: impl FnOnce(WidgetMut<'_, Box<dyn Widget>>) -> R) -> R;
```

(The exact signature may differ.)

### Fine-grained tree-structure tracking

To be able to send events directly to a widget while propagating invalidations up the widget tree, we need to keep track of every widget's parent and children.

We need to keep track of new children that are added.
This is currently done with the `WidgetAdded` lifecycle event.
(Though that might change in the future.)

We also need to keep track of children that are removed.
This is done with `LifecycleCtx/EventCtx::child_removed`.


### Widget children method

Each widget should return a list of its children, that Masonry can use to browse the widget tree.
This is done through a `children()` method in the Widget trait:

```rust
    fn children(&self) -> SmallVec<[WidgetId; 16]>;
```

## Implementation strategy

### Storage

An arena storing all widgets of the application will be added to the `RenderRootState` struct.

As mentioned in the previous section, the exact storage used to store Widgets won't be part of the public API.

The initial implementation will store all widgets in a hashmap for simplicity.
Later implementations may switch to a different storage type (eg a slotmap) for efficiency.

### WidgetPod

A large number of methods in WidgetPod that access the underlying Widget or its WidgetState will have to be removed.
These methods are listed in the previous section.
Most of them are currently either unused or used in places where equivalent methods can be found on `SomethingCtx` arguments or on `WidgetRef`.
Removing and replacing these methods would be the first step of implementing this proposal.

The WidgetPod constructors would stay the same: they would only take a Widget (and possibly an Id) as a parameter.
Because these constructors don't include a reference to the arena, this means a newly-created WidgetPod will need to store a copy of the Widget it's created with.

Thus, the new WidgetPod type should look something like:

```rust
enum WidgetPod {
    Initial(Widget),
    Valid(WidgetId),
}
```

The implementation might evolve to make this unnecessary, for instance by making it impossible to create a WidgetPod without a reference to the arnea.
Doing so would have some knock-on effects that are out of scope for this proposal.


### Interior mutability

The basic trade-off of storing a tree in an arena is that, while we're decoupling ownership from memory allocation, we couple every node to the arena.

As far as the type system is concerned, if we're borrowing a node, we're borrowing the entire tree, because they're all in the same arena.
This is especially annoying because a lot of Masonry code needs to iterate on a widget's children while the parent is still borrowed.

This means we need escape hatches, taking the form of either unsafe code or interior mutability.
The latter has a performance cost, the former is a minefield.

Unsafe code could exploit extra information we have about the widget tree: while the Rust compiler only sees arena keys as integers, we as the designers know they have to obey ownership-like rules: widgets can only own keys to child widgets, every widget can only have one parent, children can't be shared, etc.
However, it's unclear to me whether that information could be surfaced to the compiler both safely and conveniently.

In the short term, any solution would probably be based on mutexes.
The runtime cost will probably be low in practice, because all mutexes will be un-contended by design (failing to lock a widget's mutex would be a symptom of a logic error in Masonry's code).

In the medium term, we may want to add tests and fuzzing, to check that we never get failed locks in practice.

In the longer term, we'll probably want to create our own arena crate for the specific purpose of efficiently accessing a tree store in an arena.
There are clever algorithms that can bypass atomic checks, though they all have trade-offs.


## Drawbacks

This is a significant change in the architecture.
It means that code browsing the widget tree must be a lot more aware of the arena.
It adds some internal complexity to WidgetPod so that external users aren't exposed to arena internals.

## Rationale and alternatives

Wanting widgets to be stored in an arena has been a constant throughout my work on Druid and Masonry.

Storing widgets in an arena would make it much faster to add new features to Masonry's code.
Something like "sending an accessibility event to the widget with this id" could become as simple as calling `root.get_widget_mut(id).send_event(accessibility_event)`, whereas it currently requires code to be added to multiple modules.

Handling text focus would become easier, handling hot state and pointer capture would become easier, etc.
I have a backlog of improvements blocked on transitionning to the arena architecture.


## Future possibilities

I plan to write a second RFC named "Update passes" which will expand on this current RFC, especially by restricting when the widget tree can be mutated.

That said, the current RFC should stand on its own.
