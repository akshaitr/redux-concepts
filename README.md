## Table of Contents

1. [The Mental Model: What Middleware Actually Is](#the-mental-model-what-middleware-actually-is)
2. [The `dispatch` Pipeline](#the-dispatch-pipeline)
3. [`applyMiddleware` — Reading the Source](#applymiddleware--reading-the-source)
4. [Writing Middleware from Scratch](#writing-middleware-from-scratch)
5. [Common Middleware Patterns](#common-middleware-patterns)
6. [Redux-Saga — Mental Model](#redux-saga--mental-model)
7. [Generators — The Foundation](#generators--the-foundation)
8. [Effects — Declarative Side Effects](#effects--declarative-side-effects)
9. [The Saga Middleware Internals](#the-saga-middleware-internals)
10. [Effect-by-Effect Mechanics](#effect-by-effect-mechanics)
11. [Concurrency Patterns](#concurrency-patterns)
12. [Channels — The Low-Level Primitive](#channels--the-low-level-primitive)
13. [Testing Sagas](#testing-sagas)
14. [Saga vs Thunk vs RTK Query](#saga-vs-thunk-vs-rtk-query)
15. [Common Pitfalls](#common-pitfalls)

---

## The Mental Model: What Middleware Actually Is

Redux middleware is **a chain of functions that wrap `store.dispatch`**. Each middleware gets a chance to inspect, modify, delay, or replace an action before it reaches the reducer.

The signature is famously curried:

```js
const middleware = store => next => action => { ... };
```

Three layers, each providing access to different scopes:

1. **`store`** — `{ getState, dispatch }` of the *enhanced* store. Captured once when middleware initializes.
2. **`next`** — the next middleware's `action` handler in the chain. Captured once per dispatch chain.
3. **`action`** — the actual action being dispatched. Received per dispatch.

The currying isn't aesthetic — it's how Redux composes middlewares into a chain. Each layer is "frozen" at the right time. Without currying, you'd have to pass everything to every call.

### The chain

If you have `[m1, m2, m3]`, the effective dispatch becomes:

```
dispatch(action)
  → m1(action)
    → next1 = m2(action)  
      → next2 = m3(action)
        → next3 = store.dispatch(action)  // reaches reducer
```

`m1` sits closest to the caller. `m3` sits closest to the reducer. Each middleware can:
- Call `next(action)` to pass it down the chain
- Skip `next(action)` to swallow the action entirely
- Modify the action and pass the modified version
- Dispatch *new* actions via `store.dispatch` (re-entering the top of the chain)
- Do async work and dispatch later

That last bullet is the key insight — `store.dispatch` (the one captured in the closure) goes through the *entire* chain again, while `next` goes only to the *next* middleware down. The distinction is the source of most middleware bugs.

---

## The `dispatch` Pipeline

Before middleware, `store.dispatch(action)`:

1. Sets an internal `isDispatching` flag
2. Calls the root reducer: `state = reducer(state, action)`
3. Clears the flag
4. Notifies all subscribers (`store.subscribe` listeners)
5. Returns the action

That's it. Synchronous, single-threaded, no async, no side effects.

`applyMiddleware` wraps this with the middleware chain. After application, when you call `store.dispatch`, you're actually calling the *outermost middleware's* `action` handler. The original `dispatch` (the one that hits the reducer) is only reachable at the end of the chain.

### Why this design

Redux's core stays pure — reducers are still `(state, action) => newState`, deterministic and testable. All the messy real-world stuff (async, logging, analytics, optimistic updates, conditional dispatch) lives in middleware, layered on top.

This is also why Redux DevTools time-travel works: replay any sequence of actions through the reducer, and you get the same state. Middleware side effects are *not* replayed — they're treated as out-of-band.

---

## `applyMiddleware` — Reading the Source

The actual Redux source is ~20 lines. Worth reading once:

```js
function applyMiddleware(...middlewares) {
  return createStore => (reducer, preloadedState) => {
    const store = createStore(reducer, preloadedState);
    
    let dispatch = () => {
      throw new Error('Dispatching during construction is not allowed');
    };
    
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args),
    };
    
    const chain = middlewares.map(middleware => middleware(middlewareAPI));
    dispatch = compose(...chain)(store.dispatch);
    
    return { ...store, dispatch };
  };
}
```

Three things worth noting:

### 1. The `dispatch` indirection

`middlewareAPI.dispatch` is defined as `(action, ...args) => dispatch(action, ...args)` — a wrapper around the variable `dispatch`, not a direct reference. This is deliberate.

When middlewares initialize (the `middleware(middlewareAPI)` call), `dispatch` is still the "throws an error" stub. If we passed `store.dispatch` directly, middlewares that try to dispatch during *initialization* would bypass the chain entirely.

By the time anyone calls `middlewareAPI.dispatch(action)`, the variable has been reassigned to the composed chain. So calls work, and they go through *every* middleware.

The error-throwing stub catches the case where a middleware tries to dispatch synchronously during its own setup — which is meaningless because the chain isn't built yet.

### 2. `compose(...chain)`

`compose` is right-to-left function composition:

```js
compose(f, g, h)(x) === f(g(h(x)))
```

Applied to the chain `[m1, m2, m3]`:

```js
compose(m1_n, m2_n, m3_n)(store.dispatch)
// === m1_n(m2_n(m3_n(store.dispatch)))
```

Where `m1_n` is `m1`'s `next => action => ...` function (the result of `m1(middlewareAPI)`).

Working outside-in:
- `m3_n(store.dispatch)` returns `m3`'s action handler, with `next = store.dispatch`
- `m2_n(...)` returns `m2`'s action handler, with `next = m3`'s handler
- `m1_n(...)` returns `m1`'s action handler, with `next = m2`'s handler

The result is `m1`'s action handler, which is what becomes the new `store.dispatch`.

### 3. The returned store

`return { ...store, dispatch }` — Redux spreads the original store and overrides `dispatch`. `getState`, `subscribe`, `replaceReducer` are unchanged. This means middleware cannot intercept subscriptions or state reads — only dispatches.

---

## Writing Middleware from Scratch

The classic logger:

```js
const logger = store => next => action => {
  console.group(action.type);
  console.log('prev state', store.getState());
  console.log('action', action);
  const result = next(action);
  console.log('next state', store.getState());
  console.groupEnd();
  return result;
};
```

Things to notice:
- We call `next(action)` to let the action reach the reducer
- We read state *before* and *after* `next` — that's where the state actually changes
- We return whatever `next` returns — so downstream callers (or other middleware) get the same return value

### A middleware that swallows actions

```js
const blockTypes = blocklist => store => next => action => {
  if (blocklist.includes(action.type)) return; // swallow
  return next(action);
};
```

Useful for feature flags, A/B test gating, etc.

### A middleware that transforms actions

```js
const addTimestamp = store => next => action => {
  return next({
    ...action,
    meta: { ...action.meta, timestamp: Date.now() },
  });
};
```

### A middleware that dispatches new actions

```js
const crashReporter = store => next => action => {
  try {
    return next(action);
  } catch (err) {
    store.dispatch({ type: 'ERROR', error: err.message });
    throw err;
  }
};
```

Note `store.dispatch` here, not `next`. If we used `next`, the error action would skip middlewares above us — the logger wouldn't log it, analytics wouldn't track it. Using `store.dispatch` re-enters the chain at the top.

### Async via promises

This is essentially `redux-thunk` in two lines:

```js
const thunk = store => next => action => {
  if (typeof action === 'function') {
    return action(store.dispatch, store.getState);
  }
  return next(action);
};
```

If the "action" is actually a function, call it with `(dispatch, getState)`. The function can do whatever it wants — async work, conditional dispatch, multiple dispatches.

That's it. The entire library is ~10 lines. The reason saga exists is because thunk's simplicity becomes a liability at scale (more on that below).

---

## Common Middleware Patterns

### Order matters

Middleware order determines the chain order. A typical chain:

```js
applyMiddleware(
  crashReporter,  // outermost — catches errors from everything below
  logger,         // logs all actions, including ones from other middleware
  promise,        // unwraps promise actions
  saga,           // intercepts and runs sagas
  thunk,          // handles function actions
);
```

Putting the logger *after* saga means you'd miss the original `FETCH_USER_REQUESTED` action and only see the resulting `FETCH_USER_SUCCEEDED` — usually not what you want.

Putting `crashReporter` outermost ensures errors from any other middleware are caught.

### Why not just put async in reducers?

You'd lose:
- Determinism (reducers can't be replayed without side effects)
- Testability (every reducer test needs network mocks)
- Time-travel debugging
- Pure-function reasoning

The whole point of Redux is that the reducer is pure. Async lives in middleware.

### The `Symbol.observable` interop

Redux stores implement the Observable interop spec — `store[Symbol.observable]()` returns a minimal observable. Middlewares like `redux-observable` use this to bridge Redux to RxJS, so you can write epics as observable transformations of the action stream.

Not commonly needed, but if you see `redux-observable` in a codebase, that's the bridge.

---

## Redux-Saga — Mental Model

Saga is a middleware that runs **long-lived background processes** alongside your app. These processes are written as ES6 generator functions and communicate with Redux via *effects* — plain JS objects describing operations the middleware should perform.

The mental model that makes sagas click:

> **A saga is like a daemon thread that watches the action stream and produces side effects, controlled via a coroutine protocol.**

Sagas don't *call* `dispatch` directly. They `yield` effect objects, and the middleware interprets them — running async work, dispatching actions, waiting for other actions, forking subprocesses. The saga is paused between yields.

This indirection gives you three big wins over thunk:

1. **Pure logic** — sagas describe what should happen; the middleware does it. You can test a saga by stepping through yields without mocking anything.
2. **Cancellation** — sagas can be cancelled cleanly because the middleware controls execution. With thunks, once an async function is running, you can't pause or cancel it.
3. **Complex coordination** — race conditions, throttling, debouncing, retries, parent/child task lifecycles — these are 2-line effects in saga, but bespoke code in thunk.

The cost: a steeper learning curve, generator syntax that some teams find alien, and more lines of code for simple async cases.

---

## Generators — The Foundation

Generators are the language feature sagas are built on. If they're hazy, the rest of saga won't click.

### The basics

```js
function* counter() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = counter();
gen.next(); // { value: 1, done: false }
gen.next(); // { value: 2, done: false }
gen.next(); // { value: 3, done: false }
gen.next(); // { value: undefined, done: true }
```

Calling a generator function doesn't run the body — it returns an *iterator*. The body runs only when `.next()` is called, and it pauses at every `yield`. Calling `.next()` again resumes from where it stopped.

### Two-way communication

The `value` argument to `.next(value)` becomes the result of the `yield` expression:

```js
function* dialog() {
  const name = yield 'What is your name?';
  const age = yield `Hi ${name}, what is your age?`;
  return `${name} is ${age}`;
}

const g = dialog();
g.next();          // { value: 'What is your name?', done: false }
g.next('Akshai'); // { value: 'Hi Akshai, what is your age?', done: false }
g.next(31);        // { value: 'Akshai is 31', done: true }
```

This bidirectional flow is what makes generators useful for sagas. The saga `yield`s an instruction; the middleware sends back a result via `.next(result)`.

### Throwing into a generator

```js
function* example() {
  try {
    const result = yield somethingThatMightFail();
  } catch (err) {
    yield handleError(err);
  }
}

const g = example();
g.next();
g.throw(new Error('boom')); // jumps to the catch block
```

This is how sagas handle errors from async effects — the middleware calls `.throw()` on the generator when a yielded operation rejects.

### `yield*` — delegation

```js
function* sub() { yield 1; yield 2; }
function* main() {
  yield 0;
  yield* sub();  // delegates to sub's iterator
  yield 3;
}
// Sequence: 0, 1, 2, 3
```

Sagas use this for composition — one saga can `yield*` another and inherit its behavior.

### Why generators for sagas

A saga is essentially a coroutine — a function that can pause, yield control, and be resumed externally. Promises can't do this (you can't pause a `then` chain). `async/await` can't do this (you can't externally cancel an `await`). Only generators give the *middleware* control over execution.

---

## Effects — Declarative Side Effects

The saga yields **effect objects**, not actions. An effect is a plain JS object describing what the middleware should do:

```js
{ '@@redux-saga/IO': true, type: 'CALL', payload: { fn: fetchUser, args: [id] } }
```

You don't write these literals — you use creator functions:

```js
import { call, put, take, fork } from 'redux-saga/effects';

call(fetchUser, id)              // → CALL effect
put({ type: 'FETCH_SUCCESS' })   // → PUT effect (dispatch)
take('LOGIN_REQUESTED')          // → TAKE effect (wait for action)
fork(watcherSaga)                // → FORK effect (spawn child)
```

The middleware interprets each effect type:
- `CALL` → invoke the function, suspend the saga until result/error
- `PUT` → dispatch the action via `store.dispatch`
- `TAKE` → suspend until a matching action is dispatched
- `FORK` → start a detached saga (non-blocking)
- etc.

### Why effects instead of direct calls?

If the saga did `const data = yield fetchUser(id)`, the saga would *actually call* `fetchUser` — making real network requests during tests. Worse, the test couldn't easily intercept it.

By yielding `call(fetchUser, id)` instead, the saga only *describes* the intent. The middleware does the actual call. In tests, you assert that the saga yielded `call(fetchUser, 5)` — no mocking required.

This is the same separation-of-concerns idea as React's render/commit split, or `dispatch(action)` vs reducer execution. **Describe the operation, let the runtime perform it.**

---

## The Saga Middleware Internals

Roughly, the middleware does this:

```js
function createSagaMiddleware() {
  let runSaga; // exposed for starting top-level sagas
  
  const middleware = store => next => action => {
    const result = next(action);  // let action reach reducer first
    sagaChannel.put(action);      // then notify sagas waiting on TAKE
    return result;
  };
  
  middleware.run = (saga, ...args) => {
    return runSaga({ dispatch: store.dispatch, getState: store.getState, channel: sagaChannel }, saga, ...args);
  };
  
  return middleware;
}
```

Two important details:

### 1. `next(action)` runs BEFORE notifying sagas

The action hits the reducer first, then sagas waiting on `take` are notified. This means by the time your saga's `take` resolves, `getState()` already reflects the action. If it were the other way around, you'd see stale state.

### 2. The saga channel

Internally, the middleware maintains a *channel* — a buffer of actions. When you `take('SOMETHING')`, the middleware registers your saga as a listener on this channel. When that action flows through middleware, listeners get notified and resume.

### Running a saga

When you call `sagaMiddleware.run(rootSaga)`:

1. The middleware calls `rootSaga()` to get a generator iterator
2. It calls `iterator.next()` to get the first yielded effect
3. It interprets the effect — performs the operation, gets a result
4. It calls `iterator.next(result)` to resume the saga with the result
5. Loop until `done: true`

If the effect's operation rejects, the middleware calls `iterator.throw(error)` instead — which jumps into the saga's nearest `try/catch`.

### Forks and the task tree

Every `fork` creates a child task. The parent saga gets a `Task` object (a handle to the child) and continues without waiting. Children run independently but are part of the parent's task tree.

When the parent finishes, the middleware waits for all attached forks to complete before considering the parent "done." If the parent is cancelled, all child forks are cancelled too.

This automatic lifecycle management is one of saga's biggest wins over thunk — child operations don't leak when their parent is torn down.

---

## Effect-by-Effect Mechanics

### `take(pattern)` — wait for an action

```js
const action = yield take('LOGIN_SUCCESS');
const action = yield take(['LOGIN_SUCCESS', 'LOGIN_FAILURE']);
const action = yield take(a => a.type.startsWith('USER_'));
```

The saga blocks until an action matching the pattern is dispatched. The matched action is returned (so you can read its payload).

`take` only matches actions dispatched *after* the `take` is yielded. Earlier actions are missed.

### `put(action)` — dispatch

```js
yield put({ type: 'SHOW_NOTIFICATION', message: 'Saved' });
```

Dispatches the action. Goes through the full middleware chain — including saga middleware, so other sagas listening via `take` will react.

`put` is non-blocking by default — the saga continues immediately. To wait for downstream effects, use the `resolve` channel pattern (rare).

### `call(fn, ...args)` — call a function

```js
const user = yield call(api.fetchUser, userId);
```

The middleware calls `fn(...args)`. If the result is a promise, the saga suspends until it resolves or rejects. If it's a plain value, the saga continues immediately.

`call` is what you use for any async work — fetch, axios, database access. Use it even for sync work when you want testability (assertions become trivial).

### `fork(fn, ...args)` — non-blocking call

```js
const task = yield fork(watcher);  // saga doesn't wait
```

Spawns the function as a child task and returns a `Task` object. The parent saga continues immediately.

Use `fork` when you want a saga to run alongside others — typical for watchers (see below).

### `spawn(fn, ...args)` — detached fork

Like `fork`, but the child is **detached** from the parent's task tree. If the parent is cancelled, the child is *not*. If the child crashes, the parent isn't affected.

Use `spawn` for truly independent processes. Use `fork` when child failure or cancellation should propagate up.

### `cancel(task)` — cancel a child

```js
yield cancel(task);
```

Cancels a forked task. The middleware calls `iterator.return()` on the cancelled saga, which runs any `finally` blocks for cleanup.

```js
function* downloader() {
  try {
    while (true) {
      yield call(api.downloadChunk);
    }
  } finally {
    if (yield cancelled()) {
      yield call(api.cleanupDownload);  // cleanup if cancelled
    }
  }
}
```

`yield cancelled()` returns true inside a finally block if the task was cancelled (as opposed to finishing normally).

### `cancelled()` — am I being cancelled?

Used in `finally` blocks to branch on cancellation vs normal completion (as above).

### `select(selector, ...args)` — read store state

```js
const userId = yield select(state => state.user.id);
```

Returns the result of `selector(state)`. Sync — no suspension.

This is the saga equivalent of `useSelector`. Use it sparingly — heavy state reads in sagas can become hard to reason about, and most sagas should be driven by actions (which carry their own payload).

### `takeEvery(pattern, saga)` — fork on every match

```js
yield takeEvery('FETCH_USER', handleFetchUser);
```

Equivalent to:

```js
while (true) {
  const action = yield take('FETCH_USER');
  yield fork(handleFetchUser, action);
}
```

For every `FETCH_USER` action, spawn a new `handleFetchUser` task. Multiple can run concurrently — if the user clicks "fetch" 5 times, 5 fetches run in parallel.

### `takeLatest(pattern, saga)` — cancel previous, fork new

```js
yield takeLatest('FETCH_USER', handleFetchUser);
```

Like `takeEvery`, but if a new matching action arrives while a previous handler is still running, the previous task is cancelled.

Perfect for search-as-you-type — each keystroke cancels the previous in-flight request. The user only sees the result for the latest input.

This is one of saga's killer features. The equivalent in thunk requires manual cancellation tokens, AbortControllers, or stale-response checks.

### `takeLeading(pattern, saga)` — ignore while busy

Opposite of `takeLatest`. Once a handler is running, new matching actions are ignored until it completes. Good for "don't double-submit" patterns.

### `throttle(ms, pattern, saga)` — rate-limit

```js
yield throttle(500, 'SCROLL', handleScroll);
```

Runs the saga at most once per `ms` milliseconds for the given pattern. The first matching action runs immediately; subsequent actions within the window are dropped (except the most recent, which runs at the end of the window).

### `debounce(ms, pattern, saga)` — wait for quiet

```js
yield debounce(500, 'INPUT_CHANGED', handleSearch);
```

Waits `ms` ms of silence before firing. Useful for search input — wait until the user stops typing.

### `delay(ms)` — sleep

```js
yield delay(1000);  // pause 1 second
```

Suspends the saga for the duration. Cancellable — if the saga is cancelled during the delay, the timer is cleared.

### `all([...effects])` — wait for all

```js
const [user, posts, comments] = yield all([
  call(api.fetchUser, id),
  call(api.fetchPosts, id),
  call(api.fetchComments, id),
]);
```

Runs effects in parallel. Resolves when all complete. If any rejects, all are cancelled.

Equivalent to `Promise.all`, but for effects.

### `race({...effects})` — first one wins

```js
const { posts, timeout } = yield race({
  posts: call(api.fetchPosts),
  timeout: delay(5000),
});

if (timeout) {
  yield put({ type: 'FETCH_TIMEOUT' });
}
```

Runs effects in parallel, returns the first to resolve. All others are cancelled.

Classic uses: timeouts, cancellation tokens, "wait for either of these actions."

---

## Concurrency Patterns

### Watcher / worker

The canonical saga structure:

```js
function* fetchUserWorker(action) {
  try {
    const user = yield call(api.fetchUser, action.payload.id);
    yield put({ type: 'FETCH_USER_SUCCESS', user });
  } catch (err) {
    yield put({ type: 'FETCH_USER_FAILURE', error: err.message });
  }
}

function* watchFetchUser() {
  yield takeEvery('FETCH_USER_REQUESTED', fetchUserWorker);
}

function* rootSaga() {
  yield all([
    fork(watchFetchUser),
    fork(watchOtherStuff),
  ]);
}
```

- **Worker** — handles one action, does the work, dispatches success/failure
- **Watcher** — runs forever, listens for actions, forks workers
- **Root** — composes all watchers, started once at app boot

### Search-as-you-type

```js
function* searchWorker(action) {
  yield delay(300);  // debounce
  const results = yield call(api.search, action.payload.query);
  yield put({ type: 'SEARCH_RESULTS', results });
}

function* watchSearch() {
  yield takeLatest('SEARCH_INPUT_CHANGED', searchWorker);
}
```

`takeLatest` cancels in-flight workers on new input. The `delay(300)` adds debouncing — but since cancellation is automatic, you don't need a separate timer to clear.

A user typing "react" produces 5 actions, but only the last `searchWorker` runs to completion. The others are cancelled during their `delay` (before any API call) or during the `call` (the in-flight request is abandoned — though the actual HTTP request continues unless you wire up an AbortController).

### Authentication flow

```js
function* authFlow() {
  while (true) {
    yield take('LOGIN_REQUESTED');
    try {
      const token = yield call(api.login);
      yield call(setAuthToken, token);
      yield put({ type: 'LOGIN_SUCCESS' });
      
      // Now logged in — wait for logout
      yield take('LOGOUT_REQUESTED');
      yield call(clearAuthToken);
      yield put({ type: 'LOGOUT_SUCCESS' });
    } catch (err) {
      yield put({ type: 'LOGIN_FAILURE', error: err.message });
    }
  }
}
```

This is impossible to express cleanly with thunks — the state machine of "waiting for login → logging in → logged in → waiting for logout" lives in a single readable function.

### Polling with cancellation

```js
function* pollWorker() {
  while (true) {
    try {
      const data = yield call(api.fetchUpdates);
      yield put({ type: 'UPDATE', data });
      yield delay(5000);
    } catch (err) {
      yield put({ type: 'POLL_ERROR', error: err.message });
      yield delay(30000);  // back off on error
    }
  }
}

function* watchPolling() {
  while (true) {
    yield take('START_POLLING');
    const task = yield fork(pollWorker);
    yield take('STOP_POLLING');
    yield cancel(task);
  }
}
```

The polling loop is cancellable mid-delay. When `STOP_POLLING` fires, `cancel(task)` interrupts the worker — even if it's mid-fetch or mid-delay.

### Retry with backoff

```js
function* fetchWithRetry(action) {
  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      const data = yield call(api.fetch, action.payload);
      yield put({ type: 'FETCH_SUCCESS', data });
      return;
    } catch (err) {
      yield delay(2 ** attempt * 1000);  // 1s, 2s, 4s, 8s, 16s
    }
  }
  yield put({ type: 'FETCH_FAILED' });
}
```

Linear, readable, no callback hell, no nested promises.

---

## Channels — The Low-Level Primitive

Internally, saga is built on **channels** — message queues that decouple producers from consumers. You usually don't touch them directly, but understanding them clarifies how the middleware works.

### Action channel

The default channel that emits every Redux action. `take` (without a custom channel argument) listens on this channel.

### `actionChannel(pattern, buffer)` — buffered action listener

By default, if `FETCH_USER` is dispatched twice in quick succession and your saga is processing the first one, the second is *missed*. The saga only resumes on the next `take`.

`actionChannel` buffers actions so you can process them serially without missing any:

```js
function* watchRequests() {
  const channel = yield actionChannel('PROCESS_QUEUE_ITEM');
  while (true) {
    const action = yield take(channel);
    yield call(processItem, action.payload);
    // serial — next action waits in the channel until we finish
  }
}
```

Useful for job queues, ordered processing, request throttling.

### `eventChannel(subscribe)` — bridge to external sources

For non-Redux event sources (WebSocket, server-sent events, browser APIs):

```js
function createWebSocketChannel(socket) {
  return eventChannel(emit => {
    socket.onmessage = e => emit(JSON.parse(e.data));
    socket.onclose = () => emit(END);
    return () => socket.close();  // cleanup
  });
}

function* watchSocket(socket) {
  const channel = yield call(createWebSocketChannel, socket);
  try {
    while (true) {
      const message = yield take(channel);
      yield put({ type: 'WS_MESSAGE', message });
    }
  } finally {
    if (yield cancelled()) channel.close();
  }
}
```

Channels turn callback-based APIs into saga-friendly streams.

---

## Testing Sagas

### Step-through testing (classic)

```js
import { call, put } from 'redux-saga/effects';

it('fetches user successfully', () => {
  const gen = fetchUserWorker({ payload: { id: 5 } });
  
  expect(gen.next().value).toEqual(call(api.fetchUser, 5));
  expect(gen.next({ name: 'Akshai' }).value).toEqual(
    put({ type: 'FETCH_USER_SUCCESS', user: { name: 'Akshai' } })
  );
  expect(gen.next().done).toBe(true);
});
```

You step through the generator, asserting each yielded effect. No mocking — the saga never actually calls `api.fetchUser`, because the test never executes the effect.

Pros: zero setup, no mocks, completely deterministic.

Cons: brittle — if you reorder yields or add a new one, every test downstream breaks. Couples tests to implementation order.

### Integration testing with `redux-saga-test-plan`

```js
import { expectSaga } from 'redux-saga-test-plan';

it('fetches user', () => {
  return expectSaga(fetchUserWorker, { payload: { id: 5 } })
    .provide([[call(api.fetchUser, 5), { name: 'Akshai' }]])
    .put({ type: 'FETCH_USER_SUCCESS', user: { name: 'Akshai' } })
    .run();
});
```

You provide stub values for specific calls, then assert that certain effects were dispatched. No step ordering, no brittleness — assert on outcomes.

Strongly preferred for non-trivial sagas.

### Testing forks and cancellation

`redux-saga-test-plan` handles these too — you can assert on `fork`, `cancel`, simulate actions arriving, etc. For most teams, this is the right testing layer.

---

## Saga vs Thunk vs RTK Query

| | **Thunk** | **Saga** | **RTK Query** |
|---|---|---|---|
| Mental model | Async function with dispatch | Coroutine watching action stream | Declarative cache |
| Boilerplate | Minimal | Moderate | Very low |
| Cancellation | Manual (AbortController) | Built-in | Built-in |
| Race conditions | Manual handling | `takeLatest`, `race` | Automatic |
| Polling | Manual | `delay` + loop | Built-in `pollingInterval` |
| Cross-action coordination | Hard | Easy | Limited |
| Testability | Mock-heavy | Trivial (step yields) | Mock at network layer |
| Bundle size | ~200 bytes | ~14 KB | ~9 KB |
| Learning curve | Low | High | Low–Medium |

### When to choose what

**RTK Query** — data fetching is most of your async work, you want caching, invalidation, polling, and optimistic updates without writing them yourself. This is the default choice for most new apps.

**Saga** — you have non-trivial action choreography: long-lived workflows, multi-step processes with cancellation, WebSocket integration, complex state machines, anything where actions trigger other actions in interesting ways. Worth the learning curve for the codebases that need it.

**Thunk** — small app, simple async, no complex coordination. Or, mix-and-match: use RTK Query for data, thunk for the rare bespoke action. You almost never want to add thunk to a saga codebase or vice versa.

**Don't** mix thunk and saga in a new codebase. Pick one async pattern and stay consistent. Mixing is a sign of evolutionary drift and makes the codebase harder to navigate.

---

## Common Pitfalls

### `put` doesn't wait

```js
yield put({ type: 'SHOW_LOADER' });
// loader is NOT necessarily visible yet — it's queued
const data = yield call(api.fetch);
```

`put` returns immediately. The dispatch happens, the reducer runs, but the component re-render is async. If your saga depends on the resulting state, use `select` *after* a yield boundary.

### Forgetting to `fork` watchers

```js
function* rootSaga() {
  yield watchA();  // blocks here forever — watchB never starts
  yield watchB();
}
```

Watchers are infinite loops. Yielding directly blocks the parent. Use `fork` (or `all([fork(watchA), fork(watchB)])`) to start them in parallel.

### `takeEvery` vs `takeLatest` confusion

`takeEvery` runs handlers concurrently. If your handler isn't idempotent (e.g., it increments a counter, or its order matters), you'll get bugs.

`takeLatest` cancels in-flight handlers — fine for "I only care about the latest result," but **wrong** if each action represents a distinct unit of work that must complete (e.g., individual save operations).

### Sharing state via module-level variables

```js
let cachedToken;

function* loginSaga() {
  cachedToken = yield call(api.login);  // wrong
}
```

Sagas don't have isolated state. Module-level variables are shared across all saga instances and across hot reloads. Use Redux state or pass values via action payloads.

### Infinite saga loops

```js
function* badSaga() {
  while (true) {
    yield put({ type: 'PING' });  // synchronous loop, no yield boundary
  }
}
```

Without a `take`, `call`, or `delay` to suspend, this is a tight infinite loop. The middleware will hang. Always include a yield that suspends.

### `select` returning new references

```js
const items = yield select(state => state.list.filter(x => x.active));
// `items` is a new array every time — fine for one-shot reads
// but if you yield this in a loop and check equality, you'll loop forever
```

Sagas don't have React's referential-equality re-render trap, but if you use `select` in loops with comparisons, the same caution applies. Memoize selectors or extract specific values.

### Forgetting to handle errors

A saga that throws an uncaught error stops *that saga*, but other sagas keep running. The user might see "nothing happened" with no indication of failure. Always wrap risky operations in `try/catch` and dispatch failure actions.

### Tests that don't reflect production

Step-through tests assert that you yield `call(api.fetchUser, 5)` — but they don't verify `api.fetchUser` does what you think. Pair step-through tests with at least one integration test (using `expectSaga` or a full Redux store) per saga.

---

## Further Reading

- [Redux source — applyMiddleware](https://github.com/reduxjs/redux/blob/master/src/applyMiddleware.ts) — ~50 lines, worth reading once
- [Redux-Saga docs](https://redux-saga.js.org/) — surprisingly thorough, especially the "Advanced" section
- [Yassine Elouafi's "Saga Pattern" essay](https://yelouafi.github.io/redux-saga/) — original motivation
- [Coroutines and CSP](https://www.youtube.com/watch?v=KBmM4HCfqAY) — Rob Pike on the underlying CS concepts (saga draws heavily from Go's goroutines and channels)
- [Channels in saga](https://redux-saga.js.org/docs/advanced/Channels) — deep dive on the lower-level primitives
- [redux-saga-test-plan](https://github.com/jfairbank/redux-saga-test-plan) — testing library, docs include good usage patterns
