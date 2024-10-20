# Feature Name: Pass specification

## Summary

This proposal formally defines the semantics of the passes that Masonry runs in its event loop.

It includes these major changes:

- Container widgets no longer need to recurse pass methods on their children.
- Widgets can no longer add or remove child widgets in most passes.
- New passes: "mutate", "update" and "compose".

## Motivation

Masonry de-facto already has a pass system, where `on_pointer/text/access_event`, `lifecycle`, `layout`, `paint`, and `accessibility` passes are run roughly in that order whenever an interaction happens.

These passes are only loosely documented, and their interactions aren't formally specified.
For instance, what happens when a layout change triggers a lifecycle event which triggers another layout change isn't specified.

Furthermore, each pass is associated with a context type (`EventCtx`, `LifecycleCtx`, `LayoutCtx`, etc) which provides methods to access the environment.
Some of these methods are shared between all passes, some by all but one pass, and some are specific to a single pass.
Why a given method is available in one pass and not another is currently undocumented; it's obvious in some cases (e.g. it makes sense that EventCtx would have a `set_handled` method and PaintCtx wouldn't), but even then we should have a general model what capabilities are available in which passes.


## User-facing explanation

Masonry has a set of **passes**, which are computations run over a subset of the widget tree during a frame.


### Event passes

When a user interacts with the application in some way, like a mouse click, Masonry runs an **event pass** over the tree.
There are three types of event passes:

- **on_pointer_event:** covers positional events from the mouse and other pointing devices (pen, stylus, touchpad, etc).
- **on_text_event:** text input events like keyboard presses, IME, clipboard paste, etc.
- **on_access_event:** events from the OS's accessibility API.

When an event occurs, the application selects the widget targeted by the event.
For pointer events, this is either the widget under the pointer or the widget with pointer capture.
For text and accessibility events, this is the widget with focus.

The widget's event handling method (`on_pointer_event`, `on_text_event`, or `on_access_event`) is called.
Then, the same method is called for each of the widget's parents, up to the root.
This behavior is known in browsers as event bubbling.


### Animation pass

The **update_anim** pass runs an animation frame, which occurs at set intervals if the widget tree includes animated widgets.

It runs in depth-first preorder on all animated widgets in the tree.


### Rewrite passes

After an event pass, some flags may have been changed, and some values may have been invalidated and need to be recomputed.
To address these invalidations, Masonry runs a set of **rewrite passes** over the tree:

- **mutate** pass.
- **update_widget_tree** pass.
- **update_disabled** pass.
- **update_stashed** pass.
- **update_focus_chain** pass.
- **update_focus** pass.
- **layout** pass.
- **update_scrolls** pass.
- **compose** pass.
- **update_pointer** pass.

The layout and compose passes have methods with matching names in the Widget trait.
The update_xxx passes call the widgets' update method.

By default, each of these passes completes immediately, unless pass-dependent invalidation flags are set or work is requested.
Each pass can generally request work for later passes; for instance, the mutate pass can invalidate the layout of a widget, in which case the layout pass will run on that widget and its children and parents.

Passes may also request work for *previous* passes, in which case all rewrite passes are run again in sequence.
For instance, the update_pointer pass may change a widget's size, requiring another layout pass.

To avoid infinite loops in those cases, the number of reruns has a static limit.
If passes are still requested past that limit, they're delayed to a later frame.

#### The mutate pass

The **mutate** pass runs a list of callbacks with mutable access to the widget tree.
These callbacks can be queued with the `mutate_later()` method of various context types.

"Mutable access" means that those callbacks are given a `WidgetMut` to the widget that requested them, something that is otherwise only accessible from the owner of the global `RenderRoot` object (see "External Mutation" section).

If a callback is scheduled to run on a widget which is deleted before the callback is run, that callback is silently dropped.

*Note:* The mutate pass is meant to be *an escape hatch*.
It covers widgets which don't quite fit into the pass system and future use-cases that we didn't foresee while developing Masonry.
It's more powerful and gives complete access to the tree, but is also slightly more expensive and less idiomatic than doing things in other passes.

