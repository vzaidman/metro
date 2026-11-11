---
id: symbolication
title: Symbolicating Stack Traces
---

_Symbolication_ turns locations in a built bundle back into source files, lines and function names. It is how a production crash becomes debuggable.

## How it works

Your app does not run your source files; it runs one generated, usually minified bundle. So a crash report can only point into that file, as `file:line:column` — for example `index.bundle:1:2345` means line 1, column 2345 of `index.bundle`. Minified code packs whole modules onto very long lines and shortens names, so on its own that location tells you nothing about which of your functions crashed. The bundle's source map records, for each generated position, the original file, line and name it came from. Symbolication applies that map to every frame of a stack trace (or profile), turning `index.bundle:1:2345` into something like `formatPrice` at `src/cart/pricing.js:42` — a place you can open and fix. Without the map from that exact build, the locations cannot be recovered.

For the source map format itself, see [Source Map Format](./SourceMapFormat.md).

## In development

In development, Metro does this for you: React Native sends stack traces to the development server's `/symbolicate` endpoint, which symbolicates them with the source map of the bundle it served.

To change how the development server presents stack traces, for example collapsing frames from library internals, use [`symbolicator.customizeFrame`](./Configuration.md#customizeframe) and [`symbolicator.customizeStack`](./Configuration.md#customizestack). These only affect the `/symbolicate` endpoint in development; release symbolication below is unaffected.

## In release builds

For release builds, you symbolicate with the source map from that build and the `metro-symbolicate` command.

### Keeping the source map

Symbolication needs the exact source map (`.map` file) produced with the bundle that crashed. A map from any other build gives wrong locations. Keep it with the build it belongs to, for example upload it to your crash reporter but do not ship it in the app.

To write a bundle and its map with [`runBuild`](./API.md#async-runbuildconfig-options):

```js
const Metro = require('metro');

const config = await Metro.loadConfig();
await Metro.runBuild(config, {
  entry: 'index.js',
  platform: 'ios',
  dev: false,
  out: 'dist/main.js',
  sourceMap: true,
  sourceMapOut: 'dist/main.map',
});
```

The [`metro build`](./CLI.md#build-entry) command takes the same options as `--source-map` and related flags. In development, request the map from the server by changing `.bundle` to `.map` in the bundle URL, with the same query parameters (see [Source maps](./Bundling.md#source-maps)).

### Stack traces

Install the standalone tool once, then pipe a stack trace to it with the map. `metro-symbolicate` is published as its own package, locked to the same version as the rest of Metro (see [Packages](./Packages.md)):

```bash
npm install --save-dev metro-symbolicate
cat stack.txt | npx metro-symbolicate dist/main.map
```

Each frame that matches the map is rewritten with its source file, line, column and function name. Frames that do not match pass through unchanged, which usually means the map does not belong to that bundle.

### Single locations

Pass a line, and optionally a column, from the bundle instead of a whole stack:

```bash
# Line 1, column 2345 in the bundle
npx metro-symbolicate dist/main.map 1 2345

# Line 12 of module 7 in an indexed (RAM) bundle map
npx metro-symbolicate dist/main.map 7.js 12
```

Lines are counted from 1 and columns from 0. If your input counts differently, shift it with `--input-line-start` and `--input-column-start`; `--output-line-start` and `--output-column-start` do the same for the output. Add `--no-function-names` to report identifier names instead of function names.

### Profiles

`metro-symbolicate` also rewrites profiling files against a map:

```bash
npx metro-symbolicate dist/main.map trace.cpuprofile
npx metro-symbolicate dist/main.map trace.profmap
npx metro-symbolicate dist/main.map --attribution < in.jsonl > out.jsonl
```

The first two forms take a Chrome CPU profile (`.cpuprofile`) or a Hermes profile map (`.profmap`); the last rewrites JSON Lines attribution input from stdin to stdout.

### Hermes crashes

Hermes crash and coverage data are JSON, read from stdin and written as JSON to stdout. Pass exactly one of:

- `--hermes-crash`: Symbolicate a Hermes minidump trace.
- `--hermes-coverage`: Symbolicate a Hermes coverage trace.

```bash
cat hermes-crash.json | npx metro-symbolicate dist/main.map --hermes-crash
```

Heap snapshots (`.heapsnapshot` and `.heaptimeline` files) are symbolicated the same way as profiles: `npx metro-symbolicate dist/main.map heap.heapsnapshot`.
