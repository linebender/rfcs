# Feature Name: `widgets-in-arenas`

## Summary

Change Masonry's architecture so that all widgets are stored in a per-application arena.
Container widgets won't store child widgets anymore, and will instead store keys/indices into that arena.


## Motivation

In Masonry's architecture, containers widgets store their children inside `WidgetPod`s.
For instance, the `SizedBox` widget is declared roughly like this:

```rust
pub struct SizedBox {
    child: WidgetPod<Box<dyn Widget>>,
    width: Option<f64>,
    height: Option<f64>,
    background: Option<, egBackgroundBrush>,
}
```

The WidgetPod struct stores a widget with some associated data like its position, size, invalidation flags, etc.

This architecture, where each widget directly owns its children, means that any event or feature that targets a specific widget must nevertheless be passed through the widget hierarchy.
Leaving aside performance, this requires a lot of boilerplate code.

For instance, Masonry has a `RouteFocusChanged` event that widgets aren't supposed to react to, but are supposed to pass down to all their children.
Accessibility events include a target widget id and are similarly passed through.

To avoid sending events to the entire tree needlessly, widgets use bloom filters as a heuristic to estimate whether a given widget id is within their subtree.
This is an imperfect heuristic: bloom filters have more false positives as their number of entries grow, and even without false positives this algorithm can still end up visiting a lot of widgets just to rule them out.

Performance aside, having widgets own their children means that any new functionality in the widget tree requires adding new event variants and special cases in WidgetPod methods to handle them.
You can't add a feature that directly mutates a widget's sub-sub-sub-child, you have to route an event meant for the sub-sub-sub-child to be mutated through the tree.

Storing widgets in an arena accessible from the root both improves performance and simplifies access patterns.


## User-facing explanation

Containers widgets store their children through `WidgetPod`s.
For all intents and purposes, a `WidgetPod<MyWidget>` stores an both an instance of `MyWidget` and some metadata common to all widgets, like its position, size, and various update flags.

The root WidgetPod is owned by the `RenderRoot` type.

WidgetPods will gate access to the widget they hold.
A `WidgetPod<MyWidget>` doesn't not unconditionally give you a `&mut MyWidget` reference; this is to enforce that any code that mutates child widgets will also update invalidation flags and uphold invariants.

There are a few different ways to access child widgets, which we'll cover in turn.

### Pass methods

The Masonry architecture revolves around update passes: when the end user does something (type text, move the mouse, etc), a succession of passes is run on the widget tree:

- Event
- Lifecycle
- Layout
- Paint
- Accessibility

In some cases, the widget tree's state can be considered "partially valid" in-between those passes.
Some passes can run multiple times per user event or be skipped entirely.

Widgets are type which implement the Widget trait; this trait has methods linked to passes:

- `on_text_event`,
- `on_pointer_event`,
- `on_access_event`,
- `lifecycle`,
- `layout`,
- `paint`,
- `accessibility`,

WidgetPod has matching methods, which must be called by container widgets.

In other words, if you implement a custom container widget with a single child, your implementation of the `on_pointer_event` method must call `my_child.on_pointer_event(...)`.
This will indirectly call the child's own implementation of `Widget::on_pointer_event`.

#### Context types

Each pass method of the Widget trait takes a context type as parameter:

- `on_text_event` has `EventCtx<'_>`,
- `on_pointer_event` has `EventCtx<'_>`,
- `on_access_event` has `EventCtx<'_>`,
- `lifecycle` has `LifeCycleCtx<'_>`,
- `layout` has `PaintCtx<'_>`,
- `paint` has `LayoutCtx<'_>`,
- `accessibility` has `AccessCtx<'_>`,

These context types have a lot of overlapping methods.
Broadly speaking, they provide information about the current widget accessible during the matching pass.

Some context types also have methods that access a child WidgetPod's metadata, like `LayoutCtx::child_layout_rect(...)`.

### WidgetRef

WidgetRef is a rich reference type that carries both a shared reference to the widget and to its metadata.

You can only get a WidgetRef to a fully valid widget; you can't get a WidgetRef in the middle of a set of passes.

Right now WidgetRef is mostly be created during unit tests: RenderRoot can give you a WidgetRef to either the root or an arbitrary widget whose id you know, which you can use to get information on the state of the widget tree.

A WidgetRef can give you a Widget's layout, whether it's hovered/focused/disabled, etc, and can create WidgetRefs of the widget's children.

### WidgetMut

WidgetMut is another rich reference type, with a mutable reference to the widget and to its metadata.

Again, you can only get a WidgetMut to a fully valid widget; you can't get a WidgetMut in the middle of a set of passes.

Developers implementing custom widgets won't *use* WidgetMut a lot, but they must implement methods that lets a consumer with a `WidgetMut<MyWidget>` mutate the MyWidget instance.
These methods must both change the fields of MyWidget and use the bundled context object to uphold invariants.

For example:

```rust
impl WidgetMut<'_, MyLabel> {
    pub fn set_text_size(&mut self, new_size: u32) {
        self.widget.text_size = new_size;
            self.ctx.request_layout();
    }
}
```

