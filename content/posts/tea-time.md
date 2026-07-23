
---

title: "The Elm Architecture, without Elm"
date: 2026-07-23
tags: ["tea", "crave", "c", "raylib", "clay", "architecture", "open-source"]
draft: false
summary: "What The Elm Architecture is, why most modern UI frameworks quietly ended up shaped like it, and how a small C app uses it with no runtime enforcing any of it."
---

## The problem TEA solves

[Crave](https://github.com/bichanna/crave) is a recipe manager I wrote in C. There's no framework under the drawing library, no React, no GTK, no SwiftUI. It's just [Clay](https://github.com/nicbarker/clay) laying out rectangles and [raylib](https://www.raylib.com/) putting pixels on screen, once a frame forever until the window closes.

Once you're redrawing everything 60 times a second instead of mutating a DOM tree, bugs stemming from state changes from three places that don't know about each other: a click handler mutates a field; a keyboard shortcut mutates the same field a different way; a database callback mutates it a third way, on a frame where the first two also fired. Nothing crashes. The screen just shows something nobody asked for, and you spend an evening adding print statements to figure out which of the three writers won that frame.

The Elm Architecture, TEA for short, structures an app so that question doesn't come up, because there's only ever one writer.

## The four pieces

TEA splits an app into four things and data only flows one direction through them.

```
        ┌──────────────┐
        │    Model     │   all your app state, one value
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │    view      │   Model -> what's on screen
        └──────┬───────┘
               │  user clicks / types
               ▼
        ┌──────────────┐
        │     Msg      │   a description of what happened
        └──────┬───────┘
               │
               ▼
        ┌──────────────┐
        │   update     │   (Model, Msg) -> new Model
        └──────┬───────┘
               │
               └──────────────┐
                              ▼
                        back to Model
```

**Model** is one value that holds everything: which screen you're on, what's typed into every field, what's loading. Not scattered across components, one value.

**view** is a pure function from Model to what the screen shows. It doesn't mutate anything. It just describes.

**Msg** is data, not a function call. A button doesn't say "delete this recipe right now." It says "here's a `Msg` describing a delete click, do with it what you will."

**update** is the only place a new Model gets built. It pattern matches on the incoming `Msg` and returns the next Model. That's its whole job.

The loop closes on its own. `view` renders the Model and wires up which `Msg` each interaction produces. The user does something. Then `update` folds that `Msg` into a new Model. `view` runs again on the result. No `setState` calls sprinkled through click handlers, because click handlers never get to touch state directly. They only get to emit a `Msg` and hand it off.

This is [Elm](https://elm-lang.org/)'s actual architecture, hence the name, and Evan Czaplicki wrote up the canonical version, commands and subscriptions included, [here](https://guide.elm-lang.org/architecture/). What I'm describing below is the stripped-down version I actually use.

## It's the modern default now, just under other names

TEA sounds a little academic, a functional-programming thing, and a decade ago when Elm was making the case for it, it kind of was. What's happened since is most of the UI world landed on the same shape from a completely different direction.

SwiftUI's `View` is a pure function of `@State`, and `@State` only changes through explicit bindings, not mutation from wherever you feel like. Jetpack Compose is the same idea with different keywords: composables are pure functions of state, state hoisting pushes the source of truth up to one place, and a composable re-runs instead of being told to mutate itself. React, once you're on hooks, and not reaching for a ref to sneak around the model, is doing the same thing, too. A component is a function of state, and `setState` schedules a new rendr rather than mutating in place.

None of these call themselves TEA and they all disagree on small details. Elm's `Msg` and `update` are explicit and easy to inspect in a way React's `setState` calls usually aren't, and Elm gives you no mutation escape hatch at all, but Swift and Kotlin very much do. But once your rendering is declarative, once you're describing what the screen should look like instead of issuing draw calls by hand, you need state changes to be legible and centralized, or the whole "describe, don't mutate" contract falls apart the first time two things want to touch the same value on the same frame. TEA stopped being a framework you opt into a while ago. It's become closer to just the shape UI state ends up in once rendering stops being imperative.

Crave's rendering never stopped being imperative. Clay and raylib redraw the whole tree every frame no matter what changed, no diffing, no reconciler. So, I didn't inherit TEA from a framework enforcing it. I picked it because the alternative was a raylib app full of unguarded mutation, and I'd already been burned by that shape once on an earlier version of Crave, the kind of codebase where I was scared to open `ui.c` (I don't have the history on git, though).

## Crave has no framework, so I built the loop myself

There's no Elm runtime here, nothing tying `Msg` dispatch to `update` for me automatically. `Model`, `Msg`, `update`, and `view` are just four things I chose to write as four separate things, and the loop connecting them is about thirty lines in `main.c`:

```c
while (!WindowShouldClose()) {
  float dt = GetFrameTime();

  app.pending = (Msg){.tag = MSG_NONE};

  // ...mouse & scroll state fed to Clay...

  ui_handle_mouse(&app);
  app_handle_input(&app);

  // TEA loop
  update(&app, app.pending);
  Clay_BeginLayout();
  view(&app);
  Clay_RenderCommandArray cmds = Clay_EndLayout(dt);

  BeginDrawing();
  Clay_Raylib_Render(cmds, app.fonts);
  ui_overlay(&app);
  EndDrawing();
}
```

`App` is the Model in this, one struct holding which screen is active, the recipe currently being edited, which text field has focus, whether a confirmation modal is open. `Msg` is a tagged union:

```c
typedef enum {
  MSG_NONE = 0,
  MSG_SHOW,        // grid: open read-only detail for .id
  MSG_EDIT,        // detail: switch to edit mode
  MSG_NEW,         // grid: edit a blank recipe
  MSG_SAVE,        // edit: commit, back to detail
  MSG_CANCEL,      // edit: discard, back to detail/grid
  MSG_BACK,        // detail: back to grid
  MSG_FOCUS,       // edit: focus field index .i
  MSG_LIST_ADD,    // edit: append a row to list .list
  MSG_LIST_DEL,    // edit: remove item .i from list .list
  MSG_PICK_IMAGE,  // edit: choose + copy an image
  MSG_DELETE,      // detail: open the delete confirmation
  MSG_DELETE_YES,  // modal: confirm delete
  MSG_DELETE_NO,   // modal: dismiss
  MSG_DISCARD_YES, // modal: confirm discarding unsaved edits
  MSG_DISCARD_NO,  // modal: keep editing
} MsgTag;

typedef struct {
  MsgTag tag;
  long id;     // MSG_SHOW
  int i;       // MSG_FOCUS field index and MSG_LIST_DEL item index
  ListId list; // MSG_LIST_ADD and MSG_LIST_DEL
} Msg;
```

`update` is just one big switch. Each arm rebuilds a piece of `App`, and nothing outside `update` touches `App`:

```c
void update(App *app, Msg msg) {
  switch (msg.tag) {
  case MSG_SAVE: {
    editor_commit(&app->editor);
    long id = db_save_recipe(app->db, &app->editor.recipe);
    if (id > 0) {
      app->dirty = false;
      app_reload(app);
      open_recipe(app, id);
      app->screen = SCREEN_DETAIL;
    }
    break;
  }

  case MSG_DELETE_YES:
    if (app->editor.recipe.id != 0)
      db_delete_recipe(app->db, app->editor.recipe.id);
    app->confirm_delete = false;
    app_reload(app);
    app->screen = SCREEN_GRID;
    app->focus = -1;
    break;

  // ... one arm per MsgTag ...
  }
}
```

Every "delete this recipe" or "switch to edit mode" in the whole app funnels through one of these arms. If a recipe gets deleted, I can grep for `MSG_DELETE_YES`, and there's exactly one place it happens. Not three, not "somewhere in `ui.c` depending on which button you clicked."

## Two ways in, one way through

Elm's `update` only ever gets called by its own runtime, on its own schedule. I don't have a runtime, so `update` gets called from two different places, and sorting that out was most of the actual design work.

The mouse path goes through Clay's hover callbacks. `view` builds the frame, and for anything clickable it registers a callback that, if this is the frame the click landed on, writes a `Msg` into `app->pending` instead of calling `update` directly:

```
view() runs, builds this frame's UI
   |
   +-- for each clickable element:
   |     register: "if clicked this frame, app->pending = <this element's Msg>"
   |
mouse click happens during Clay_OnHover's own hit-test
   |
   +-- app->pending now holds one Msg, queued, not yet applied
```

`app->pending` resets to `MSG_NONE` at the top of the frame and gets read exactly once, right before `view` runs. So at most one click-sourced `Msg` reaches `update` per frame, which happens to be exactly how many clicks a mouse can register in one frame anyway.

The keyboard path skips the queue entirely and calls `update` directly, more than once a frame if it needs to. Escape while a delete confirmation modal is open calls `update(app, {.tag = MSG_DELETE_NO})` on the spot. Cmd or Ctrl+S while editing calls `update(app, {.tag = MSG_SAVE})` on the spot. Both come out of a chain of early returns in `app_handle_input`. There's no real batching benefit to queuing a keypress, so I didn't bother pretending there was one.

The one-way-door rule from TEA still holds either way. Whether a `Msg` got queued from a click or fired straight from a keypress, `App` only ever gets rebuilt inside `update`. Two doors into the same room, not two rooms.

## What I gave up

None of this is what Elm actually gives you. There's no `Cmd`, no `Sub`, no compiler statically proving `update` handles every `MsgTag` (I get a `-Wswitch` warning if I forget one, which is a much weaker guarantee, and `MSG_NONE`'s empty arm exists specifically so an unhandled `Msg` doesn't silently do nothing without me noticing). Side effects like `db_save_recipe` happen inline inside `update` arms instead of getting handed off as data the way Elm's commands would, so `update` isn't actually pure here. It just has one caller and one job per arm, which turned out to be most of what I wanted out of this.

What I got for four structs and a switch statement is a debugging property that's already paid for itself a few times over. When Crave does something I didn't expect. I didn't go digging through `ui.c` for whoever mutated `App` this time. I break on `update`, and whatever `Msg` came in tells me exactly what the rest of the app thinks happened.

Anyway, Crave source code is [here](https://github.com/bichanna/crave) if you want to see the rest of the `update` switch.
