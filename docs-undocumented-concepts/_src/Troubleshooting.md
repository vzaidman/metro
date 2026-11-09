---
id: troubleshooting
title: Troubleshooting
---

Start with the error Metro reported: most failures say whether an import, a file or the server is at fault. Find your symptom below before resetting anything. Resetting is last, under [Last resort: reset](#last-resort-reset).

## Reading bundling errors

When a bundle request fails, Metro responds with an error object (custom clients receive the same error over the Fast Refresh connection). Two signals narrow it down:

- **HTTP status.** `404` means the bundle itself could not be found: usually a wrong entry file or URL. `500` means Metro found the bundle but building it failed, for example an import inside it could not be resolved. Fix your code or config, not the URL.
- **Error `type`.** The common types, and what to do:

  | `type`                                                                 | Meaning                                                                                  |
  | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
  | `UnableToResolveError`                                                 | An import could not be resolved. See below                                               |
  | `TransformError`                                                       | A file could not be compiled. See below                                                  |
  | `NotFoundError`                                                        | A required file is missing                                                               |
  | `AmbiguousModuleResolutionError`                                       | Several files provide one Haste module. Delete or rename all but one                     |
  | `GraphNotFoundError`, `RevisionNotFoundError`, `ResourceNotFoundError` | Metro no longer holds that bundle in memory. Reload the app; restart Metro if it repeats |
  | `InternalError`                                                        | A bug in Metro or a custom resolver or transformer. Note the file and line shown         |

Resolution and transform errors include the file, and usually the line, to look at. Read that file first.

## Unable to resolve module

Metro cannot find a file for an import such as `Unable to resolve module ./Button from /app/index.js`. Check, in order:

1. The path and casing are correct, and the file exists.
2. Following symlinks, the file is under your [`projectRoot`](./Configuration.md#projectroot) or [`watchFolders`](./Configuration.md#watchfolders) and is not excluded by [`resolver.blockList`](./Configuration.md#blocklist). This is the usual monorepo cause; see [Monorepos](./Monorepos.md#when-metro-cannot-see-a-file-that-exists).
3. Its extension is in [`resolver.sourceExts`](./Configuration.md#sourceexts), or [`resolver.assetExts`](./Configuration.md#assetexts) for an asset.
4. For a package import, the package is installed and exposes that entry point. If the error mentions `"exports"`, see [Package Exports Support](./PackageExports.md#troubleshooting).
5. For platform files such as `Button.ios.js`, the [platform](./Concepts.md#platform) is in [`resolver.platforms`](./Configuration.md#platforms).

For how each import is resolved step by step, see [Module Resolution](./Resolution.md). Do not work around a resolution error with [`resolver.extraNodeModules`](./Configuration.md#extranodemodules) until the checks above pass; it hides the real cause.

## Transform errors

A transform error means a file was found but could not be compiled, most often a syntax error or syntax your Babel config does not handle. The error shows a code frame pointing at the location. Fix that file. If the syntax is valid, check that your [`transformer.babelTransformerPath`](./Configuration.md#babeltransformerpath) and Babel config support it, and that a custom transformer returns the `ast` and `metadata` Metro expects (see [Writing a custom transformer](./Configuration.md#writing-a-custom-transformer)).

## Edits not picked up

Metro only reacts to changes it [watches](./Concepts.md#file-watching):

1. The edited file must be under `projectRoot` or `watchFolders`, following symlinks to its real path. Edited a symlinked package with no effect? Watch its real directory and restart Metro. See [Monorepos](./Monorepos.md#pnpm-and-symlinks).
2. Watching must be on. It is off when the `CI` environment variable is set, and [`runBuild`](./API.md#async-runbuildconfig-options) never watches.
3. Watchman, if you use it, must be healthy. Clearing its watches is step 1 under [Last resort: reset](#last-resort-reset).

To check the watcher itself, enable [`watcher.healthCheck`](./Configuration.md#healthcheckenabled): Metro periodically writes a file into your project and warns if it does not observe the change, which points at the watcher rather than your setup.

If edits apply but the app always fully reloads, that is Fast Refresh, not watching: see [Full reloads](./FastRefresh.md#full-reloads).

## Stale bundle

Metro keeps two kinds of state, and they go stale differently:

- **Transform cache:** Compiled results per file, keyed by contents and options (see [Caching](./Caching.md)). A custom transformer whose output depends on a file its cache key omits serves stale results until the key changes. Reset it with `--reset-cache` (step 3 below).
- **File map and dependency graph:** Built at startup and updated by the watcher. Files added outside watched directories, or config changes such as `watchFolders`, only take effect after a restart.

If one client sees an old bundle while another sees the new one, the first is probably revalidating with `If-Modified-Since` and receiving `304` (see [Development server URLs](./API.md#development-server-urls)). Reload without the cached validator, or restart Metro.

## Turning on debug logging

Metro logs under named scopes through the [`debug`](https://www.npmjs.com/package/debug) package. Enable them with the `DEBUG` environment variable when starting Metro:

```bash
# Everything
DEBUG='Metro:*' npx react-native start

# Only file map and watching
DEBUG='Metro:FileMap,Metro:*Watcher' npx react-native start
```

Start narrow: `Metro:*` is verbose. Watching and file-map scopes are the useful ones for "edits not picked up" and "file not found" issues. When reporting a bug, include the relevant scoped output instead of the full log.

## Last resort: reset

Try these steps in order, checking after each one whether the issue is resolved.

1. **Clear Watchman watches:** `watchman watch-del-all`

   If you use [Watchman](https://facebook.github.io/watchman/), this helps when Metro doesn't pick up file changes, or reports errors about files you've added, moved or deleted.

2. **Reinstall dependencies:** delete `node_modules` and run `yarn install` (or your package manager's equivalent).

   Helps when a package can't be found or is in a broken state, e.g. after switching branches or upgrading dependencies.

3. **Reset Metro's cache:** pass the `--reset-cache` flag (e.g. `npx react-native start --reset-cache`), or add [`resetCache: true`](./Configuration.md#resetcache) to your Metro configuration file.

   This clears Metro's [cache](./Caching.md) of transformed files and rebuilds its map of your project's files from scratch. Helps when the bundle doesn't reflect your latest code or configuration.

4. **Delete Metro's cache files:** `rm -rf ${TMPDIR:-/tmp}/metro-*`

   Try this if resetting the cache didn't help. It removes the cache files Metro keeps in your temp directory by default.

5. **Update Metro** to the [latest version](https://www.npmjs.com/package/metro), in case your issue has been fixed in a newer release.

## Still unresolved?

See the [Help](/help) pages.
