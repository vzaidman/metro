---
id: module-api
title: Module API
---

This page describes the module APIs available to your app's code when Metro bundles it. Metro is designed to allow code written for Node (or for bundlers targeting the Web) to run mostly unmodified.

## `require()`

Similar to Node's [`require()`](https://nodejs.org/api/modules.html#requireid) function, it takes a module name (or path) and returns the result of evaluating that module's code. Metro adds every module referenced by `require()` to the bundle.

```js
const localModule = require('./path/module');
const asset = require('./path/asset.png');
const jsonData = require('./path/data.json');
const {View} = require('react-native');
```

The argument to `require()` must be a compile-time constant, such as a string literal, so that Metro knows at build time which modules to include. A non-constant argument, e.g. `require('./images/' + name + '.png')`, fails at build time, except in files inside `node_modules`, where the [`dynamicDepsInPackages`](./Configuration.md#dynamicdepsinpackages) config option controls whether it fails at build time or at runtime.

### Advanced usage: `require` at runtime

At build time, Metro [resolves](./Concepts.md#resolution) module names to absolute paths and [assigns an opaque module ID](./Configuration.md#createmoduleidfactory) to each one.

At runtime, `require` refers to a function that takes an opaque module ID (_not_ a name or path) and returns a module. This is useful if you already have a module ID from another module API, such as [`require.resolveWeak`](#requireresolveweak).

```js
const localModule = require('./path/module');
const id = require.resolveWeak('./path/module');
// Bypass the restriction on non-constant require() arguments
const dynamicRequire = require;
dynamicRequire(id) === localModule; // true
```

## `require.context()`

Imports every file in a directory that matches a pattern, for example all route, story or translation files, without listing them one by one. It returns a function: call it with a key to get that file's module, and call its `keys()` method to list the keys.

```js
const routes = require.context('./routes', true, /\.js$/);

for (const key of routes.keys()) {
  // key is e.g. './settings/profile.js'
  registerRoute(key, routes(key));
}
```

The arguments are `require.context(directory, recursive, filter, mode)`:

- `directory` (string): **Required.** The directory to import from, relative to the current file.
- `recursive` (boolean): Include files in subdirectories. Defaults to `true`.
- `filter` (RegExp): Only include files whose `./relative/path` matches. Defaults to all files.
- `mode` (string): How each module is loaded. Defaults to `'sync'`:

  | Mode          | `context(key)` returns           |
  | ------------- | -------------------------------- |
  | `'sync'`      | The module                       |
  | `'eager'`     | A `Promise` for the module       |
  | `'lazy'`      | A `Promise` (dynamic `import()`) |
  | `'lazy-once'` | Same as `'lazy'` in Metro        |

Like `require()`, every argument must be a compile-time constant, so Metro knows at build time which files to include. Adding or removing a matching file updates the context on the next build. If no file matches, `keys()` is empty and calling the context throws.

`require.context()` is experimental and off by default. Enable it with [`transformer.unstable_allowRequireContext`](./Configuration.md#unstable_allowrequirecontext-experimental); without it, calling `require.context()` throws at runtime.

## Optional dependencies

A `require()` inside a `try` block is optional by default: if Metro can't resolve it, the build still succeeds, and the `require()` throws at runtime instead, for the `catch` block to handle. For example, to use a package only if it's installed:

```js
let Haptics = null;
try {
  Haptics = require('react-native-haptic-feedback');
} catch {}
```

To make these failures fail the build instead, set [`transformer.allowOptionalDependencies`](./Configuration.md#allowoptionaldependencies).

## `module.exports`

Similar to [`module.exports`](https://nodejs.org/api/modules.html#moduleexports) in Node: holds the value `require()` returns for the current module after it finishes evaluating.

## `module.hot`

See [`module.hot`](./FastRefresh.md#modulehot) in Fast Refresh.

## ES Modules syntax (`import` and `export`)

To support `import` and `export` in Metro projects, we recommend using [`@babel/plugin-transform-modules-commonjs`](https://babeljs.io/docs/babel-plugin-transform-modules-commonjs).

:::note

In React Native projects that use `@react-native/babel-preset`, `import` and `export` are supported out of the box.

:::

## `import()` (dynamic import)

[`import()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) calls are supported out of the box. In React Native, using `import()` automatically splits your application code so that it loads faster during development, without affecting release builds.

In development with lazy bundling, each `import()` fetches its module from the Metro server the first time it runs, so you will see extra bundle requests in network logs. Release builds include the module in the single bundle by default, and the same `import()` resolves without a network request. If `import()` fails at runtime with an error about `__loadBundleAsync`, the framework loading the bundle has enabled lazy bundling without providing that function; see below.

:::info

**For framework implementers**:

1. Enable [lazy bundling](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0605-lazy-bundling.md) by adding `&lazy=true` to the initial HTTP bundle URL your framework requests from Metro.
2. At runtime, `import()` calls a framework-defined function to [fetch and evaluate](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0605-lazy-bundling.md#__loadbundleasync-in-metro) the split bundle. Your framework **must** implement this function if it uses the `lazy=true` parameter, or runtime errors will occur.

   For example, if the initial bundle URL is `/index.bundle?platform=ios&dev=true&lazy=true`, then `import('./src/Settings')`, resolving to `src/Settings.js`, calls `global.__loadBundleAsync('/src/Settings.js.bundle?platform=ios&dev=true&lazy=true&modulesOnly=true&runModule=false')` before evaluating the module.

:::

## `require.resolveWeak()`

Takes a module name (or path) and returns that module's opaque ID, without including it in the bundle. A specialised API for frameworks; application code rarely needs it. See [using `require` at runtime](#advanced-usage-require-at-runtime).
