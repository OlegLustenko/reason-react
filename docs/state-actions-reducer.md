---
id: state-actions-reducer
title: State, Actions & Reducer
---

Finally, we're getting onto stateful components!

ReasonReact stateful components are like ReactJS stateful components, except with the concept of "reducer" (like [Redux](http://redux.js.org)) built in. If that word doesn't mean anything to you, just think of it as a state machine. If _that_ word doesn't mean anything to you, just think: "Woah this is great".

To declare a stateful ReasonReact component, instead of `ReasonReact.statelessComponent "MyComponentName"`, use `ReasonReact.reducerComponent "MyComponentName"`:

```reason
let component = ReasonReact.reducerComponent "Greeting";

let make ::name _children => {
  ...component,
  initialState: fun () => 0, /* here, state is an `int` */
  render: fun self => {
    let greeting = "Hello " ^ name ^ ". You've clicked the button " ^ (string_of_int self.state) ^ " time(s)!";
    <div> (ReasonReact.stringToElement greeting) </div>
  }
};
```

## `initialState`

ReactJS' `getInitialState` is called `initialState` in ReasonReact. It takes `unit` and returns the state type. The state type could be anything! An int, a string, a ref or the common record type, which you should declare **right** before the `reducerComponent` call:

```
type state = {counter: int, showPopUp: bool};

let component = ReasonReact.reducerComponent "Dialog";

let make ::onClick _children => {
  ...component,
  initialState: fun () => {counter: 0, showPopUp: false},
  render: ...
};
```

Since the props are just the arguments on `make`, feel free to read into them to initialize your state based on them.

## Actions & Reducer

In ReactJS, you'd update the state inside a callback handler, e.g.

```
{
  ...
  handleClick: function() {
    this.setState({count: this.state.count + 1});
  },
  handleSubmit: function() {
    this.setState(...);
  },
  render: function() {
    return (
      <MyForm
        onClick={this.handleClick}
        onSubmit={this.handleSubmit} />
    );
  }
}
```

In ReasonReact, you'd gather all these state-setting handlers into a single place, the component's `reducer`! Here's a full example, which we'll explain in detail:

```reason
type action =
  | Click
  | Toggle;

type state = {count: int, show: bool};

let component = ReasonReact.reducerComponent "MyForm";

let make _children => {
  ...component,
  initialState: fun () => {count: 0, show: false},
  reducer: fun action state =>
    switch action {
    | Click => ReasonReact.Update {...state, count: state.count + 1}
    | Toggle => ReasonReact.Update {...state, show: not state.show}
    },
  render: fun self => {
    let message = "Clicked " ^ string_of_int self.state.count ^ " times(s)";
    <div>
      <MyDialog
        onClick=(self.reduce (fun _event => Click))
        onSubmit=(self.reduce (fun _event => Toggle)) />
      (ReasonReact.stringToElement message)
    </div>
  }
};
```

A few things:

- There's a user-defined type called **`actions`**, named so by convention. It's a variant of all the possible state transitions in your component. _In state machine terminology, this'd be a "token"_.
- A user-defined `state` type, and an `initialState`. Nothing special.
- The current `state` value is accessible through `self.state`, whenever `self` is passed to you as an argument of some function.
- A "**reducer**"! This [pattern-matches](https://reasonml.github.io/guide/language/pattern-matching) on the possible actions and specify what state update each action corresponds to. _In state machine terminology, this'd be a "state transition"_.
- In `render`, instead of `self.handle` (which doesn't allow state updates), you'd use `self.reduce`. `reduce` takes a callback, passing it the event (or whatever callback payload `onSubmit`/`onClick`/`onFoo` gives you from `MyDialog`) and asking for an action as the return value.

So, when a click on the dialog is triggered, we send the `Click` action to the reducer, which handles the `Click` case by returning the new state that increment a counter. ReasonReact takes the state and updates the component.

**Note**: just like for `self.handle`, sometimes you might be forwarding `reduce` to some helper functions. Pass the whole `self` instead and **annotate it**. This avoids a complex `self` record type behavior. See [Common Type Errors](common-errors.md). Example:

## State Update Through Reducer

Notice the return value of `reducer`? The `ReasonReact.Update` part. Instead of returning a bare new state, we ask you to return the state wrapped in this "update" variant. Here are its possible values:

- `ReasonReact.NoUpdate`: don't do a state update.
- `ReasonReact.Update state`: update the state.
- `ReasonReact.SideEffects (self => unit)`: no state update, but trigger a side-effect, e.g. `ReasonReact.SideEffects (fun _self => Js.log "hello!")`.
- `ReasonReact.UpdateWithSideEffects state (self => unit)`: update the state, **then** trigger a side-effect.

_If you're a power user, there's also `SilentUpdate` and `SilentUpdateWithSideEffects`. See reasonReact.rei to see what they do. Don't use them if you're trying to update a ref/timer/subscription/any other instance variable_.

### Important Notes

**Please read through all these points**, if you want to fully take advantage of `reducer` and avoid future ReactJS Fiber race condition problems.

- The `action` type's variants can carry a payload: `onClick=(self.reduce (fun data => Click data.foo))`.
- Don't pass the whole event into the action variant's payload. ReactJS events are pooled; by the time you intercept the action in the `reducer`, the event's already recycled.
- `reducer` must be pure (not to be confused with `self.reduce`, which can be impure)! Don't do side-effects in them directly. You'll thank us when we enable the upcoming concurrent React (Fiber). Use `SideEffects` or `UpdateWithSideEffects` to enqueue a side-effect. The side-effect (the callback) will be executed after the state setting, but before the next render.
- If you need to do e.g. `ReactEventRe.BlablaEvent.preventDefault event`, do it in `self.reduce`, before returning the action type. Again, `reducer` must be pure.
- If your state only holds instance variables, it also means (by the convention in the instance variables section) that your component only contains `self.handle`, no `self.reduce`. You still needs to specify a `reducer` like so: `reducer: fun () _state => ReasonReact.NoUpdate`. Otherwise you'll get a `variable cannot be generalized` type error.

## Async State Setting

In ReactJS, you could use `setState` inside a callback, like so:

```
setInterval(() => this.setState(...), 1000);
```

In ReasonReact, since the new state, if any, is returned from the `reducer`, the above wouldn't work; naturally, returning a new state from a `setInterval` doesn't make sense!

Instead, you'd do:

```reason
type action =
  | Tick;

type state = {
  count: int,
  timerId: ref (option Js.Global.intervalId)
};

let component = ReasonReact.reducerComponent "Counter";

let make _children => {
  ...component,
  initialState: fun () => {count: 0, timerId: ref None},
  reducer: fun action state =>
    switch action {
    | Tick => ReasonReact.Update {...state, count: state.count + 1}
    },
  didMount: fun self => {
    self.state.timerId := Some (Js.Global.setInterval (self.reduce (fun _ => Tick)) 1000);
    ReasonReact.NoUpdate
  },
  render: fun {state} => <div> (ReasonReact.stringToElement (string_of_int state.count)) </div>
};
```

Aka, creating a `reducer` handler as you would normally, and let that setInterval (or whatver async state setting you use) asynchronously call the callback returned by your `self.reduce`.