Widgets should try to fit their logic into the other passes, and use `mutate_later()` sparsely.

#### Update passes

Update passes mostly run internal calculations.
They compute if some widget's property has changed, and send it a matching `update` event (see "Status" section below).

For instance, if a user presses `Tab` and the event isn't handled in a widget, Masonry will run the `update_focus` pass, which will automatically switch keyboard focus to the next focus-accepting widget.
Both the previously-focused widget and the newly-focused widget will get an `update` call with relevant values.

#### Update tree pass

The `update_widget_tree` pass is a special case.
It is called when new widgets are added to the tree, or existing widgets are removed.

It will call `register_children` widget method on container widgets whose children changed, then the `update` method with the `WidgetAdded` event on new widgets.

#### Layout pass

The layout pass runs bidirectionally, passing constraints from the top down and getting back sizes and other layout info from the bottom up.

It is subject to be reworked in the future to be closer to the semantics of web layout engines and the Taffy crate.

Unlike with other passes, container widgets' `Widget::layout()` method must call `WidgetPod::layout()` on all of their children.

Not doing so is a logical bug.
When debug assertions are on, Masonry will actively try to detect cases where you forget to compute a child's layout and panic if it finds such a case.

#### Compose pass

The **compose** pass runs top-down and assigns transforms to children.
Transform-only layout changes (e.g. scrolling) should request compose instead of requesting layout.

Compose is meant to be a cheaper way to position widgets than layout.
Because the compose pass is more limited than layout, it's easier to recompute in many situations.

For instance, if a widget in a list changes size, its siblings and parents must be re-laid out to account for the change; whereas changing a given widget's transform only affects its children.

Masonry automatically calls the `compose` methods of all widgets in the tree, in depth-first preorder, where child order is determined by their position in the `children_ids()` array.


### Render passes

Event and rewrite passes can invalidate how the widget tree is presented to the user.

If that happens, a redraw frame will be requested from the environment (e.g. the Winit event loop).
When the environment applies the redraw, it will run the **render passes** as needed:

- **paint:** The paint pass gets a Vello Scene description from each widget.
These scenes are then stitched together in pre-order: first the parent, then its first child, then *its* first child, etc.
- **accessibility:** The accessibility pass gets an AccessKit node description from each widget.
These nodes together form the accessibility tree.

Methods for these passes should be written under the assumption that they can be skipped or called multiple times for arbitrary reasons.
Therefore, their ability to affect the widget tree is limited.

Masonry automatically calls these methods for all widgets in the tree in depth-first preorder.


### External mutation

Code with mutable access to the `RenderRoot`, like the Xilem app runner, can get mutable access to the root widget and all its children through the `edit_root_widget()` method, which takes a callback and passes it a `WidgetMut` to the root widget.

This is in effect a MUTATE pass which only processes one callback.

External mutation is how Xilem applies any changes to the widget tree produced by its reactive step.

Calling the `edit_root_widget()` method, or any similar direct-mutation method, triggers the entire set of rewrite passes.


### Editing the widget tree

Widgets can be added and removed during event and rewrite passes *except* inside layout and register_children methods.

Not doing so is a logic error and may trigger debug assertions.

If you do want to add or remove a child during layout, you can always defer it with the `mutate_later` context method.


### Widget methods and context types

Widgets are types which implement the `masonry::Widget` trait.

This trait includes a set of methods that must be implemented to hook into the different passes listed above:

```rust
// Exact signatures may differ
trait Widget {
    fn on_pointer_event(&mut self, ctx: &mut EventCtx, event: &PointerEvent);
    fn on_text_event(&mut self, ctx: &mut EventCtx, event: &TextEvent);
    fn on_access_event(&mut self, ctx: &mut EventCtx, event: &AccessEvent);

    fn register_children(&mut self, ctx: &mut RegisterCtx);
    fn update(&mut self, ctx: &mut UpdateCtx, event: &UpdateEvent);
    fn layout(&mut self, ctx: &mut LayoutCtx) -> Size;
    fn compose(&mut self, ctx: &mut ComposeCtx);

    fn paint(&mut self, ctx: &mut PaintCtx, scene: &mut Scene);
    fn accessibility(&mut self, ctx: &mut AccessCtx);

    // ...
}
```

