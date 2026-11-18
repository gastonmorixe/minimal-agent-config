# @minimal-agent/config

Shared Biome, oxlint and TypeScript config for the minimal-agent repos.

Private. Pin it by git SHA, the same way the repos pin `@minimal-agent/core`.

Consumers today: `minimal-agent-core`, `minimal-agent-cli`,
`minimal-agent-plugins`. `minimal-agent-web` is not wired yet.

## What it contains

| Export | File | Consumer line |
| --- | --- | --- |
| `@minimal-agent/config/biome` | `biome.json` | `"extends": ["@minimal-agent/config/biome"]` |
| `@minimal-agent/config/oxlintrc.json` | `oxlintrc.json` | `"extends": ["./node_modules/@minimal-agent/config/oxlintrc.json"]` |
| `@minimal-agent/config/tsconfig.json` | `tsconfig.base.json` | `"extends": "@minimal-agent/config/tsconfig.json"` |

## What is in the base, and why

`biome.json`: formatter (2-space indent, 100 columns), the custom
`organizeImports` groups, double quotes, `asNeeded` semicolons, `all` trailing
commas, JSON without trailing commas, and the Biome linter off.

`oxlintrc.json`: the plugin list, `options.typeAware`,
`options.reportUnusedDisableDirectives`, and the rule set that is policy for all
three repos. Two rules are deliberately off there, with reasons recorded in the
commits:

- `typescript/unbound-method`: fires on `const { join } = await import(...)`.
- `typescript/no-unsafe-enum-assignment`: fires on numeric literals passed to
  enum fields in test fixtures.

`tsconfig.base.json`: `strict`, `ESNext` target and module,
`moduleResolution: bundler`, `esModuleInterop`, `allowImportingTsExtensions`,
`noEmit`, `skipLibCheck`, `types: ["bun-types"]`. No `include`, no `exclude`.

## Consumer setup

Add the dependency:

```jsonc
"devDependencies": {
  "@minimal-agent/config": "git+ssh://git@github.com/gastonmorixe/minimal-agent-config.git#<full-sha>"
}
```

Do not use `workspace:*` here. The submodules are separate install roots, and a
standalone clone has no monorepo root above it, so `workspace:*` fails with
`the root package depends on workspace ... not on disk`. A git+ssh SHA pin works
in a standalone clone, in CI, and inside the root workspace.

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
The `$schema` version must match the installed Biome, or Biome prints a schema
mismatch notice on every run.

If a consumer needs different `organizeImports` groups, write the local
`assist` block in OBJECT form, not as the shorthand `"organizeImports": "on"`.
The shorthand does not override the base groups. The object form does:

```jsonc
"assist": {
  "enabled": true,
  "actions": { "source": { "organizeImports": { "level": "on" } } }
}
```

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

The peers are pinned to exact versions on purpose. A range let Bun resolve
Biome 2.6.6 for a repo that ran 2.5.15, so the shared base was tested against a
binary the repos did not have.

`oxlint` looks for the `tsgolint` binary at `node_modules/.bin/tsgolint`. The
`oxlint-tsgolint` package provides that name.

## Upgrading a tool

1. Change the exact version in `package.json` `peerDependencies` here.
2. Commit and push. Note the new SHA.
3. In each consumer, bump the `@minimal-agent/config` pin and the matching
   devDependency to the same version. Core and plugins hold it in their
   `workspaces.catalog`, cli and web in `devDependencies`.
4. Change the same entry in the `catalog` of the monorepo root.
5. Run `bun install` at the monorepo root, then `bun run check`. It runs
   `bun run catalogs:check`, which fails while one of these places, or the
   `peerDependencies` here, differs from the root catalog.
6. The root install does not read a consumer's own `bun.lock`. Run
   `bun run pins:lock` at the monorepo root to update each committed one,
   commit it in its repo, then run `bun run pins:verify`.

Expect fallout, and fix it rather than pinning back:

- A new Biome formats more file types. 2.6 formats Markdown, which padded
  tables across the repo. Keep that diff separate from a logic change.
- A new Biome version needs the `$schema` URL bumped in every consumer, or
  every run prints a schema-mismatch notice.
- New oxlint rules become errors. Decide per rule: fix the code, or disable it
  in the base with the reason in the commit message.
- New `eslint-plugin-jsdoc` rules can require doc comments that were never
  there. Add the comment; do not turn the rule off.

## Verify a consumer

```bash
bunx biome format . --reporter=summary
bunx oxlint src
bunx tsc --noEmit
```
