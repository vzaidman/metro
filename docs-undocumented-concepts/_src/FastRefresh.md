---
id: fast-refresh
title: Fast Refresh
---

In development, when you save a file, Metro sends the change to your running app, which applies it without restarting. React Native calls this [Fast Refresh](https://reactnative.dev/docs/fast-refresh). When it can, the app keeps its state, e.g. the current screen and what you typed. Otherwise, it reloads. Fast Refresh only exists in [development builds](./Concepts.md#development-and-production-builds).

This page describes Metro's part. For what React treats as a component and how it keeps their state, see React Native's [Fast Refresh docs](https://reactnative.dev/docs/fast-refresh).

## What happens when you edit a file

1. Metro's [file watcher](./Concepts.md#file-watching) notices the change.
2. For each bundle a running app has loaded, Metro updates the bundle's [dependency graph](./Concepts.md#dependency-graph) and [transforms](./Concepts.md#transformation) the changed files.
3. Metro sends the changed modules to the app over a WebSocket.
4. The app runs the new code, either in place, keeping its state, or by reloading.

If the change can't be bundled, e.g. because of a syntax error or a missing import, Metro sends the error instead. The app shows it and keeps running its previous code until you fix it. See [Reading bundling errors](./Troubleshooting.md#reading-bundling-errors).

## When state is kept and when the app reloads

When a file changes, the app re-runs it. The files importing it still hold its old exports, so they re-run too, then the files importing them, and so on, until each import path reaches a _boundary_: a file that takes the update without the files importing it re-running. If an import path reaches the [entry point](./Concepts.md#entry-point) without passing a boundary, the app reloads and loses its state.

A file is a boundary if either:

- **It exports only React components.** This is a _React Refresh boundary_: React re-renders the components with the new code and keeps their state. A single other export, such as a constant, a helper or a hook, means it isn't one.
- **It calls [`module.hot.accept()`](#modulehot).** This is a _manual boundary_.

For example, with `App.js` importing `Header.js` and `Button.js`, which both import `theme.js`:

- Editing `Header.js`, which exports only a component, re-runs just `Header.js`.
- Editing `theme.js`, which exports a color object, re-runs `theme.js`, `Header.js` and `Button.js`. `App.js` doesn't re-run.

Metro checks a file's exports each time an update arrives, so there's nothing to declare. Unlike some bundlers, e.g. webpack's [React Refresh plugin](https://github.com/pmmmwh/react-refresh-webpack-plugin), Metro doesn't add `module.hot.accept()` calls to React Refresh boundaries. If an edit changes a boundary's exports, e.g. adds a non-component export or renames a component, the update continues to the files importing it, which must all be React Refresh boundaries, or the app reloads.

The app also reloads when:

- The update runs into a [require cycle](./Bundling.md#require-cycles).
- The changed file matches [`resolver.unstable_forceFullRefreshPatterns`](./Configuration.md#unstable_forcefullrefreshpatterns-experimental).

If re-running a file throws, the update stops, and the file keeps its previous exports until you fix the error.

## `module.hot`

In development builds, `module.hot` lets a file handle updates itself. It's `undefined` in production builds, so check for it:

- `module.hot.accept(callback?)`: Makes the file a manual boundary. `callback` runs after the file re-runs.
- `module.hot.dispose(callback?)`: `callback` runs before the file re-runs, to undo its side effects.

```js
const interval = setInterval(poll, 1000);

if (module.hot) {
  module.hot.dispose(() => clearInterval(interval));
  module.hot.accept();
}
```

With React Refresh, app and library developers rarely need this: only for a file that isn't a component but can safely re-run on its own, like the one above. Without React Refresh, it's the only way to avoid reloading the app, e.g. in the root file, which renders the app again when it re-runs.

## Using Fast Refresh outside React Native

React Native sets up Fast Refresh for you. Other apps and frameworks bundled with Metro set it up in their development builds:

1. **Serve updates.** [`runServer`](./API.md#async-runserverconfig-options) serves them on the `/hot` WebSocket. With [`createConnectMiddleware`](./API.md#createconnectmiddlewareconfig-options), call `attachHmrServer` on your HTTP server.
2. **Receive updates.** In the app, create an `HMRClient` from `metro-runtime/modules/HMRClient`, register the loaded bundles, and call `enable()`. This is enough for manual boundaries:

   ```js
   const HMRClient = require('metro-runtime/modules/HMRClient');

   const client = new HMRClient('ws://localhost:8081/hot');
   client.send(
     JSON.stringify({
       type: 'register-entrypoints',
       entryPoints: [
         'http://localhost:8081/index.bundle?platform=web&dev=true',
       ],
     }),
   );
   client.on('error', error => console.error(error.message));
   client.enable();
   ```

3. **Set up React Refresh**, for React Refresh boundaries. Apply the `react-refresh/babel` plugin, e.g. through the [Babel transformer](./Configuration.md#babeltransformerpath), and set `global.__ReactRefresh` (prefixed with [`transformer.globalPrefix`](./Configuration.md#globalprefix)) to the React Refresh runtime (`react-refresh/runtime`), with an added `performFullRefresh(reason)` method that reloads the app. In a browser, Metro reloads the page itself.