(``self.ctx` is of type WidgetCtx, a special context type available to WidgetMut.)

This constraint ensures that user code can never forget to uphold invariants (set invalidation flags, send signals, etc) when mutating a widget.

WidgetMut is used in-between user events: RenderRoot can provide a WidgetMut to the root widget on demand.

Container widgets must implement methods to let users map WidgetMut of parents to WidgetMut of children.

### RawWrapper and RawWrapperMut

WidgetRef and WidgetMut can't be accessed in the middle of pass methods.

If developers implementing these pass methods for custom widgets really need a reference to a child widget, context types have an escape hatch in the form of two  methods `get_raw_ref` and `get_raw_mut`.

These methods return a RawWrapper/RawWrapperMut which holds both a reference to the child and a context value "scoped" to it.

For example:

```rust
let mut my_child = lifecycle_ctx.get_raw_mut(&mut self.my_child);
// Do stuff with the child widget
my_child.widget().foo = foo(stuff);
my_child.widget().bar = bar();
// This is equivalent to calling ctx.request_paint()
// inside the child's lifecycle() method.
my_child.ctx().request_paint();
```

Note that not only the parent widget *can* call these methods, it is *responsible* for doing so to maintain any invariants related to how it mutates the child widget.
For instance, if a container changes the `my_padding` attribute of a child widget, it should likely call `request_layout()` for it.

In general, while RawWrapper can be used liberally, RawWrapperMut is only expected to be used for private widgets used as an implementation detail of public widgets.
These private widgets must be tagged with the `AllowRawMut` marker trait.

### `Widget::children_ids()`

Each widget must return a list of its children:

```rust
    // Rough prototype of the method
    fn children_ids(&self) -> [WidgetId]
```

WidgetRef relies on the ids this returns to browse the widget tree.
widgetPod's pass methods rely on these ids to perform sanity checks (for instance, to check that you didn't forget to paint a container widget's children while painting the container).

Therefore, that list of children should be precisely accurate.

#### Fine-grained tree-structure tracking

For efficiency and correctness reasons, we need to keep track of every widget's parent and children.

This currently has a few implications:

- WidgetPods shouldn't be passed around after they've been inserted into the widget tree.
If X is the child of Y, it can't ever become the child of Z instead.
- To keep track of new widgets, parent widgets must *always* propagate the `WidgetAdded` lifecycle event.
- To keep track of widgets that are removed, widgets can only be removed with the `LifecycleCtx/EventCtx::child_removed` method.










## Implementation strategy

**The public API should maintain the *illusion* that `WidgetPod<W>` owns a value of type W.**

But widgets and WidgetStates should be stored in an arena owned by RenderRoot.

(That arena could be a hashmap, a slotmap, an ECS store, or something else entirely.)

This means parent widgets should still own `WidgetPod`, but `WidgetPod` should only store a key for the arena.

RenderRoot and TestHarness will both acquire a new method:

```rust
pub fn edit_widget<R>(&mut self, id: WidgetId, f: impl FnOnce(WidgetMut<'_, Box<dyn Widget>>) -> R) -> R;
```

(The exact signature may differ.)

### Storage

An arena storing all widgets of the application and their WidgetState will be added to the `RenderRootState` struct.

How the arena is implemented won't be part of the public API.
The previous section mentions a few possible candidates.

Because the internal code will need to repeatedly borrow from the same arena mutably, whatever storage method we use must allow multiple borrows from the same arenas: as far as the type system is concerned, if we're borrowing a node, we're borrowing the entire tree, because they're all in the same arena.
This is especially annoying because a lot of Masonry code needs to iterate on a widget's children while the parent is still borrowed.

We could write unsafe code to exploit extra information we have about the widget tree: while the Rust compiler only sees arena keys as integers, we as the designers know they have to obey ownership-like rules: widgets can only own keys to child widgets, every widget can only have one parent, children can't be shared, etc.

The bottom line is, there are three ways to implement our arena:

- (1) Not actually using an arena, storing items in an actual tree, with lots of allocations.
- (2) Storing items in cells or mutexes, and borrowing dynamically.
- (3) Storing items in UnsafeCells, and enforcing hierarchical borrow rules that guarantee that overlapping mutable borrows of these cells is impossible.

The initial implementation will use solution 1, with an API compatible with solution 3.

The goal is so have a %100 safe (if inefficient) implementation that we can build Masonry on top of, so that once we write the unsafe version we already have a battle-tested API and a test suite for e.g. MIRI.

### WidgetPod

Because WidgetPod no longer stores either a widget or a state, the following methods on WidgetPod can no longer exist:

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

- I plan to write a second RFC named "Update passes" which will expand on this current RFC, especially the notions of "partially valid" widgets, the role of context types, restrictions on when the widget tree can be mutated, etc.
- While we currently use the `WidgetAdded` lifecycle event to keep track of new widget, we might replace it with an entirely different pass, or limit how widgets are added to the arena to make WidgetAdded unnecessary.
