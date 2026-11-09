---
id: assets
title: Assets
---

An _asset_ is a file, such as an image or a font, that your code requires like a JavaScript module:

```js
const logo = require('./logo.png');
```

Metro treats a file as an asset if its extension is listed in [`resolver.assetExts`](./Configuration.md#assetexts), which by default includes common image, video and audio formats.

## Variants

An asset can have variants for a specific [platform](./Concepts.md#platform) (dotted suffix, e.g. `.ios`) or, for images, screen density (`@` suffix, e.g. `@2x`, see [`resolver.assetResolutions`](./Configuration.md#assetresolutions)). For example, `logo@2x.ios.png` is the 2x iOS variant of `logo.png`.

`require('./logo.png')` refers to all of them: Metro picks the files for the bundle's platform, and includes every density, e.g. `logo.png`, `logo@2x.png` and `logo@3x.png`, so the app can load the best one for the screen.

## In the bundle

An asset is a [module](./Concepts.md#module) that passes a description of the asset (name, type, dimensions, available densities, and URL path on the server) to `registerAsset()` of the [`transformer.assetRegistryPath`](./Configuration.md#assetregistrypath) module, and exports the result. In React Native, `require` therefore returns a numeric ID that components such as `<Image>` accept.

To change the description, e.g. to add data computed from the file, use [`transformer.assetPlugins`](./Configuration.md#assetplugins).

## Serving and copying assets

The development server serves asset files, generally as-is, under `/assets/` (see [`transformer.publicPath`](./Configuration.md#publicpath)), e.g. `/assets/src/logo@2x.png?platform=ios` for `src/logo@2x.png`.

Release builds load assets from the app instead. [`runBuild`](./API.md#async-runbuildconfig-options) with `assets: true` returns the bundle's assets and their files, for build tools to copy into the app. React Native's `bundle` command does this with `--assets-dest`.
