---
id: fast-refresh
title: Fast Refresh
---

During development, Metro sends code changes to running apps, and its runtime applies them without reloading the app. React Native builds [Fast Refresh](https://reactnative.dev/docs/fast-refresh) on this, using [React Refresh](https://github.com/facebook/react/tree/main/packages/react-refresh) to re-render the updated components while keeping their state.

This page describes Metro's part. Updates only apply to development bundles (`dev=true`).

## Detecting changes

Metro's [file watcher](./Concepts.md#file-watching) reports files being added, changed or deleted in the [project root or watch folders](./Concepts.md#project-root-and-watch-folders).

## Updating the dependency graph

For each bundle that a running app has [registered](#delivering-updates), Metro updates the bundle's [dependency graph](./Concepts.md#dependency-graph) and [transforms](./Concepts.md#transformation) the changed modules.

## Delivering updates

Apps register with Metro by connecting to the server's `/hot` WebSocket and sending the full URL of each bundle they loaded from it, e.g. `http://localhost:8081/index.bundle?platform=ios&dev=true`. Metro first sends the changes made since the bundle was built, then an update after each change: the code of added and modified modules, and the IDs of deleted modules.

If a change can't be bundled, e.g. because of a syntax error or a missing import, Metro sends the error instead, and the app keeps running its previous code. React Native displays the error.

## Applying updates

If a changed module hasn't run yet, Metro's runtime only replaces its code, which runs when the module is first required.

If it has already run, re-running it alone isn't enough: the modules importing it still hold its old exports. So the runtime also re-runs those modules, then the modules importing them, and so on up the import chain, until each chain reaches a _boundary_. Modules re-run in dependency order, then React Refresh re-renders the updated components.

### Boundaries

A boundary is a module that accepts an update: the update stops there, and the modules importing it don't re-run. A module accepts an update in one of two ways:

- **Automatically**, as a _React Refresh boundary_: with [React Refresh set up](#setting-up-fast-refresh), a module whose exports are all React components, none of them a getter. The modules importing it don't need to re-run, because they use its components through React, and React Refresh re-renders them with the new code. See React Native's [Fast Refresh docs](https://reactnative.dev/docs/fast-refresh) for what counts as a component.
- **Manually**, as a _manual boundary_: a module that calls [`module.hot.accept()`](#modulehot) when it runs. With React Refresh, app and library developers rarely need this, only for a module that isn't a component but can safely re-run on its own, e.g. one that starts a timer and stops it on [`module.hot.dispose()`](#modulehot). Without React Refresh, it's the only way to avoid reloading the app, e.g. in the root module.

For example, with `App.js` importing `Header.js` and `Button.js`, which both import `theme.js`:

- `Header.js` exports only a component, so it's a React Refresh boundary. Editing it re-runs just `Header.js`.
- `theme.js` exports a color object, so it isn't a boundary. Editing it re-runs `theme.js`, then `Header.js` and `Button.js`. `App.js` doesn't re-run.

:::note

Unlike some bundlers' React Refresh integrations (e.g. webpack's [React Refresh plugin](https://github.com/pmmmwh/react-refresh-webpack-plugin)), Metro doesn't add `module.hot.accept()` calls to React Refresh boundaries. Metro's runtime checks a module's exports when an update arrives, so the two kinds of boundaries behave differently:

- After a React Refresh boundary re-runs, the runtime checks its new exports. If it no longer qualifies, or the names or kinds of its exports changed, the update continues to the modules importing it, which must all be React Refresh boundaries, or the app reloads. A manual boundary always stops the update.
- Only manual boundaries run an `accept()` callback.

:::

### Full reloads

The app reloads instead, losing its state, when:

- The update reaches the [entry point](./Concepts.md#entry-point) without passing a boundary.
- The update runs into an import cycle.
- A re-run React Refresh boundary no longer qualifies, and the modules importing it aren't all React Refresh boundaries.
- The changed module matches [`resolver.unstable_forceFullRefreshPatterns`](./Configuration.md#unstable_forcefullrefreshpatterns-experimental).

If a re-run module throws, the module keeps its previous exports and the update stops, so the app keeps working until the error is fixed and another update arrives.

## Setting up Fast Refresh

React Native sets up Fast Refresh; see its [Fast Refresh docs](https://reactnative.dev/docs/fast-refresh). Other apps and frameworks bundled with Metro set it up in their development builds:

1. **Connect to Metro.** Create an `HMRClient` from `metro-runtime/modules/HMRClient` with the server's `/hot` WebSocket URL, register the loaded bundles, and call `enable()` to apply updates as they arrive. Its `error` event reports changes Metro couldn't bundle, and its `update-start` and `update-done` events report progress:

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

   This is enough for manual boundaries.

2. **Set up React Refresh**, for React Refresh boundaries:
   - Apply the `react-refresh/babel` plugin, e.g. through the [Babel transformer](./Configuration.md#babeltransformerpath), so React Refresh can track components.
   - Set `global.__ReactRefresh` (prefixed with [`transformer.globalPrefix`](./Configuration.md#globalprefix)) to the React Refresh runtime (`react-refresh/runtime`), with an added `performFullRefresh(reason)` method that reloads the app. In a browser, Metro reloads the page itself.

## `module.hot`

Available in development builds, for modules that handle updates themselves:

- `module.hot.accept(callback?)`: Makes the current module a [manual boundary](#boundaries): when it or a module it imports changes, the update stops at this module. `callback` runs after the module re-runs.
- `module.hot.dispose(callback?)`: `callback` runs before the current module re-runs, e.g. to undo side effects it had.

Use them in a module that can safely re-run on its own:

- With React Refresh, a module that isn't a component and has side effects it can undo and redo, such as starting a timer or registering handlers, routes or translations: `dispose` undoes them, and re-running the module redoes them with the new code. In a library, this only takes effect while the library's own files are edited, e.g. from its example app.
- Without React Refresh, the root module, which renders the app again when it re-runs.

For example, a module that polls a server:

```js
const interval = setInterval(poll, 1000);

if (module.hot) {
  module.hot.dispose(() => clearInterval(interval));
  module.hot.accept();
}
```
