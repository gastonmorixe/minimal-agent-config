# @minimal-agent/config

Shared Biome, oxlint and TypeScript config for the minimal-agent repos.

Private. Each consumer pins it by git SHA (the **config pin**), the same way
the repos pin `@minimal-agent/core`.

Consumers today: `minimal-agent-core`, `minimal-agent-cli`,
`minimal-agent-plugins`. `minimal-agent-web` is not wired yet: it keeps its
own Biome and oxlint config.

## Work in the monorepo

This repo is a git submodule of
[gastonmorixe/minimal-agent](https://github.com/gastonmorixe/minimal-agent)
(the monorepo). Its
[README](https://github.com/gastonmorixe/minimal-agent#readme) has the full
workflow. Config differs from the other submodules in one way:

**An edit here is not live in the consumers.** The monorepo links the local
core checkout into its consumers, but not this repo. Core, plugins and cli
each install the config commit that they pin, also inside the monorepo. So a
change reaches a consumer only after these steps:

1. Make a branch (`git switch -c <name>`: the monorepo checks out each
   submodule on no branch, a "detached HEAD"). Commit, and push to `main` of
   this repo.
2. At the monorepo root, record the commit: `git add minimal-agent-config`,
   commit and push.
3. At the monorepo root, run `bun run pins:config --dry-run`, then
   `bun run pins:config`. It rewrites the config pin in core, plugins and
   cli, and updates the `bun.lock` of core and cli. It commits nothing.
4. In core, plugins and cli: run `bun run check`, commit
   `chore: pin config to <short sha>`, and push.
5. Record plugins and cli in the monorepo. For the core commit, run
   `bun run pins:core` at the monorepo root.

`bun run pins` at the monorepo root shows a note while a config pin is
behind, and the next command to run.

This repo has no gate, no CI and no commit hook of its own. Its test is the
gate of each consumer (step 4). Commit subjects follow Conventional Commits,
as in the other repos.

To try a change before you push it:

1. Save a copy of the installed file, for example
   `minimal-agent-cli/node_modules/@minimal-agent/config/biome.json`.
2. Copy your changed file over it. In the monorepo, core, plugins and cli
   share one installed copy, so all three gates see the change.
3. Run the gates.
4. Copy the saved file back. `bun install` does not do that: it leaves an
   edited file in place, and the gates then run on it with no sign.

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
three repos: `typescript/no-floating-promises` is an error, and 13 rules are
off. Two of those were turned off after the first version, with the reason in
the commit:

- `typescript/unbound-method`: fires on `const { join } = await import(...)`.
- `typescript/no-unsafe-enum-assignment`: fires on numeric literals passed to
  enum fields in test fixtures.

`tsconfig.base.json`: `strict`, `ESNext` target and module,
`moduleResolution: bundler`, `esModuleInterop`, `allowImportingTsExtensions`,
`noEmit`, `skipLibCheck`, `types: ["bun-types"]`. No `include`, no `exclude`.

## Consumer setup

These steps add config to a repo that does not use it yet. To move the pin
of a repo that has it, see "Work in the monorepo".

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

Run the `bun run` commands at the monorepo root.

1. Change the exact version in `package.json` `peerDependencies` here.
2. Commit, push, and record the commit in the monorepo
   (`git add minimal-agent-config`, commit, push).
3. In each repo, change the matching devDependency to the same version. Core
   and plugins hold it in their `workspaces.catalog`, cli and web in
   `devDependencies`. Web has no config pin, but it has the tool versions.
4. Change the same entry in the `catalog` of the monorepo root.
5. Run `bun run pins:config`. It moves the `@minimal-agent/config` pin in
   core, plugins and cli to that commit, and updates the `bun.lock` of core
   and cli. Then run `bun run pins:lock web`: web has no config pin, so
   `pins:config` does not update its `bun.lock`.
6. Run `bun install`, then `bun run check`. It runs `bun run catalogs:check`,
   which fails while one of these places, or the `peerDependencies` here,
   differs from the root catalog.
7. Commit and push in each repo, with its `bun.lock`. Then record the commits
   in the monorepo ("Work in the monorepo", steps 4 and 5).
8. Run `bun run pins:verify`. It installs each repo alone, at the commit that
   the monorepo records.

Bump one tool per commit.

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

Run the gate of the consumer:

```bash
bun run check
```

Or run one tool:

```bash
bunx biome format . --reporter=summary
bunx oxlint
bunx tsc --noEmit
```

## License

Proprietary. All rights reserved. `package.json` has `"license": "UNLICENSED"`.
