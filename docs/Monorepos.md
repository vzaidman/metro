---
id: monorepos
title: Monorepos
---

In a monorepo, your app imports code from sibling packages, often through symlinks in `node_modules`. Metro only knows about files under its [project root and watch folders](./Concepts.md#project-root-and-watch-folders), so shared packages must be listed explicitly. Without that, Metro fails with errors such as `Unable to resolve module` for files that exist on disk, or silently serves stale code. This page is the checklist; option details are in [Configuring Metro](./Configuration.md).

## Example layout

```text
monorepo/
  apps/
    mobile/        <- projectRoot, contains metro.config.js and index.js
      index.js       imports the ui package
  packages/
    ui/            <- the ui package, symlinked into apps/mobile/node_modules
```

`index.js` imports the `ui` package, which resolves through the symlink to `packages/ui`. That directory is outside `apps/mobile`, so Metro cannot see it until it is added to `watchFolders`.

## Checklist

In the app's `metro.config.js`:

```js
const path = require('path');

const projectRoot = __dirname;
const monorepoRoot = path.resolve(projectRoot, '../..');

module.exports = {
  projectRoot,
  watchFolders: [monorepoRoot],
};
```

1. **Set [`projectRoot`](./Configuration.md#projectroot)** to the app directory (the directory containing `metro.config.js`). Entry points and bundle URLs are resolved from here.
2. **List every shared directory in [`watchFolders`](./Configuration.md#watchfolders)**, including the real targets of symlinks. A symlinked package whose target is not watched is invisible to Metro, even though Node.js and your editor find it. Watching the monorepo root, as above, covers all packages at once; list individual packages instead if the repo is large and startup is slow.
3. **Restart Metro** after changing `projectRoot` or `watchFolders`. The file map is built at startup, and newly watched directories are only crawled then. Note that [`loadConfig`](./API.md#loadconfigoptions-defaultconfigoverrides) always adds `projectRoot` itself to `watchFolders`; you do not need to repeat it.

:::note

In React Native, merge these options on top of `getDefaultConfig(__dirname)` from `@react-native/metro-config` as usual (see [Configuring Metro](./Configuration.md)). Only `projectRoot` and `watchFolders` are monorepo-specific.

:::

## pnpm and symlinks

Package managers such as pnpm install dependencies as symlinks to a shared store. Metro resolves the real path behind a symlink, so the store and the linked packages must be inside `projectRoot` or `watchFolders` to be bundled and watched. If an edit to a linked package does not trigger [Fast Refresh](./FastRefresh.md), its real directory is almost certainly not watched.

Two resolver options cover layouts that `watchFolders` alone does not:

- [`resolver.nodeModulesPaths`](./Configuration.md#nodemodulespaths): Look in additional `node_modules` directories, such as the monorepo root's, when resolving a package.
- [`resolver.extraNodeModules`](./Configuration.md#extranodemodules): Map a package name to a specific directory. Use this sparingly, for example to force a single copy of a package; each mapping bypasses normal [resolution](./Resolution.md).

If several apps in one repo are served by one Metro instance, [`server.unstable_serverRoot`](./Configuration.md#unstable_serverroot-experimental) changes the directory bundle URLs are resolved against, so `/apps/mobile/index.bundle` builds that app's entry point.

## When Metro cannot see a file that exists

Work through these in order:

1. Is the file, following symlinks to its real path, under `projectRoot` or a `watchFolders` entry? If not, add its directory and restart Metro.
2. Is it excluded by [`resolver.blockList`](./Configuration.md#blocklist)? Blocked files are treated as missing.
3. Does its extension appear in [`resolver.sourceExts`](./Configuration.md#sourceexts) (or [`resolver.assetExts`](./Configuration.md#assetexts) for assets)? Other files are not modules.
4. For package imports, does the package's `package.json` expose the file? See [Package Exports Support](./PackageExports.md) and [Unable to resolve module](./Troubleshooting.md#unable-to-resolve-module).

Avoid watching more than you need on very large repos: every watched directory is crawled at startup. Start with the directories your app actually imports.
