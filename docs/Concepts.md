---
id: concepts
title: Concepts
---

Metro is React Native's default JavaScript bundler. It takes an entry file and various options, and returns a single JavaScript file containing all your code and its dependencies: a _bundle_.

Starting a React Native app in development (e.g. `npx react-native start`) starts a Metro server. Your app downloads its bundle from it, and Metro keeps the bundle updated as you edit files, powering [Fast Refresh](./FastRefresh.md).

This page defines the terms used throughout the Metro docs.

## Bundling stages

Metro's bundling process has three separate stages: resolution, transformation and serialization.

### Resolution

A _resolver_ finds which file each `import` or `require` refers to, so that Metro can build the [dependency graph](#dependency-graph). For example, the resolver decides that `import Button from './Button'` refers to `./Button.ios.js` in an iOS bundle. See [Module Resolution](./Resolution.md) for the full algorithm. To customize resolution, set [`resolver.resolveRequest`](./Configuration.md#resolverequest) to your own resolver.

### Transformation

A _transformer_ converts (transpiles) every module to a format the target platform (e.g. React Native) understands. For example, the default transformer uses [Babel](https://babeljs.io/) to compile JSX, TypeScript and Flow syntax, and the results are [cached](./Caching.md). To customize how each file's code is compiled, set [`transformer.babelTransformerPath`](./Configuration.md#babeltransformerpath); to replace the whole transformer, set [`transformerPath`](./Configuration.md#transformerpath).

### Serialization

Once all modules are transformed, a _serializer_ combines them, along with Metro's module runtime, into one or more bundles: JavaScript files your app can load and run. See [Bundling](./Bundling.md). To customize serialization, set [`serializer.customSerializer`](./Configuration.md#customserializer).

## Terminology

### Module

A single file in a bundle: a JavaScript (or TypeScript, JSON, etc.) source file, or an asset such as an image. Each module is assigned a numeric ID in the bundle ([example](./Bundling.md#modules)).

### Entry point

The module a bundle is built from. Metro follows its dependencies recursively to find every module the bundle needs. In a React Native app, this is typically `index.js` in the project root.

### Asset

A file, such as an image or a font, that's required like a JavaScript module, e.g. `require('./logo.png')`. Metro treats files whose extensions are in [`resolver.assetExts`](./Configuration.md#assetexts) as assets, serves them from the development server, and lists them for build tools to copy into the app, e.g. with [`runBuild`](./API.md#async-runbuildconfig-options)'s `assets` option. See [Assets](./Assets.md).

### Dependency graph

All modules reachable from an entry point, and the dependencies between them. During development, Metro keeps this graph in memory and updates it incrementally when [files change](#file-watching), instead of rebuilding the whole bundle.

The first request for a bundle builds its graph. Later requests only rebuild what was added, changed or deleted since — an _incremental rebuild_ — which is why they are fast. Each distinct bundle (different entry point, platform or `dev` setting, for example) has its own graph, and restarting Metro discards them all. See [Development server URLs](./API.md#development-server-urls).

### Platform

The platform a bundle is built for, e.g. `ios` or `android`. Metro uses it to pick [platform-specific files](https://reactnative.dev/docs/platform-specific-code) such as `Button.ios.js` or `Button.android.js`. See [`resolver.platforms`](./Configuration.md#platforms).

### Development and production builds

Every bundle is built either for development or production, set by the `dev` option, e.g. `dev=true` in a [bundle URL](./Bundling.md#bundle-urls) or `--dev` for [`metro build`](./CLI.md#build-entry). Development builds support [Fast Refresh](#fast-refresh), keep module paths in the bundle for debugging, and warn about require cycles at runtime (see [`resolver.requireCycleIgnorePatterns`](./Configuration.md#requirecycleignorepatterns)).

Metro's default transformer replaces React Native's `Platform.OS` and `Platform.select()` with the bundle's platform. In production builds, it also replaces `__DEV__` with `false` and `process.env.NODE_ENV` with `'production'`, then removes code these replacements make unreachable. For example, in a production iOS bundle:

```js
if (__DEV__) {
  console.log('debug info'); // removed
}
const padding = Platform.select({ios: 12, android: 8}); // becomes `12`
```

Production builds are usually also minified, with [`transformer.minifierPath`](./Configuration.md#minifierpath).

### Project root and watch folders

Metro only knows about files inside the [`projectRoot`](./Configuration.md#projectroot) and [`watchFolders`](./Configuration.md#watchfolders) directories: it crawls them on startup, keeps an in-memory map of their files, and [watches them for changes](#file-watching). `projectRoot` is your app's directory; `watchFolders` lists any other directories it depends on. That matters most in a monorepo, where an app in `apps/mobile` imports a shared package in `packages/ui`: unless `packages/ui` is in `watchFolders`, Metro can't bundle it, even though the files exist on disk. See [Monorepos](./Monorepos.md) for the setup.

### File watching

With file watching enabled, Metro listens for files being added, changed or deleted in the [project root and watch folders](#project-root-and-watch-folders), and updates its file map and [dependency graph](#dependency-graph); this is how the development server picks up edits for Fast Refresh. Otherwise, Metro only sees files as they were at startup.

Watching is on unless the [JavaScript API](./API.md)'s `watch` option or the `CI` environment variable turns it off, and [`runBuild`](./API.md#async-runbuildconfig-options) never watches. See [`resolver.useWatchman`](./Configuration.md#usewatchman) and [Watcher Options](./Configuration.md#watcher-options).

### Fast Refresh

Applying an edit to a running development app without restarting it, keeping its state when the change allows. Metro sends the changed modules to the app, which applies them in place or falls back to a full reload. See [Fast Refresh](./FastRefresh.md).

### Inline requires

Moving a top-level `require` to the places its result is used, so the required module initializes at first use instead of at import time. Controlled per bundle; React Native enables it by default. See [Inline requires](./ModuleAPI.md#inline-requires).

### Hermes

React Native's default JavaScript engine. Metro bundles JavaScript either way; release builds for Hermes then compile the plain bundle to bytecode, which is why [RAM bundles](./Bundling.md#ram-bundles) are not used with Hermes. A framework can ask the transformer to target Hermes with the `unstable_transformProfile` bundle option (see [Bundle URLs](./Bundling.md#bundle-urls)).

### Haste

An opt-in module system in which modules are imported by a globally unique name instead of a path, e.g. `import Foo from 'Foo'` instead of `import Foo from '../components/Foo'`, wherever `Foo.js` lives in the project. Most React Native apps don't use Haste. See [`resolver.hasteImplModulePath`](./Configuration.md#hasteimplmodulepath).
