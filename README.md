# @minimal-agent/config

Shared Biome, oxlint and TypeScript config for the minimal-agent repos.

Private. Pin it by git SHA, the same way the repos pin `@minimal-agent/core`.

## What it contains

| Export | File | Consumer line |
| --- | --- | --- |
| `@minimal-agent/config/biome` | `biome.json` | `"extends": ["@minimal-agent/config/biome"]` |
| `@minimal-agent/config/oxlintrc.json` | `oxlintrc.json` | `"extends": ["./node_modules/@minimal-agent/config/oxlintrc.json"]` |
| `@minimal-agent/config/tsconfig.json` | `tsconfig.base.json` | `"extends": "@minimal-agent/config/tsconfig.json"` |

## Consumer setup

Add the dependency:

```jsonc
"devDependencies": {
  "@minimal-agent/config": "git+ssh://git@github.com/gastonmorixe/minimal-agent-config.git#<full-sha>"
}
```

Then make three small edits per repo.

### biome.json

```jsonc
{
  "$schema": "https://biomejs.dev/schemas/<version>/schema.json",
  "extends": ["@minimal-agent/config/biome"],
  "files": { "includes": ["src/**"] }
}
```

Keep the local `$schema` and the local `files.includes`. The base sets no
`files.includes`, because Biome appends those lists rather than replacing them.

Use the extensionless subpath. `@minimal-agent/config/biome.json` fails with
`Could not resolve ... module not found`, because Biome resolves through the
`exports` map and does not add an extension.

### .oxlintrc.json

```jsonc
{
  "extends": ["./node_modules/@minimal-agent/config/oxlintrc.json"],
  "categories": { "correctness": "error", "suspicious": "error", "perf": "warn" },
  "settings": { "jsdoc": { "ignorePrivate": true, "ignoreInternal": true } },
  "jsPlugins": [
    { "name": "jsdoc-js", "specifier": "eslint-plugin-jsdoc" },
    { "name": "tsdoc", "specifier": "eslint-plugin-tsdoc" }
  ],
  "ignorePatterns": ["node_modules/**"],
  "overrides": []
}
```

oxlint inherits only `plugins`, `options` and `rules` from this base. Everything
else in the example above must stay local, or it silently stops applying:
`categories`, `settings`, `jsPlugins`, `ignorePatterns`, `overrides`.

A bare package name in `extends` fails (`invalid config file ... NotFound`).
Use the relative path shown.

Keep `eslint-plugin-jsdoc` and `eslint-plugin-tsdoc` in the consumer's own
devDependencies. `jsPlugins` specifiers resolve from the consumer root.

### tsconfig.json

```jsonc
{
  "extends": "@minimal-agent/config/tsconfig.json",
  "include": ["src"],
  "exclude": ["node_modules"]
}
```

The base has `compilerOptions` only. A base `include` resolves from the base
file and breaks the consumer. Keep `include` and `exclude` local. Extra flags
such as `verbatimModuleSyntax`, `resolveJsonModule` and `isolatedModules` stack
on top.

`moduleResolution: "bundler"` is required. Other values fail with TS5110 or
TS5108.

## Required peer packages

`@biomejs/biome`, `oxlint`, `oxlint-tsgolint` (needed by `options.typeAware`)
and `typescript` must be devDependencies in the consumer. Only `typescript` is
optional in this package's peer set. Bun installs the other three peers
automatically, verified on a clean `git+ssh` install, and `node_modules/.bin`
then holds `biome`, `oxlint` and `tsgolint`.

`oxlint` looks for the `tsgolint` binary at `node_modules/.bin/tsgolint`. The
`oxlint-tsgolint` package provides that name.

## Verify a consumer

```bash
bunx biome format . --reporter=summary
bunx oxlint src
bunx tsc --noEmit
```
