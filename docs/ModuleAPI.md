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

By default, a `require()` inside a `try` block is optional: if Metro can't resolve it, the build succeeds and the `require()` throws at runtime instead. See [`allowOptionalDependencies`](./Configuration.md#allowoptionaldependencies).

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

## `module.exports`

Similar to [`module.exports`](https://nodejs.org/api/modules.html#moduleexports) in Node: holds the value `require()` returns for the current module after it finishes evaluating.

## `module.hot`

Available in [development builds](./Concepts.md#development-and-production-builds) only, for handling [Fast Refresh](./FastRefresh.md) updates manually. See [`module.hot`](./FastRefresh.md#modulehot).

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

## `require.context()`

Requires every file in a directory whose path matches a pattern, as in [webpack](https://webpack.js.org/guides/dependency-management/#requirecontext). Enable it with [`transformer.unstable_allowRequireContext`](./Configuration.md#unstable_allowrequirecontext-experimental).

```js
const context = require.context(directory, recursive, filter, mode);
```

The arguments must be compile-time constants:

- `directory`: The directory to search, relative to the current file.
- `recursive`: Whether to include subdirectories. Defaults to `true`.
- `filter`: A regular expression literal tested against each file's path relative to `directory`, starting with `./`, e.g. `./screens/Home.js`. Defaults to `/.*/`.
- `mode`: How matching modules are loaded. Defaults to `'sync'`.
  - `'sync'`: `context(key)` returns the module's exports.
  - `'eager'`: `context(key)` returns a promise of the module's exports. Every module is included in the bundle.
  - `'lazy'`, `'lazy-once'`: `context(key)` returns a promise of the module's exports, loaded like [`import()`](#import-dynamic-import).

`context.keys()` returns the matching paths, which are the keys `context` accepts. In development, adding or deleting a matching file updates the context. For example, to require every screen in `./screens`:

```js
const screens = require.context('./screens', false, /\.js$/);
for (const key of screens.keys()) {
  register(key, screens(key).default); // e.g. key = './Home.js'
}
```

## `require.resolveWeak()`

Takes a module name (or path) and returns that module's opaque ID, without including it in the bundle. A specialised API for frameworks; application code rarely needs it. See [using `require` at runtime](#advanced-usage-require-at-runtime).

## Require cycles

A _require cycle_ is a chain of imports that loops back to a module that is still loading, such as `A` imports `B` and `B` imports `A`. Metro allows cycles. The module that closes the cycle receives the first module's exports as they are at that moment, which are often not set yet:

```js
// a.js
const b = require('./b');
exports.name = 'a';
exports.b = b;

// b.js
const a = require('./a'); // a is still loading: a.name is not set yet
module.exports = {getA: () => a};
```

Values read while the cycle is loading can therefore be `undefined`. Reading them later, inside a function such as `getA` above, sees the finished values — unless the module replaces `module.exports` wholesale, in which case the importer keeps the original empty object. In development, Metro warns when it detects a cycle:

```text
Require cycle: a.js -> b.js -> a.js

Require cycles are allowed, but can result in uninitialized values. Consider refactoring to remove the need for a cycle.
```

Prefer removing the cycle, for example by moving the shared code into a third module both files import. For cycles you cannot remove, such as in third-party code, silence the warning with [`resolver.requireCycleIgnorePatterns`](./Configuration.md#requirecycleignorepatterns). Silencing the warning does not change the loading behaviour.

## Inline requires

With _inline requires_, Metro moves a top-level `require` to each place its result is used, so the required module loads at first use instead of when the importing file loads:

```js
// You write:
const Foo = require('./Foo');
function render() {
  return Foo.render();
}

// The bundle contains the equivalent of:
function render() {
  return require('./Foo').render();
}
```

This defers module initialization, and its side effects, until they are needed, which can make startup faster. It also means a module's top-level code may run later than its position in the import graph suggests, so do not rely on import order alone for side effects.

Inline requires are controlled per bundle by [`transformer.getTransformOptions`](./Configuration.md#gettransformoptions) (`transform.inlineRequires`). Metro leaves them off by default; React Native turns them on.

## Bundle globals

Every bundle starts by defining these globals for your code. They are plain writable globals — later code, or another bundle in the same realm, can reassign them — but Metro sets them once at startup and does not expect them to change. Reassigning them can break the bundle: Metro's runtime reads them as it loads (for example, it captures `__METRO_GLOBAL_PREFIX__` to install `__r`/`__d`, and reads `__DEV__` throughout), so changed values leave code running against a mix of old and new ones.

| Global                    | Value                                                                           |
| ------------------------- | ------------------------------------------------------------------------------- |
| `__DEV__`                 | `true` in development bundles, `false` otherwise                                |
| `process.env.NODE_ENV`    | `'development'` or `'production'`, matching `__DEV__`                           |
| `__BUNDLE_START_TIME__`   | When the bundle started running, for startup timing                             |
| `__METRO_GLOBAL_PREFIX__` | The bundle's [`globalPrefix`](./Configuration.md#globalprefix), `''` by default |

In release builds, reads of `__DEV__` and `process.env.NODE_ENV` are replaced with their values during [transformation](./Concepts.md#transformation), so reassigning them at runtime has no effect there and checks like `if (__DEV__) { ... }` can be dropped when minified. See [Bundling](./Bundling.md#modules) for what else runs before your code.
