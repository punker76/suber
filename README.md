# suber - a message bus compatible with Redux middlewares
A message bus / pubsub prepared for usage in combination with
other bus/pubsub systems. For example, Redux to take advantage of the Redux
ecosystem for handling side effects, but without having to store
everything in the Redux global store.

## Use cases
### Stand-alone
It can be used as the main message bus / event emitter / pubsub
in an application to keep components separated without any dependencies (except for suber)
and be able to communicate.

For code testability it's better to have all side effect handlers listening and acting on
messages on a bus rather than to have async / ajax calls directly in the components.
With subers Redux middleware compability, great middlewares like
[redux-observable](https://redux-observable.js.org) and
[redux-saga](https://redux-saga.github.io/redux-saga/) can be used to handle the side effects.

On the other hand, creating a suber middleware is super easy so having direct
[fetch](https://developer.mozilla.org/en/docs/Web/API/Fetch_API) calls
in a self-made middleware is often enough for smaller applications.

### In combination with Redux
Redux is great for shared application state handling, but often too much state are stored in Redux.
This makes the reducer and action files big and the amount of boilerplate code needed to
type increases.

Application state that is not shared should not go into Redux, it should go directly,
and only into, the component that needs it.

This is where suber comes in.

Setup your Redux middleware side-effect handler to listen for certain action types
(`GET_USER` in the below example) and pass suber to your component (or have a container component
handle the calls and pass the response as props to it).
Your component can now super easy with

```javascript
bus.one('GOT_USER_1', (r) => this.setState({user: r.user}))
bus.send('GET_USER', {id: 1})
```

do a ajax call and set the component state so component can update the DOM can accordingly.
Now all code with side effects is in the same place no matter if the endpoint for the
response is Redux or single component state.

## Installation
suber is published on npm: https://www.npmjs.com/package/suber

```bash
yarn add suber
# or
npm install suber
```

## Usage
Stand-alone usage:

```javascript
// Import
import { getBus } from 'suber'

// Get bus
const bus = getBus()

// Listen until manual unsubscription
const unsub = bus.take('MY_CHANNEL', (data) => {
  console.log(data)
})
// or just one (unsubscribes automatically)
bus.one('MY_CHANNEL', (data) => {
  console.log(data)
})

// Send
bus.send('MY_CHANNEL', 'Hello world')
```

Usage with Redux.
This way we can use local state only but still take advantage of
the Redux ecosystem of great middlewares.

```javascript
// app.js
// App init file
import { applyMiddleware as applySuberMiddleware, createReduxMiddleWare } from 'suber'
import { createStore, applyMiddleware } from 'redux'

// Init
const mw = createReduxMiddleWare()
// Next line enables sending actions from Redux into suber
let store = createStore(yourApp, applyMiddleware(mw))

// Send everything from suber into Redux
applySuberMiddleware((_) => (channel, message, source) => {
  // No loop-backs
  if (source === 'redux') return
  // Send to Redux with the channel as the action type
  store.dispatch({...message, type: channel})
})

// Imagine you have redux-saga setup listening for actions
// with the type: 'GET_USER' and dispatches a new action with
// type 'GET_USER_RESPONSE_' + responseAction (to make it "random" to allow concurrency)
// when response arrives from API.

// ----

// component.js
// React component or anything that has been given access to the suber bus
// either via direct import or through a jsx property.

// Prepare a somewhat random channel / redux action
const responseAction = Math.floor(Math.round()*100 + 1)

// Listen for a single message on that channel
bus.one('GET_USER_RESPONSE_' + responseAction, (data) => {
  console.log(data)
})

// Ask for user with id = 1 and tell redux-saga what output
// type suffix you want the response to have.
bus.send('GET_USER', {id: 1, responseAction })
```


## API
### Factory
- [`getBus`](#getBus)

### Methods
- [`take`](#take)
- [`one`](#one)
- [`send`](#send)

### Utility functions
- [`applyMiddleware`](#applyMiddleware)
- [`applyReduxMiddleware`](#applyReduxMiddleware)
- [`createReduxMiddleWare`](#createReduxMiddleWare)

## Factory
Factories are functions that returns you the bus.

### <a id="getBus"></a> `getBus()`
Returns the singleton bus.

## Methods
These are methods that are attached to the bus instance.
### <a id="take"></a> `take(channel, fn, filterFn)`
Sets up a listener on the bus and calls `fn` every time a message with a matching `channel` arrives.
#### Arguments
- `channel: String` What channel to listen on the bus.
- `fn: Function(message)` The function to call when a message on the channel arrives.
- `filterFn: Function(message)` An optional filter function that can be used to just listen for a specific kind of data
on the channel. See test file for example.

#### Returns `unsubscribe: Function` Call this function to unsubscribe this `fn` on the channel.

### <a id="one"></a> `one(channel, fn, filterFn)`
Sets up a listener on the bus and calls `fn` **one time** time once a message with a matching `channel` arrives.
Just like `take` above but with automatic unsubscription after the first message.

### <a id="send"></a> `send(channel, message, source)`
Send a message on a channel on the bus.
#### Arguments
- `channel: String` The channel to send the message on.
- `message: any` The message, can be of any data type.
- `source: String` Optional argument to specify the message source.
Best used when `applyMiddleware` and `createReduxMiddleWare` both are specified to avoid loop-backs.

#### Returns `void`

## Utility functions
Functions to configure or extend the bus.
### <a id="applyMiddleware"></a> `applyMiddleware(fn)`
Add middleware to Suber. All messages on all channels gets passed to the middleware.
#### Arguments
- `fn: Function(channel, message, source)` The function to be called with every message on the bus.

#### Returns `void`

### <a id="applyReduxMiddleware"></a> `applyReduxMiddleware(rmw)`
Add a Redux middleware to Suber. This way the excellent Redux middlewares for
handling side effects like redux-saga and redux-observable can be used without
Redux.
The params for `store.getState` and `next` are unsuable, so middlewares that rely on
these will not be compatible since Suber isn't about state / reducers.
#### Arguments
- `rmw: Function()` The Redux middleware with `(store) => (next) => (action) => ` signature.

#### Returns `void`

### <a id="createReduxMiddleWare"></a> `createReduxMiddleWare()`
Creates a function that has the redux middleware signature. This function repeats everything on
the Redux bus into the suber bus. Redux action types are mapped to be suber channels.

```javascript
redux.action = {
  type: 'MY_ACTION',
  some: 'data',
  more: 'data'
}
// becomes
suber.send(redux.action.type, redux.action, 'redux')
```

#### Returns `Function` to be passed into Redux's `applyMiddleware`

## Development setup

```bash
yarn
npm test
```
