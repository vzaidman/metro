---
id: getting-started
title: Getting Started
---

Metro is the JavaScript bundler used by React Native. This page shows how to install Metro and run it from the command line or your own Node.js code. For how Metro works, see [Concepts](./Concepts.md).

:::note Using React Native?

React Native projects already include Metro: `npx react-native start` starts a Metro development server, and `npx react-native bundle` builds a bundle. To customize Metro, edit your `metro.config.js` (see [Configuring Metro](./Configuration.md)). Read on to use Metro outside React Native or build your own tooling on it.

:::

Install Metro using [`npm`](https://www.npmjs.com/) or [`yarn`](https://yarnpkg.com/):

```bash
npm install --save-dev metro metro-core
```

```bash
yarn add --dev metro metro-core
```

## Running from the command line

```bash
# Start a development server
npx metro serve
# Build a bundle from index.js into bundle.js
npx metro build index.js --out bundle.js
```

See [Metro CLI Options](./CLI.md) for all commands and options.

## Running programmatically

The `metro` module (`const Metro = require('metro')`) exports these main functions:

- [`runServer`](./API.md#async-runserverconfig-options): Start a complete development server, as `npx react-native start` does.
- [`runBuild`](./API.md#async-runbuildconfig-options): Build a bundle once, e.g. in a build step, as `npx react-native bundle` does.
- [`createConnectMiddleware`](./API.md#createconnectmiddlewareconfig-options): Add Metro to an existing Express or Connect server.
- [`runMetro`](./API.md#async-runmetroconfig-options): Get a Metro server and handle HTTP yourself.

Each takes your Metro configuration, loaded with [`Metro.loadConfig()`](./API.md#loadconfigoptions-defaultconfigoverrides), as the first argument:

```js
const config = await Metro.loadConfig();
await Metro.runBuild(config, {entry: 'index.js', out: 'bundle.js'});
```

See the [Bundling API](./API.md) for every function and option.

## Configuration

Metro reads its configuration from `metro.config.js` in the project root, and merges it with its defaults. For example, to also watch a shared directory outside the project, limit Metro to two worker processes, and serve on another port:

```js
// metro.config.js
const path = require('path');

module.exports = {
  watchFolders: [path.resolve(__dirname, '../shared')],
  maxWorkers: 2,
  server: {
    port: 8082,
  },
};
```

See [Configuring Metro](./Configuration.md) for all configuration options, including how to customize the [transformer](./Configuration.md#babeltransformerpath) and [resolver](./Configuration.md#resolverequest).

## Requesting bundles

A running Metro server serves bundles, source maps and [assets](./Bundling.md#assets) over HTTP. In development, a React Native app loads its JavaScript by requesting a URL like `http://localhost:8081/index.bundle?platform=ios&dev=true`: the bundle built from `index.js`, for iOS, in development mode. See [Bundle URLs](./Bundling.md#bundle-urls) for all options, and how to request source maps.
