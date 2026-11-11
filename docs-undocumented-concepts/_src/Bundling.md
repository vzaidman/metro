---
id: bundling
title: Bundling
---

_Bundling_ turns your app's source files into a _bundle_: the JavaScript your app loads at runtime. A bundle contains every [module](./Concepts.md#module) reachable from the [entry point](./Concepts.md#entry-point), [transformed](./Concepts.md#transformation) into code the app's JavaScript engine can run, and combined by Metro's [serializer](./Concepts.md#serialization).

## Building a bundle

You can get a bundle from Metro:

- **From the development server**, by requesting the entry point's path with `.bundle` instead of `.js`, e.g. `http://localhost:8081/index.bundle?platform=ios`. This is how React Native apps load their code in development. See [Bundle URLs](#bundle-urls).
- **As build output**, with `npx react-native bundle` for React Native release builds, [`metro build`](./CLI.md#build-entry), or [`runBuild`](./API.md#async-runbuildconfig-options).

To build a bundle, Metro:

1. [Resolves](./Concepts.md#resolution) and [transforms](./Concepts.md#transformation) the entry point and, recursively, every module it depends on, building the [dependency graph](./Concepts.md#dependency-graph).
2. Assigns each module a numeric ID, using [`serializer.createModuleIdFactory`](./Configuration.md#createmoduleidfactory).
3. [Serializes](./Concepts.md#serialization) the graph into a bundle, in this order:
   - Variable declarations Metro's runtime relies on, such as `__DEV__`.
   - [Scripts](#scripts): Metro's module system and polyfills.
   - Each [module](#modules), wrapped in a `__d` call.
   - Calls that run the modules from [`serializer.getModulesRunBeforeMainModule`](./Configuration.md#getmodulesrunbeforemainmodule), then the entry point.

## Bundle URLs

Any JS file in the [project root](./Concepts.md#project-root-and-watch-folders) can be the [entry point](./Concepts.md#entry-point) of a bundle served by the development server. To request a bundle, use the file's path relative to the project root (or [`server.unstable_serverRoot`](./Configuration.md#unstable_serverroot-experimental), if set), with `.js` changed to `.bundle`.

Pass build options as query parameters. All are optional. Boolean ones take `true` or `false` and default to `false` unless noted. `dev`, `platform` and `minify` correspond to the bundle's settings of the same name.

- `dev`: Build a [development bundle](./Concepts.md#development-and-production-builds). Defaults to `true`.
- `platform`: The [platform](./Concepts.md#platform) requesting the bundle (e.g. `ios`). Defaults to the platform suffix of the requested file name, if any (e.g. `ios` for `index.ios.bundle`).
- `minify`: Minify the code.
- `excludeSource`: Leave the source code out of the source map, e.g. `/index.map?platform=ios&dev=true&excludeSource=true` returns a source map without `sourcesContent`.
- `inlineSourceMap`: Append the source map to the bundle as a base64 `data:` URL.
- `lazy`: Leave modules loaded with [`import()`](./ModuleAPI.md#import-dynamic-import) out of the bundle and load them on demand.
- `modulesOnly`: Leave out Metro's runtime and polyfills, serving only the modules.
- `runModule`: End the bundle with the statements that run the entry point. Defaults to `true`.
- `shallow`: Include only the entry module, without its dependencies.
- `sourcePaths`: Reference source files in the source map as absolute file paths (`absolute`) or URL paths on the server (`url-server`). Defaults to `absolute`.
- `unstable_transformProfile`: Passed to the [Babel transformer](./Configuration.md#babeltransformerpath) to target a specific JavaScript engine (`default`, `hermes-stable` or `hermes-canary`). Defaults to `default`.
- `resolver.<key>` and `transform.<key>`: [Custom resolver options](./Resolution.md#customresolveroptions-string-mixed) and custom transform options, e.g. `transform.foo=bar` passes `{foo: 'bar'}` to the transformer.

For example, `http://localhost:8081/foo/bar/baz.bundle?dev=true&platform=ios` builds a development bundle for iOS from `foo/bar/baz.js`.

### Source maps

Metro builds a source map for each bundle. To request it, use the bundle's URL with `.map` instead of `.bundle` and the same query parameters, or the source map won't match the bundle. For example, the source map of `http://localhost:8081/index.bundle?platform=ios&dev=true` is `http://localhost:8081/index.map?platform=ios&dev=true`, which the bundle references with a `//# sourceMappingURL=` comment. This only works when `inlineSourceMap` is `false`. See [Source Map Format](./SourceMapFormat.md) for Metro's extensions to the format.

## Modules

Each module's code is wrapped in a function and registered under its numeric ID. For example, an `index.js` that requires `./Button` becomes (simplified, non-dev build):

<!-- prettier-ignore -->
```js
// index.js: module ID 0, depends on module 1
__d(function (global, require, _$$_IMPORT_DEFAULT, _$$_IMPORT_ALL, module, exports, _dependencyMap) {
  var Button = require(_dependencyMap[0]);
  Button.render();
}, 0, [1]);
// Button.js: module ID 1, no dependencies
__d(function (/* same parameters */) {
  module.exports = {render: function () {}};
}, 1);
__r(0);
```

`__d` registers a module under its ID, and `__r` runs a module by ID. Both are defined by Metro's runtime, included at the start of the bundle. In development builds, each `__d` call also gets the module's path as a last argument, e.g. `"index.js"`.

The string `_$$_METRO_` is reserved for identifiers that Metro generates for a serializer to find, such as `_$$_METRO_MODULE_ID`. To prevent accidental collisions, Metro's default transformer fails with an error on any source or JSON file containing the string, including in a comment or string literal. Custom transforms (e.g. Babel plugins) should not generate this string unless they expect Metro to process it.

Modules refer to each other by numeric IDs assigned at build time, not by path. That's why `require()` needs a path known at build time, and dynamic requires such as `require('./images/' + name)` aren't supported (see [`require()`](./ModuleAPI.md#require)).

### Scripts

Metro's runtime and the polyfills from [`serializer.getPolyfills`](./Configuration.md#getpolyfills) and [`serializer.polyfillModuleNames`](./Configuration.md#polyfillmodulenames) are transformed as _scripts_ instead of modules, and placed at the start of the bundle, before the modules. A script:

- Is wrapped in a function that runs as soon as the bundle loads, with the global object as its `global` argument, instead of in a `__d` call.
- Is not registered as a module, so it has no module ID and can't be required.
- Has no dependencies: its `require()` calls are left as they are, and nothing they refer to is added to the bundle.

For example, React Native's [`console.js`](https://github.com/facebook/react-native/blob/main/packages/polyfills/console.js) polyfill sends `console.log`, `console.warn` and `console.error` to the native log through `global.nativeLoggingHook`, so they work in every module, including the first one to run. In the bundle, it becomes (simplified):

```js
(function (global) {
  if (global.nativeLoggingHook) {
    global.console = {
      ...global.console,
      log: (...args) => global.nativeLoggingHook(args, /* level */ 1),
      warn: (...args) => global.nativeLoggingHook(args, /* level */ 2),
      error: (...args) => global.nativeLoggingHook(args, /* level */ 3),
    };
  }
})(globalThis);
```

## Bundle globals

Every bundle starts by defining these globals for your code. They are plain writable globals — later code, or another bundle in the same realm, can reassign them — but Metro sets them once at startup and does not expect them to change. Reassigning them can break the bundle: Metro's runtime reads them as it loads (for example, it captures `__METRO_GLOBAL_PREFIX__` to install `__r`/`__d`, and reads `__DEV__` throughout), so changed values leave code running against a mix of old and new ones.

| Global                    | Value                                                                           |
| ------------------------- | ------------------------------------------------------------------------------- |
| `__DEV__`                 | `true` in development bundles, `false` otherwise                                |
| `process.env.NODE_ENV`    | `'development'` or `'production'`, matching `__DEV__`                           |
| `__BUNDLE_START_TIME__`   | When the bundle started running, for startup timing                             |
| `__METRO_GLOBAL_PREFIX__` | The bundle's [`globalPrefix`](./Configuration.md#globalprefix), `''` by default |

In production builds, Metro's default transformer replaces `__DEV__` with `false` and `process.env.NODE_ENV` with `'production'`, then removes code these replacements make unreachable, so reassigning them at runtime has no effect there. In every build, it also replaces React Native's `Platform.OS` and `Platform.select()` with the bundle's [platform](./Concepts.md#platform). For example, in a production iOS bundle:

```js
if (__DEV__) {
  console.log('debug info'); // removed
}
const padding = Platform.select({ios: 12, android: 8}); // becomes `12`
```

See [Modules](#modules) for what else runs before your code.

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

## Assets

An _asset_ is a file, such as an image or a font, that your code requires like a JavaScript module:

```js
const logo = require('./logo.png');
```

Metro treats a file as an asset if its extension is listed in [`resolver.assetExts`](./Configuration.md#assetexts), which by default includes common image, video and audio formats.

### Variants

An asset can have variants for a specific [platform](./Concepts.md#platform) (dotted suffix, e.g. `.ios`) or, for images, screen density (`@` suffix, e.g. `@2x`, see [`resolver.assetResolutions`](./Configuration.md#assetresolutions)). For example, `logo@2x.ios.png` is the 2x iOS variant of `logo.png`.

`require('./logo.png')` refers to all of them: Metro picks the files for the bundle's platform, and includes every density, e.g. `logo.png`, `logo@2x.png` and `logo@3x.png`, so the app can load the best one for the screen.

### In the bundle

An asset is a [module](./Concepts.md#module) that passes a description of the asset (name, type, dimensions, available densities, and URL path on the server) to `registerAsset()` of the [`transformer.assetRegistryPath`](./Configuration.md#assetregistrypath) module, and exports the result. In React Native, `require` therefore returns a numeric ID that components such as `<Image>` accept.

To change the description, e.g. to add data computed from the file, use [`transformer.assetPlugins`](./Configuration.md#assetplugins).

### Serving and copying assets

The development server serves asset files, generally as-is, under `/assets/` (see [`transformer.publicPath`](./Configuration.md#publicpath)), e.g. `/assets/src/logo@2x.png?platform=ios` for `src/logo@2x.png`.

Release builds load assets from the app instead. [`runBuild`](./API.md#async-runbuildconfig-options) with `assets: true` returns the bundle's assets and their files, for build tools to copy into the app. React Native's `bundle` command does this with `--assets-dest`.

## Bundle formats

You only need the formats below if you work with Metro's output directly, e.g. in a tool or native loader that reads bundles. Metro can write two kinds of bundles:

- [Plain bundle](#plain-bundle): All modules in one JS file, as in the examples above.
- [RAM bundle](#ram-bundles): RAM (Random Access Modules) bundles store each module separately, so the app can load a module's code only when it's first required.

:::note

React Native doesn't use RAM bundles with Hermes, its default engine. See [RAM bundles](#ram-bundles).

:::

### Plain bundle

The standard format, for environments that expect a single JS file, such as React Native or a browser: the [scripts](#scripts) and [modules](#modules) described above, concatenated into one file. The development server serves it, and `npx react-native bundle` writes it for release builds.

For example, the `index.js` and `Button.js` [above](#modules) become this bundle (simplified, non-dev build):

<!-- prettier-ignore -->
```js
var __DEV__ = false, process = globalThis.process || {};
(function (global) {
  // Metro's module system, which defines __d and __r
})(globalThis);
(function (global) {
  // a polyfill
})(globalThis);
__d(function (global, require, _$$_IMPORT_DEFAULT, _$$_IMPORT_ALL, module, exports, _dependencyMap) {
  var Button = require(_dependencyMap[0]);
  Button.render();
}, 0, [1]);
__d(function (/* same parameters */) {
  module.exports = {render: function () {}};
}, 1);
__r(0);
```

### RAM bundles

A RAM bundle stores each module's code separately, either in its own file or at its own offset in a single file. A runtime that parses JavaScript source can then parse only the startup code at launch, and each module when it's first required, instead of the whole bundle. With [inline requires](./Configuration.md#gettransformoptions), a module is required, and so parsed, only when code first uses it, not when a file that imports it loads. This helps when parsing a large bundle slows startup. Metro only writes RAM bundles as build output; the development server doesn't serve them.

:::note

In React Native, RAM bundles were designed for JavaScriptCore, and aren't used with Hermes, React Native's default engine.

:::

Hermes runs bytecode compiled at build time, so there's no source to parse at startup, and it memory-maps the bytecode, loading only the parts that run. A plain bundle compiled to Hermes bytecode already gets the benefit a RAM bundle would give. React Native's legacy architecture is also the only one that can load RAM bundles, and React Native doesn't write them by default: its release builds run its `bundle` command, which writes a [plain bundle](#plain-bundle), unless configured to run a different command (`bundleCommand` in the Android Gradle plugin, `BUNDLE_COMMAND` for iOS's `react-native-xcode.sh`).

For example, as a [file RAM bundle](#file-ram-bundle), the `index.js` and `Button.js` [above](#modules) become these files (simplified, non-dev build):

<!-- prettier-ignore -->
```js
// index.android.bundle: the startup code, run when the bundle loads
var __DEV__ = false, process = globalThis.process || {};
(function (global) {
  // Metro's module system, which defines __d and __r
})(globalThis);
__r(0);

// js-modules/0.js: loaded when module 0 is first required
__d(function (global, require, _$$_IMPORT_DEFAULT, _$$_IMPORT_ALL, module, exports, _dependencyMap) {
  var Button = require(_dependencyMap[0]);
  Button.render();
}, 0, [1]);

// js-modules/1.js: loaded when module 1 is first required
__d(function (/* same parameters */) {
  module.exports = {render: function () {}};
}, 1);

// js-modules/UNBUNDLE: marks the bundle as a file RAM bundle.
// Contains only the magic number 0xFB0BD1E5, as 4 binary bytes.
```

When `__r` or `require` is called with a module that isn't registered yet, Metro's module system calls `global.nativeRequire` with its ID, and the app loads and runs that module's code, which registers it.

#### Writing a RAM bundle

Pass `require('metro/private/shared/output/RamBundle')` as the `output` option of [`runBuild`](./API.md#async-runbuildconfig-options). Metro picks the format based on the platform:

- [Indexed](#indexed-ram-bundle) for every platform except Android. On iOS, reading hundreds of separate files is slow, so all modules go in one file and are read by offset.
- [File](#file-ram-bundle) for Android. APKs are zip archives, where reading one small file is fast, but reading one module from a large file would require unzipping all of it.

To write the indexed format for Android, call the output's `save` with `indexedRamBundle: true` in its output options. `runBuild` doesn't set this option.

#### Indexed RAM bundle

A single binary file. All numbers are little-endian `uint32`s:

```text
magic number          0xFB0BD1E5
number of entries     highest module ID + 1
startup code length
offset, length        one pair per module ID
startup code          followed by \0
module code           followed by \0, one per module
```

Offsets are relative to the start of the startup code, and lengths include the null byte, so a module's code can also be read as a null-terminated string. The entry for module `x` is at byte `(3 + 2 * x) * 4`, and the startup code starts right after the table, at byte `(3 + 2 * numEntries) * 4`. The entries of modules grouped with [`ramGroups`](./Configuration.md#gettransformoptions) all point to the same code, which contains the whole group.

The startup code runs when the bundle loads: polyfills, Metro's module system, [`preloadedModules`](./Configuration.md#gettransformoptions), and the call that runs the entry point.

#### File RAM bundle

The startup code is written to the bundle output file, and each module to `js-modules/${id}.js` next to it. `js-modules/UNBUNDLE`, which contains only the magic number `0xFB0BD1E5`, marks the bundle as a file RAM bundle.
