# redux undo/redo

_simple undo/redo functionality for redux state containers, augmenting existing store structure_

[![https://i.imgur.com/M2KR4uo.gif](https://i.imgur.com/M2KR4uo.gif)](https://github.com/omnidan/redux-existing-undo-boilerplate)

---

# WARNING
 
 ## In order fot this to work, the reducer's slice has to be an object, and not a single value
 ```diff
 const appReducer = combineReducers({
  todos: todos.reducers
 )}
- var todosInitialState = 3
+ var todosInitialState = { value : 3 }
```


---

If you use Redux Undo in CommonJS environment, **don’t forget to add `.default` to your import**.

```diff
- var ReduxUndo = require('redux-existing-undo')
+ var ReduxUndo = require('redux-existing-undo').default
```

If your environment support es modules just go by:

```js
import ReduxUndo from 'redux-existing-undo';
```

We are also supporting UMD build:

```js
var ReduxUndo = window.ReduxUndo.default;
```

**once again `.default` is required.**

## Installation

```
npm install --save redux-existing-undo
```


## API

```js
import undoable from 'redux-existing-undo';
undoable(reducer)
undoable(reducer, config)
```


## Making your reducers undoable

`redux-existing-undo` is a reducer enhancer (higher-order reducer), it provides the `undoable` function, which
takes an existing reducer and a configuration object and enhances your existing
reducer with undo functionality.

To install, firstly import `redux-existing-undo`:

```js
// Redux utility functions
import { combineReducers } from 'redux';
// redux-existing-undo higher-order reducer
import undoable from 'redux-existing-undo';
```

Then, add `undoable` to your reducer(s) like this:

```js
combineReducers({
  counter: undoable(counter)
})
```

A [configuration](#configuration) can be passed like this:

```js
combineReducers({
  counter: undoable(counter, {
    limit: 10 // set a limit for the history
  })
})
```


## History API

Wrapping your reducer with `undoable` makes the state look like this:

```js
{
  past: [...pastStatesHere...],
  ...currentStateHere,
  future: [...futureStatesHere...]
}
```

Now you can get your current state like you already did.

And you can access all past states (e.g. to show a history) like this: `state.past`


## Undo/Redo Actions

Firstly, import the undo/redo action creators:

```js
import { ActionCreators } from 'redux-existing-undo';
```

Then, you can use `store.dispatch()` and the undo/redo action creators to
perform undo/redo operations on your state:

```js
store.dispatch(ActionCreators.undo()) // undo the last action
store.dispatch(ActionCreators.redo()) // redo the last action

store.dispatch(ActionCreators.jump(-2)) // undo 2 steps
store.dispatch(ActionCreators.jump(5)) // redo 5 steps

store.dispatch(ActionCreators.jumpToPast(index)) // jump to requested index in the past[] array
store.dispatch(ActionCreators.jumpToFuture(index)) // jump to requested index in the future[] array

store.dispatch(ActionCreators.clearHistory()) // [beta only] Remove all items from past[] and future[] arrays
```


## Configuration

A configuration object can be passed to `undoable()` like this (values shown
are default values):

```js
undoable(reducer, {
  limit: false, // set to a number to turn on a limit for the history

  filter: () => true, // see `Filtering Actions` section

  undoType: ActionTypes.UNDO, // define a custom action type for this undo action
  redoType: ActionTypes.REDO, // define a custom action type for this redo action

  jumpType: ActionTypes.JUMP, // define custom action type for this jump action

  jumpToPastType: ActionTypes.JUMP_TO_PAST, // define custom action type for this jumpToPast action
  jumpToFutureType: ActionTypes.JUMP_TO_FUTURE, // define custom action type for this jumpToFuture action

  clearHistoryType: ActionTypes.CLEAR_HISTORY, // [beta only] define custom action type for this clearHistory action
  // you can also pass an array of strings to define several action types that would clear the history
  // beware: those actions will not be passed down to the wrapped reducers

  initTypes: ['@@redux-existing-undo/INIT'] // history will be (re)set upon init action type
  // beware: those actions will not be passed down to the wrapped reducers

  debug: false, // set to `true` to turn on debugging

  neverSkipReducer: false, // prevent undoable from skipping the reducer on undo/redo and clearHistoryType actions
})
```

**Note:** If you want to use just the `initTypes` functionality, but not import
the whole redux-existing-undo library, use [redux-recycle](https://github.com/omnidan/redux-recycle)!

#### Initial State and History

You can use your redux store to set an initial history for your undoable reducers:

```js

import { createStore } from 'redux';

const initialHistory = {
  past: [{value: 2},{value: 3}],
  {value: 4},
  future: [{value: 5},{value: 6}]
}

const store = createStore(undoable(counter), initialHistory);

```

Or just set the current state like you're used to with Redux. redux-existing-undo will create the history for you:

```js

import { createStore } from 'redux';

const store = createStore(undoable(counter), {foo: 'bar'});

// will make the state look like this:
{
  past: [],
  foo: 'bar',
  future: []
}

```

### Filtering Actions

If you don't want to include every action in the undo/redo history, you can
add a `filter` function to `undoable`. `redux-existing-undo` provides you with the
`includeAction` and `excludeAction` helpers for basic filtering.

They should be imported like this:

```js
import undoable, { includeAction, excludeAction } from 'redux-existing-undo';
```

Now you can use the helper functions:

```js
undoable(reducer, { filter: includeAction(SOME_ACTION) })
undoable(reducer, { filter: excludeAction(SOME_ACTION) })

// they even support Arrays:

undoable(reducer, { filter: includeAction([SOME_ACTION, SOME_OTHER_ACTION]) })
undoable(reducer, { filter: excludeAction([SOME_ACTION, SOME_OTHER_ACTION]) })
```

#### Custom filters

If you want to create your own filter, pass in a function with the signature
`(action, currentState, previousHistory)`. For example:

```js
undoable(reducer, {
  filter: function filterActions(action, currentState, previousHistory) {
    return action.type === SOME_ACTION; // only add to history if action is SOME_ACTION
  }
})

// The entire `history` state is available to your filter, so you can make
// decisions based on past or future states:

undoable(reducer, {
  filter: function filterState(action, currentState, previousHistory) {
    let { past, present, future } = previousHistory;
    return future.length === 0; // only add to history if future is empty
  }
})
```

### Combining Filters

You can also use our helper to combine filters.

```js
import undoable, {combineFilters} from 'redux-existing-undo'

function isActionSelfExcluded(action) {
  return action.wouldLikeToBeInHistory
}

function areWeRecording(action, state) {
  return state.recording
}

undoable(reducer, {
  filter: combineFilters(isActionSelfExcluded, areWeRecording)
})
```

### Ignoring Actions

When implementing a filter function, it only prevents the old state from being
stored in the history. **`filter` does not prevent the present state from being
updated.**

If you want to ignore an action completely, as in, not even update the present
state, you can make use of [redux-ignore](https://github.com/omnidan/redux-ignore).

It can be used like this:

```js
import { ignoreActions } from 'redux-ignore'

ignoreActions(
  undoable(reducer),
  [IGNORED_ACTION, ANOTHER_IGNORED_ACTION]
)

// or define your own function:

ignoreActions(
  undoable(reducer),
  (action) => action.type === SOME_ACTION // only add to history if action is SOME_ACTION
)
```


## What is this magic? How does it work?

Have a read of the [Implementing Undo History recipe](http://redux.js.org/docs/recipes/ImplementingUndoHistory.html) in the Redux documents, which explains in detail how redux-existing-undo works.

## License

MIT, see `LICENSE.md` for more information.
