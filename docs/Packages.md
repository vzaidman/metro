---
id: packages
title: Packages
---

Metro is published as a set of npm packages. Most run in Node.js while bundling; one runs inside your app. This page says which package does what, so you know what to install, what you can replace, and what not to depend on. If you only use React Native's defaults, you can skip it: React Native installs and wires these for you.

## Start here

| Package             | What it is                                                                                                                                                                                                                                   |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `metro`             | The bundler, development server, [CLI](./CLI.md) and [Bundling API](./API.md). This is what you run                                                                                                                                          |
| `metro-config`      | Finds, loads and merges `metro.config.js`. Owns `getDefaultConfig`, `loadConfig`, `mergeConfig` and [`resolveConfig`](./API.md#resolveconfigfilepath-cwd)                                                                                    |
| `metro-runtime`     | The only package whose code runs in your app: the module system behind `require`, dynamic `import()` loading, and the [Fast Refresh](./FastRefresh.md) client. It has no main entry point; Metro includes its modules in your bundle for you |
| `metro-symbolicate` | Standalone command that symbolicates release stack traces and profiles. See [Symbolicating Stack Traces](./Symbolication.md)                                                                                                                 |

Everything except `metro-runtime` runs at build time in Node.js. Errors from build-time packages appear in Metro's terminal output; `metro-runtime` behaviour, such as [require cycles](./ModuleAPI.md#require-cycles), appears in your app.

## Packages behind config options

You meet these through [configuration](./Configuration.md) rather than by importing them. Each customization point names a package or a module with the same shape:

| You configure                                                                 | Default package           | Purpose                                                                                                  |
| ----------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------- |
| [`resolver.resolveRequest`](./Configuration.md#resolverequest)                | `metro-resolver`          | Turns each import into a file. See [Module Resolution](./Resolution.md)                                  |
| [`transformer.babelTransformerPath`](./Configuration.md#babeltransformerpath) | `metro-babel-transformer` | Compiles each file with Babel. React Native replaces it with its own                                     |
| [`transformerPath`](./Configuration.md#transformerpath)                       | `metro-transform-worker`  | Runs the per-file pipeline (Babel, dependency collection, minification, source maps) in parallel workers |
| [`transformer.minifierPath`](./Configuration.md#minifierpath)                 | `metro-minify-terser`     | Minifies with Terser                                                                                     |
| [`cacheStores`](./Configuration.md#cachestores)                               | `metro-cache`             | Layered transform cache, including shared HTTP caches. See [Caching](./Caching.md)                       |
| [`serializer.customSerializer`](./Configuration.md#customserializer)          | Built into `metro`        | Combines modules into the bundle. See [Bundling](./Bundling.md)                                          |

Four more packages matter when you write custom tooling or debug a build:

- `metro-transform-plugins`: Metro's own Babel passes, applied on top of your Babel config. They implement import/export conversion, [inline requires](./ModuleAPI.md#inline-requires), `__DEV__` and platform inlining, and constant folding. That is why bundled code can differ from running your Babel config alone.
- `metro-cache-key`: Computes a cache key from file contents. If you write a custom transformer, its `getCacheKey` must cover every file that affects its output, or Metro reuses stale results (see [Stale bundle](./Troubleshooting.md#stale-bundle)).
- `metro-source-map`: Builds and reads Metro's source maps. Tool authors should see [Source Map Format](./SourceMapFormat.md); app developers can skip it.
- `metro-file-map`: Crawls and watches `projectRoot` and `watchFolders`, which is why files outside them are invisible. Its API is experimental and changes are not treated as breaking; configure it through Metro options instead of depending on it directly.

`metro` also exports helpers for tooling authors building on the [Bundling API](./API.md): `Terminal` and the two built-in reporters behind the [`reporter`](./Configuration.md#reporter) option, `TerminalReporter` and `JsonReporter`; and `getSchemeResolvers()`, which returns Metro's built-in `metro:` scheme resolver to spread into a custom resolution context.

## Versioning and stability

All Metro packages are released together with the same version number, and depend on each other by exact version. Install matching versions; mixing versions, for example a newer `metro` with an older `metro-runtime`, is not supported.

Stability follows the entry point:

- **Package main entries** (such as `require('metro')`) and config options without a prefix are the supported surface.
- **`unstable_*` options** are experimental and can change or disappear. They are marked Experimental in [Configuring Metro](./Configuration.md).
- **`metro/private/*` imports** have no stability guarantee. The docs name one deliberate exception, the RAM bundle output (see [Writing a RAM bundle](./Bundling.md#writing-a-ram-bundle)); treat any other private import as breakable at any release.

The remaining packages (`metro-core`, `metro-babel-register`, `buck-worker-tool`, `ob1`) are internal to Metro's own build and development. In particular, `metro-babel-register` transpiles Metro's own source when running from a checkout; it does not transform your app code — that is `metro-babel-transformer`. Do not depend on these packages directly.
