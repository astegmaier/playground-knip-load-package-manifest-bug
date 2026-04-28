# Knip `loadPackageManifest` bug — minimal reproduction

When **knip** is invoked from inside a single workspace package within a hoisting
monorepo (yarn workspaces, pnpm with `shamefully-hoist`, etc.), it fails to
resolve script-invoked binaries to their providing package, and reports the
package as an unused devDependency.

## What the bug looks like

`packages/foo/package.json` declares `cowsay` as a devDependency and uses its
binary in a script:

```json
{
  "scripts": { "moo": "cowsay hello" },
  "devDependencies": { "cowsay": "^1.6.0" }
}
```

This is a perfectly valid, used dependency. But knip reports it as unused:

```
$ cd packages/foo
$ yarn knip --include dependencies

Unused devDependencies (1)
cowsay  package.json:8:6
```

Running knip from the repo root (monorepo mode) reports correctly — `cowsay` is
used. The bug is specific to invoking knip from within the workspace dir.

## Reproducing

```bash
git clone <this repo>
cd playground-knip-load-package-manifest-bug
yarn install

# Bug: run from inside the workspace — `cowsay` reported unused
cd packages/foo
node ../../node_modules/knip/dist/cli.js --include dependencies
# → "Unused devDependencies (1)  cowsay"

# No bug: run from root in monorepo mode — silent (no issues)
cd ../..
node node_modules/knip/dist/cli.js --workspace packages/foo --include dependencies
# → exit 0, no output
```

Versions used while authoring this repro:

- yarn 4.12.0 (with `nodeLinker: node-modules`)
- knip 6.7.0
- node 24.14.0

## Why this matters

Some monorepos run knip per-package for performance, e.g. via lage / nx /
turborepo task runners. From inside the workspace dir, knip enters
single-project mode and the bug bites every script-invoked binary whose
providing dep is hoisted to the repo root — which under yarn workspaces and
pnpm-shamefully-hoist is *most of them*.

The workaround users hit on is `ignoreDependencies: ['cowsay', ...]` in
`knip.config.ts`. That suppresses the false positive but is unsatisfying — it
also masks genuine unused-dependency reports if they ever appear.

## Root cause analysis

### How knip resolves binaries

When knip parses `package.json` `scripts`, it creates an input tagged
`binary: 'cowsay'`. To learn which dep provides that binary, it consults a
`binary-name → package-name` map built by reading the `bin` field of every
installed dep's `package.json`.

That map is populated by `loadPackageManifest`
(`packages/knip/src/manifest/helpers.ts`), called once per declared dep. The
function tries two locations:

```ts
export const loadPackageManifest = ({ dir, packageName, cwd }: LoadPackageManifestOptions) => {
  try {
    return _require(join(dir, 'node_modules', packageName, 'package.json'));
  } catch (_error) {
    if (dir !== cwd) {
      try {
        return _require(join(cwd, 'node_modules', packageName, 'package.json'));
      } catch (_error) {
        // Explicitly suppressing errors here
      }
    }
    // Explicitly suppressing errors here
  }
};
```

- `dir` is the workspace's directory.
- `cwd` is knip's working directory.

### Why this fails in single-project mode within a hoisting monorepo

In single-project mode (no `workspaces` config), knip computes the workspace's
`dir` as `join(cwd, '.')` — i.e., **`dir === cwd`** by construction
(`packages/knip/src/ConfigurationChief.ts`). So:

1. First lookup: `<workspace>/node_modules/cowsay/package.json` — does not
   exist, because yarn hoisted `cowsay` to the repo-root `node_modules/`.
2. Fallback: skipped, because `dir !== cwd` is false.
3. → `cowsay`'s manifest is never loaded.
4. → The `bin → package` map never gets a `cowsay` entry.
5. → The script's binary input cannot be resolved.
6. → `cowsay` is reported as an unused devDependency.

In **monorepo mode** (knip invoked from the repo root), `dir =
<root>/packages/foo` and `cwd = <root>` are different. The fallback fires,
finds `<root>/node_modules/cowsay/package.json`, and resolution succeeds. That
explains why the bug only shows up in single-project mode.

## Suggested fix (AI-generated)

> ⚠️ **Author's note:** This proposed fix was drafted by Claude (an AI
> assistant). I have not yet validated it against knip's full test suite or
> edge cases — treat as a starting point for discussion, not a finished patch.

Have `loadPackageManifest` walk up parent directories from `dir`, the way
Node's own `require` resolution does. This handles arbitrary depths of
hoisting transparently:

```ts
// packages/knip/src/manifest/helpers.ts
import { dirname, join } from '../util/path.ts';
import { _require } from '../util/require.ts';

export const loadPackageManifest = ({ dir, packageName, cwd }: LoadPackageManifestOptions) => {
  // Walk up parent directories from `dir`, looking for the package's manifest in any
  // ancestor `node_modules`. Mirrors Node.js's module-resolution algorithm. This
  // handles hoisting monorepos (yarn workspaces, pnpm with `shamefully-hoist`) where
  // deps live one or more levels above the workspace `dir`.
  let current = dir;
  while (true) {
    try {
      return _require(join(current, 'node_modules', packageName, 'package.json'));
    } catch (_error) {
      // continue walking up
    }
    const parent = dirname(current);
    if (parent === current) break;
    current = parent;
  }

  // Final fallback at `cwd`, in case `cwd` is not an ancestor of `dir` (e.g. when knip
  // is run with --directory pointing somewhere unrelated to the workspace tree).
  if (dir !== cwd) {
    try {
      return _require(join(cwd, 'node_modules', packageName, 'package.json'));
    } catch (_error) {
      // suppressed
    }
  }
};
```

### Why a directory walk over `require.resolve(name + '/package.json')`

A tempting one-liner alternative is:

```ts
return _require(require.resolve(`${packageName}/package.json`, { paths: [dir, cwd] }));
```

This relies on `require.resolve` walking up `node_modules`, which it does. But
since Node 12, packages can declare `exports` and that field gates which
internal paths are reachable from outside. Many real-world packages forget to
list `./package.json`, in which case `require.resolve('pkg/package.json')`
throws `ERR_PACKAGE_PATH_NOT_EXPORTED`. The directory-walk approach reads the
file directly off disk and sidesteps that pitfall.

### Risk assessment

- **Loop termination:** the loop stops when `dirname(current) === current`,
  which is true at the filesystem root on every platform.
- **Wrong package picked up:** in a hoisting monorepo, the closest
  `node_modules/<pkg>` to a workspace is the *correct* one to use (this is how
  Node resolves it at runtime). Walking outward strictly mirrors that.
- **Performance:** the loop is bounded by the depth of the workspace from the
  filesystem root (typically <10). Each iteration is one `_require` call that
  short-circuits on the first hit. Negligible.
- **Tests:** knip's existing test suite covers the happy path (deps found
  in `dir/node_modules`); the new behaviour is a strict superset, so existing
  tests should continue to pass. A new test fixture mirroring this repro
  (hoisted dep + single-project run) would lock in the fix.

### Where to file

If filing upstream: <https://github.com/webpro-nl/knip/issues/new>. The change
touches one file (`packages/knip/src/manifest/helpers.ts`) plus a fixture and
test under `packages/knip/test/`.
