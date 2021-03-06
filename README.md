# redux-actions

[![build status](https://img.shields.io/travis/acdlite/redux-actions/master.svg?style=flat-square)](https://travis-ci.org/acdlite/redux-actions)

[![NPM](https://nodei.co/npm/redux-actions.png?downloads=true)](https://nodei.co/npm/redux-actions/)

[Flux Standard Action](https://github.com/acdlite/flux-standard-action) utilities for Redux.

## Installation

```bash
npm install --save redux-actions
```

If you don’t use [npm](https://www.npmjs.com), you may grab the latest [UMD](https://unpkg.com/redux-actions@latest/dist) build from [unpkg](https://unpkg.com) (either a [development](https://unpkg.com/redux-actions@latest/dist/redux-actions.js) or a [production](https://unpkg.com/redux-actions@latest/dist/redux-actions.min.js) build). The UMD build exports a global called `window.ReduxActions` if you add it to your page via a `<script>` tag. We *don’t* recommend UMD builds for any serious application, as most of the libraries complementary to Redux are only available on [npm](https://www.npmjs.com/search?q=redux).

## Usage

### `createAction(type, payloadCreator = Identity, ?metaCreator)`

```js
import { createAction } from 'redux-actions';
```

Wraps an action creator so that its return value is the payload of a Flux Standard Action. If no payload creator is passed, or if it's not a function, the identity function is used.

Example:

```js
let increment = createAction('INCREMENT', amount => amount);
// same as
increment = createAction('INCREMENT');

expect(increment(42)).to.deep.equal({
  type: 'INCREMENT',
  payload: 42
});
```

If the payload is an instance of an [Error
object](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Error),
redux-actions will automatically set ```action.error``` to true.

Example:

```js
const increment = createAction('INCREMENT');

const error = new TypeError('not a number');
expect(increment(error)).to.deep.equal({
  type: 'INCREMENT',
  payload: error,
  error: true
});
```

`createAction` also returns its `type` when used as type in `handleAction` or `handleActions`.

Example:

```js
const increment = createAction('INCREMENT');

// As parameter in handleAction:
handleAction(increment, {
  next(state, action) {...},
  throw(state, action) {...}
});

// As object key in handleActions:
const reducer = handleActions({
  [increment]: (state, action) => ({
    counter: state.counter + action.payload
  })
}, { counter: 0 });
```

**NOTE:** The more correct name for this function is probably `createActionCreator()`, but that seems a bit redundant.

Use the identity form to create one-off actions:

```js
createAction('ADD_TODO')('Use Redux');
```

`metaCreator` is an optional function that creates metadata for the payload. It receives the same arguments as the payload creator, but its result becomes the meta field of the resulting action. If `metaCreator` is undefined or not a function, the meta field is omitted.

### `createActions(?actionsMap, ?...identityActions)`

```js
import { createActions } from 'redux-actions';
```

Returns an object mapping action types to action creators. The keys of this object are camel-cased from the keys in `actionsMap` and the string literals of `identityActions`; the values are the action creators.

`actionsMap` is an optional object with action types as keys, and whose values **must** be either

- a function, which is the payload creator for that action
- an array with `payload` and `meta` functions in that order, as in [`createAction`](#createactiontype-payloadcreator--identity-metacreator)
    - `meta` is **required** in this case (otherwise use the function form above)

`identityActions` is an optional list of positional string arguments that are action type strings; these action types will use the identity payload creator.

```js
const { actionOne, actionTwo, actionThree } = createActions({
  // function form; payload creator defined inline
  ACTION_ONE: (key, value) => ({ [key]: value }),

  // array form
  ACTION_TWO: [
    (first) => first,               // payload
    (first, second) => ({ second }) // meta
  ],

  // trailing action type string form; payload creator is the identity
}, 'ACTION_THREE');

expect(actionOne('key', 1)).to.deep.equal({
  type: 'ACTION_ONE',
  payload: { key: 1 }
});

expect(actionTwo('first', 'second')).to.deep.equal({
  type: 'ACTION_TWO',
  payload: ['first'],
  meta: { second: 'second' }
});

expect(actionThree(3)).to.deep.equal({
  type: 'ACTION_THREE',
  payload: 3,
});
```

### `handleAction(type, reducer | reducerMap, ?defaultState)`

```js
import { handleAction } from 'redux-actions';
```

Wraps a reducer so that it only handles Flux Standard Actions of a certain type.

If a single reducer is passed, it is used to handle both normal actions and failed actions. (A failed action is analogous to a rejected promise.) You can use this form if you know a certain type of action will never fail, like the increment example above.

Otherwise, you can specify separate reducers for `next()` and `throw()`. This API is inspired by the ES6 generator interface.

```js
handleAction('FETCH_DATA', {
  next(state, action) {...},
  throw(state, action) {...}
});
```

If either `next()` or `throw()` are `undefined` or `null`, then the identity function is used for that reducer.

The optional third parameter specifies a default or initial state, which is used when `undefined` is passed to the reducer.

### `handleActions(reducerMap, ?defaultState)`

```js
import { handleActions } from 'redux-actions';
```

Creates multiple reducers using `handleAction()` and combines them into a single reducer that handles multiple actions. Accepts a map where the keys are passed as the first parameter to `handleAction()` (the action type), and the values are passed as the second parameter (either a reducer or reducer map).

The optional second parameter specifies a default or initial state, which is used when `undefined` is passed to the reducer.

(Internally, `handleActions()` works by applying multiple reducers in sequence using [reduce-reducers](https://github.com/acdlite/reduce-reducers).)

Example:

```js
const reducer = handleActions({
  INCREMENT: (state, action) => ({
    counter: state.counter + action.payload
  }),

  DECREMENT: (state, action) => ({
    counter: state.counter - action.payload
  })
}, { counter: 0 });
```

### `combineActions(...actionTypes)`

Combine any number of action types or action creators. `actionTypes` is a list of positional arguments which can be action type strings, symbols, or action creators.

This allows you to reduce multiple distinct actions with the same reducer.

```js
const { increment, decrement } = createActions({
  INCREMENT: amount => ({ amount }),
  DECREMENT: amount => ({ amount: -amount }),
})

const reducer = handleAction(combineActions(increment, decrement), {
  next: (state, { payload: { amount } }) => ({ ...state, counter: state.counter + amount }),
  throw: state => ({ ...state, counter: 0 }),
}, { counter: 10 })

expect(reducer(undefined, increment(1)).to.deep.equal({ counter: 11 })
expect(reducer(undefined, decrement(1)).to.deep.equal({ counter: 9 })
expect(reducer(undefined, increment(new Error)).to.deep.equal({ counter: 0 })
expect(reducer(undefined, decrement(new Error)).to.deep.equal({ counter: 0 })
```

## Usage with middleware

redux-actions is handy all by itself, however, its real power comes when you combine it with middleware.

The identity form of `createAction` is a great way to create a single action creator that handles multiple payload types. For example, using [redux-promise](https://github.com/acdlite/redux-promise) and [redux-rx](https://github.com/acdlite/redux-rx):

```js
const addTodo = createAction('ADD_TODO');

// A single reducer...
handleAction('ADD_TODO', (state = { todos: [] }, action) => ({
  ...state,
  todos: [...state.todos, action.payload]
}));

// ...that works with all of these forms:
// (Don't forget to use `bindActionCreators()` or equivalent.
// I've left that bit out)
addTodo('Use Redux')
addTodo(Promise.resolve('Weep with joy'));
addTodo(Observable.of(
  'Learn about middleware',
  'Learn about higher-order stores'
)).subscribe();
```

## See also

Use redux-actions in combination with FSA-compliant libraries.

- [redux-promise](https://github.com/acdlite/redux-promise) - Promise middleware
- [redux-rx](https://github.com/acdlite/redux-rx) - Includes observable middleware.
