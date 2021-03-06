# Setting Up The Middleware

Now that we know what [Epics](Epics.md) are, we need to provide them to the redux-observable middleware so they can start listening for actions.

## Root Epic

Similar to redux requiring a single root Reducer, redux-observable requires a single root Epic. As we [learned previously](Epics.md), we can use `combineEpics()` to accomplish this.

We recommend importing all of your Epics into a single file, which then exports the root Epic and the root Reducer.

### redux/modules/root.js

```js
import { combineEpics } from 'redux-observable';
import { combineReducers } from 'redux';
import ping, { pingEpic } from './ping';
import users, { fetchUserEpic } from './users';

export const rootEpic = combineEpics(
  pingEpic,
  fetchUserEpic
);

export const rootReducer = combineReducers({
  ping,
  users
});
```

> This pattern is an extension of the [Ducks Modular Redux pattern](https://github.com/erikras/ducks-modular-redux).

## Configuring The Store

Now create an instance of the redux-observable middleware.

```js
import { createEpicMiddleware } from 'redux-observable';

const epicMiddleware = createEpicMiddleware();
```

Then you pass this to the createStore function from Redux.

```js
import { createStore, applyMiddleware } from 'redux';

const store = createStore(
  rootReducer,
  applyMiddleware(epicMiddleware)
);
```

And after that you call `epicMiddleware.run()` with the rootEpic you created earlier.

```js
import { rootEpic } from './modules/root';

epicMiddleware.run(rootEpic);
```

Integrate the code above with your existing Store configuration so that it looks like this:

### redux/configureStore.js

```js
import { createStore, applyMiddleware } from 'redux';
import { createEpicMiddleware } from 'redux-observable';
import { rootEpic, rootReducer } from './modules/root';

const epicMiddleware = createEpicMiddleware();

export default function configureStore() {
  const store = createStore(
    rootReducer,
    applyMiddleware(epicMiddleware)
  );
  
  epicMiddleware.run(rootEpic);

  return store;
}
```

## Adding global error handler

Uncaught errors can bubble up to the root epic and cause the entire stream to terminate. As a consequence, epics registered in the middleware will no longer run in your application. To alleviate this issue, you can add a global error handler to the root epic that catches uncaught errors and resubscribes to the source stream.

```
const rootEpic = (action$, store$, dependencies) =>
  combineEpics(...epics)(action$, store$, dependencies).pipe(
    catchError((error, source) => {
      console.error(error);
      return source;
    })
  );
```

Within the body of the function based to the `catchError` operator, you can log the uncaught error to standard error or any other exception logging tool.

Note that in the example above, the `console.error` function is not supported in IE 8/9.

Also note that restarting the root epic can have some unintended consequences, especially if your application uses stateful epics, as they may lose state in the restart.

## Redux DevTools

To enable Redux DevTools Extension, just use `window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__` or [import `redux-devtools-extension` npm package](https://github.com/zalmoxisus/redux-devtools-extension#13-use-redux-devtools-extension-package-from-npm).

```js
import { compose } from 'redux'; // and your other imports from before
const epicMiddleware = createEpicMiddleware();

const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;

const store = createStore(pingReducer,
  composeEnhancers(
    applyMiddleware(epicMiddleware)
  )
);

epicMiddleware.run(pingEpic);
```
