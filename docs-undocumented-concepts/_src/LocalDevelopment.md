---
id: local-development
title: Local Development Setup
---

This page is for contributors to Metro itself. See also the [contributing guide](https://github.com/react/metro/blob/main/CONTRIBUTING.md).

## Repository Structure

Metro is a monorepo of npm packages under `packages/`, written in [Flow](https://flow.org/):

| Package                          | Responsibility                                                                                                                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `metro`                          | The bundler: [CLI](./CLI.md), [JavaScript API](./API.md), development server, [dependency graph](./Concepts.md#dependency-graph), serializers, and the [Fast Refresh](./FastRefresh.md) server |
| `metro-config`                   | Loading, validating and [merging](./Configuration.md#merging-configurations) configuration, and its defaults                                                                                   |
| `metro-file-map`                 | Crawling and [watching](./Concepts.md#file-watching) the project's files, and the [Haste](./Concepts.md#haste) map                                                                             |
| `metro-resolver`                 | [Module resolution](./Resolution.md)                                                                                                                                                           |
| `metro-transform-worker`         | The default [transformer](./Concepts.md#transformation), run in worker processes                                                                                                               |
| `metro-babel-transformer`        | The base [Babel transformer](./Configuration.md#babeltransformerpath)                                                                                                                          |
| `metro-transform-plugins`        | Babel plugins used by the transformer, e.g. for inlining build constants, constant folding and inline requires                                                                                 |
| `metro-runtime`                  | Code that runs in the app: the module system (`__d`, `__r`), the Fast Refresh client and `asyncRequire`                                                                                        |
| `metro-source-map`               | Source map generation and composition, including [Metro's extensions](./SourceMapFormat.md)                                                                                                    |
| `metro-symbolicate`              | Mapping stack traces back to source files with source maps                                                                                                                                     |
| `metro-cache`, `metro-cache-key` | [Cache stores](./Caching.md#built-in-cache-stores) and cache key hashing                                                                                                                       |
| `metro-minify-terser`            | The default [minifier](./Configuration.md#minifierpath)                                                                                                                                        |
| `metro-core`                     | Shared utilities, such as terminal output and logging                                                                                                                                          |
| `metro-babel-register`           | Running Metro's Flow source with Babel, without building it                                                                                                                                    |
| `ob1`, `buck-worker-tool`        | Type-checked 0- and 1-based offsets, and the Buck worker protocol                                                                                                                              |

Packages under `private/`, such as `metro-memory-fs`, are only used by Metro's own tests.

## Running Checks

`yarn test` runs type checking, linting, formatting checks, the build and all Jest tests. While iterating, run a subset, e.g. `yarn typecheck` (Flow) or `yarn jest packages/metro-resolver`.

## Public API

A package's public API is what its `package.json` `"exports"` expose, except `private/*` subpaths (e.g. `metro/private/DeltaBundler`), which expose internals that may change in any release.

Packages commit a snapshot of their public API, generated from their Flow source, in `API.md` (or `API-*.md`, one per entry point). After changing a public API, regenerate them with `yarn build-api-snapshots`. Jest fails if the snapshots are stale, so API changes show up in review.

## Testing Metro Changes inside a React Native Project

To test your changes end-to-end, we recommend using [`yarn link`][1] to register the local packages in your Metro clone, then switching a target project to use them. These instructions cover linking into a React Native app created with `npx @react-native-community/cli init MetroTestApp`, with this directory layout:

```sh
.
└── Development
    ├── metro        # metro clone
    └── MetroTestApp # target project
```

1. **Register local packages with `yarn link` in your `metro` clone**

   `yarn link` registers local package folders so that other projects can link to them. Use `npm exec --workspaces` to register every package in the repo — you choose which ones to link in the next step.

   ```sh
   npm exec --workspaces -- yarn link
   ```

2. **Use `yarn link` to replace Metro packages in your target project**

   Inside your target project folder, run `yarn link <package-name>` to use your registered `metro` packages in that project only. At minimum, link the `metro` and `metro-runtime` packages.

   ```sh
   # Links 3 packages
   yarn link metro metro-config metro-runtime
   ```

3. **Add the linked packages to Metro's `watchFolders`**

   The linked packages live outside the target project folder. Metro follows symlinks, but their targets must be within [`watchFolders`](./Concepts.md#project-root-and-watch-folders), so add to your `metro.config.js`:

   ```diff
   + const path = require('path');

     module.exports = {
   +   watchFolders: [
   +     path.resolve(__dirname, './node_modules'),
   +     // Include necessary file paths for `yarn link`ed modules
   +     path.resolve(__dirname, '../metro/packages'),
   +     path.resolve(__dirname, '../metro/node_modules'),
   +   ],
       ...
     };
   ```

4. **Run Metro**

   Start Metro from your target project. Restart it after any change to your `metro` clone or to the target project's `metro.config.js`.

   ```sh
   yarn react-native start
   ```

5. **(Optional) Clean up with `yarn unlink`**

   To go back to the published npm versions of the `metro` packages in your target project, repeat step 2 (and optionally step 1) with `yarn unlink`.

## Debug Logging

Metro uses the [debug](https://www.npmjs.com/package/debug) package to write logs under named debug scopes (e.g. `Metro:WatchmanWatcher`). To enable logs whose scope matches a pattern, set the `DEBUG` environment variable when starting Metro, e.g. for all of Metro's logs:

```sh
DEBUG='Metro:*' yarn metro serve
```

Separate multiple patterns with commas, e.g. for only file map and file watching logs:

```sh
DEBUG='Metro:FileMap,Metro:*Watcher' yarn metro serve
```

[1]: https://classic.yarnpkg.com/en/docs/cli/link