These methods all take a given context type as a parameter.
Methods aside, `WidgetMut` references can provide a `MutateCtx` context.
`WidgetRef` references can provide a `QueryCtx` context, which is used in some read-only methods.

Those context types have many methods, some shared, some unique to a given pass.
There are too many to document here, but we can lay out some general principles:

- Render passes should be pure and can be skipped occasionally, therefore their context types (`PaintCtx` and `AccessCtx`) can't set invalidation flags or send signals.
- The `layout` and `compose` passes lay out all widgets, which are transiently invalid during the passes, therefore `LayoutCtx`and `ComposeCtx` cannot access the size and position of the `self` widget.
They can access the layout of children if they have already been laid out.
- For the same reason, `LayoutCtx`and `ComposeCtx` cannot create a `WidgetRef` reference to a child.
- `MutateCtx`, `EventCtx` and `UpdateCtx` can let you add and remove children.
- `RegisterCtx` can't do anything except register children.
- `QueryCtx` provides read-only information about the widget.

### Other concepts

This section describes concepts mentioned by name elsewhere in the RFCs and gives them a semi-formal definition for future reference.

#### Widget status

The notion of widget status is somewhat vague, but you can think of it as similar to [CSS pseudo-classes](https://developer.mozilla.org/en-US/docs/Web/CSS/Pseudo-classes).

Widget statuses are "things" managed by Masonry that affect how widgets are presented.
Statuses include:

- Being hovered.
- Having pointer capture.
- Having local focus.
- Having active focus.
- Being disabled.
- Being stashed.

When one of these statuses changes, the `update` method is called on the widget.
However, `update` can be called for reasons other than status changes.

#### Pointer capture

When a user starts a pointer click on a widget, the widget can "capture" the pointer.

Pointer capture has a few implications:

- When a widget has captured a pointer, all events from that pointer will be sent to the widget, even if the pointer isn't in the widget's hitbox.
Conversely, no other widget can get events from the pointer (outside of bubbling).
- The "hovered" status of other widgets won't be updated even if the pointer is over them.
The hovered status of the capturing widget will be updated, meaning a widget that captured a pointer can still lose the "hovered" status.
- The pointer's cursor icon will be updated as if the pointer stayed over the capturing widget.
- If the widget loses pointer capture for some reason (e.g. the pointer is disconnected), the Widget will get a `PointerLeave` event.

Masonry should guarantee that pointers can only be captured by one widget at a time.
Masonry should force the widget to lose pointer capture when some events occur; not just MouseLeave, but also `Tab` being pressed, the window losing focus, the widget being disabled, etc.

Examples of use cases for pointer capture include selecting text, dragging a slider, or long-pressing a button.

#### Focus

Focus marks whether a widget receives text events.

To give a simple example, when you click a textbox, the textbox gets focus: anything you type on your keyboard will be sent to that textbox.

Focus can be changed with the tab key, or by clicking on a widget, both which Masonry automatically handles.
Widgets can also set custom focus behavior.

Note that widgets without text-edition capabilities such as buttons and checkboxes can also get focus.
For instance, pressing space when a button is focused will trigger that button.

There are two types of focus: active and inactive focus.
Active focus is the default one; inactive focus is when the window your app runs in has lost focus itself.

In that case, we still mark the widget as focused, but with a different color to signal that e.g. typing on the keyboard won't actually affect it.

#### Disabled

A disabled widget is one which is visibly marked as non-interactive.

It is usually grayed out, and can't receive pointer or text events.

#### Stashed

A stashed widget is one which is no longer "part of the logical tree", so to speak.

Stashed widgets can't receive keyboard or pointer events, don't get painted, aren't part of the accessibility tree, but should still keep some state.

The stereotypical stashed widget would be one inside a hidden tab in a "tab group" widget.

By contrast, widgets scrolled outside the viewport are **not** stashed: they can still get text events and are part of the accessibility tree.

#### Interactivity

A widget is considered "interactive" if it can still get text and/or pointer events.
Stashed and disabled widget are non-interactive.

#### Safety rails

When debug assertions are on, Masonry runs a bunch of checks every frame to make sure widget code doesn't have logical errors.

These checks are sometimes referred to as "safety rails".


## Implementation strategy

### More explicitly document passes in the code

Comments and method names in `RenderRoot` and `WidgetPod` should explicitly call out passes where they happen.

The "User-facing explanation" section should be added to the crate's documentation, and in-code documentation should refer to it.

### Rename WidgetCtx to MutateCtx

Since the mutate pass is becoming a documented part of the code, MutateCtx would be a clearer name for that context type.

### Create a register_children method

The RouteWidgetAdded event should be removed from `Lifecycle`, and replaced with a dedicated `update_widget_tree` calling `Widget::register_children`.

Widgets should be removed exclusively through the `remove_child` method in `MutateCtx`, `EventCtx` and `UpdateCtx`.

Widgets should not be added during `layout` and `register_children`.
Safety rails should be added to make sure that these two methods iterate on all their children.

### Change how methods are recursed

For most passes, we should switch from requiring that widgets recurse the same methods to all their children, to having pass code directly call those methods on the children.

This will mostly involve changes to `WidgetPod`'s internals.

For targeted passes, we should add a private method to RenderRoot which directly sends an arbitrary event to a specific widget, then propagates changes upwards.

### Add mutate pass

We should add a queue of callbacks with their target widget.
Calling `FoobarCtx::mutate_later()` would add a callback to that queue.

The MUTATE pass would then go through that queue.

### Add compose pass

We should add a `compose()` method to the WidgetTrait, a `WidgetPod::compose` method, a `ComposeCtx` type, a `RenderRoot::root_compose()` method, etc.

The `ParentWindowOrigin` lifecycle event should be replaced by this compose pass.

Context types should get a `request_compose()` method.

### Remove LifecycleCtx methods

The following methods specific to LifecycleCtx should be removed, and replaced with getter methods on the Widget trait:

- `register_for_focus`
- `register_as_text_input`
- `register_as_portal`

For instance, when a widget is created, `WidgetPod` should call a method called `Widget::accepts_focus` on the widget.
If so, it should add it to the set of widgets accepting focus.

Then when the widget is deleted (or disabled), it should be removed from that set.

How Masonry keeps track of which widgets accept focus is outside the scope of this RFC.

### Send lifecycle events directly from RenderRoot

The following passes should be created in RenderRoot, which will send LifecycleEvents directly to the concerned widgets:

- `update_widget_tree`: Sends WidgetAdded event and registers new children.
- `update_disabled`: Sends DisabledChanged event.
- `update_stashed`: Sends StashedChanged event.
- `update_focus_chain`: Sends BuildFocusChain event.
- `update_focus`: Sends FocusChanged event.
- `update_scrolls`: Sends RequestPanToChild.
- `update_pointer`: Sends HotChanged and updates the cursor icon.

These features should progressively be moved out of the current `WidgetPod::lifecycle` method.

### Replace Lifecycle and StatusChange with UpdateEvent

The previous sections are about progressive changes that should be implemented to take things off the lifecycle pass.

The lifecycle pass and the Lifecycle event should eventually be replaced with the UPDATE passes and StatusUpdate event altogether.

### Rework paint pass

The paint pass should now build the root scene in a single pass.

`WidgetPod::paint()` will do the following:

- Call the inner paint().
- That paint "returns" both a scene fragment and an optional clip layer.
- Append the scene fragment to the global scene.
- If the clip layer is Some, push it.
- For each child, call `WidgetPod::paint()` with the transform that was set in `compose()`.
- If the clip layer is Some, pop it.

Unlike the current algorithm where fragments are added to parent fragments, here they are added directly to the root.
This is `O(N)` [instead of `O(N * Depth)`](https://xi.zulipchat.com/#narrow/stream/354396-xilem/topic/The.20mystery.20of.20the.20Xilem.20bomb).

We should add `PaintCtx::set_clip_box()` and `PaintCtx::remove_clip_box()` methods so widgets can clip their children.
In most cases this will be a rectangle, but it can be an arbitrary path, like a rectangle with rounded corners.

### Get full test coverage of Context methods

These behaviors and the invariants that come with them should be part of a spec.
This spec should take the form of both doc comments, code comments, and a full testing suite.

This suite should have full coverage of all Context methods and WidgetPod methods.
Because some context methods are macro-duplicated between context types, some tests might need to be duplicated as well.


## Drawbacks

### Requiring complex architecture for handling passes

This RFC suggests adding one more layer of complexity in WidgetPod for recursing events directly to children, instead of letting container widgets do so in their implementations of trait methods.

That added complexity should be managed and documented.

### Being too structured

While this RFC mentions some escape hatches, overall it adds a certain rigidity to how widget behaviors are implemented.
In particular, the restrictions about adding new widgets may be too constraining.

However, I'd argue this is a rigidity that already existed, but simply was undocumented:

- Druid and Masonry have always worked on a pass system, with the passes being based on recursing the Widget methods.
- Druid's passes were already constraining, with the Notification and Command events being included as escape hatches to escape some of these constraints.
- Adding a new widget in the middle of a layout pass and trying to lay out that widget could already crash the application or trigger a warning.

With this RFC's design the restrictions are documented and, wherever possible, enforced by the type system.


## Rationale and alternatives

The main benefit of this RFC is to untangle code.

By specifying what passes do and how they interact with each other, we can make pass-related code more structured and modular.

We could make a less ambitious RFC where we document the widget behavior more formally, but avoid some major changes:

- Keep the current "`MyParentWidget::on_text_event` must call `WidgetPod<MyChildWidget>::on_text_event` for every child" approch.
- Cut the part where mutating the widget tree is only allowed in `MutateCtx`.

However, these major changes both help us massively simplify Masonry's internal code:

- Having `RenderRoot` be in charge of event targeting means we can get rid of the WidgetPod's "should this event be recursed to the inner widget" logic.
- Having direct event targeting means we can remove a bunch of `RouteFoobar` events that do nothing but carry a `Foobar` event through the widget tree.
- Limiting mutations to the widget tree to a specific pass lets us remove the `WidgetAdded` event and some very thorny logic.

Overall, these changes together will make the new pass system more cohesive and predictable.


## Prior art

The pass system is inspired from the web platform.

As far as I'm aware, no other Rust framework has an explicitly documented pass system.


## Unresolved questions

The RFC is vague on the following implementation points:

- Exact pointer capture behavior.
- Exact focus behavior.
- Detailed description of the UPDATE passes.
- Static limit on the number of reruns.
- The total list of context methods.

We will likely figure them out during implementation and document them in real-time.


## Future possibilities

### Restricting children creation/removal

A previous version of this RFC had rules under which widgets could only create new widgets inside the mutate pass.
This has been removed, because it proved too complicated to implement, especially on the Xilem side.

A future RFC could add that restriction back.

### Reparenting children

Right now children can only be added and removed.

In the future, we could add the ability to splice a widget subtree from one parent widget to another.

### Focus

Focus needs to be better defined.

The concept of "local focus" vs "active focus" might need a more formal definition.

A related question is how we preserve local focus for situations that "borrow" it. For instance, if you press `Alt` to open a menu, the menu should have focus.
When the menu is closed, focus should go back to the previous focused element.

### Layout

Right now the layout algorithm is a single-pass traversal of the widget tree which passes down constraints and returns sizes.

In the future, we're likely to shift to a multi-pass algorithm closer to how the web platform does layout, probably integrating with Taffy in the process.

### Skipping layout

Right now layout is always run over the entire tree whenever any widget needs its layout change.

We could start caching box constraints, and only running layout if either the constraints have changed or a layout change has been explicitly requested.

### Reserving scenes in paint

Since each widget's scene fragment is cached, and fragments are added to the root scene in linear order, the paint pass might be a good place for "reserve"-type APIs.

We could run a prepass that goes over the entire widget tree and sums the sizes of every scene fragment, then creates a root scene with the total size reserved.
