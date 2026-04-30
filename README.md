# Knip `loadPackageManifest` bug — minimal reproduction

When **knip** is invoked from inside a single workspace package within a
hoisting monorepo (yarn workspaces, pnpm with `shamefully-hoist`, etc.), it
fails to load the `package.json` of installed dependencies that have been
hoisted to the repo root. As a result, the metadata that knip builds from
those manifests — `bin` entries, `peerDependencies`, types-included info — is
silently missing, and any downstream check that relies on that metadata
produces false positives.

This repo demonstrates two distinct symptoms with the same root cause:

- **Symptom A — script binaries:** A binary invoked from `package.json`
  `scripts` can't be resolved to its providing dep, so the dep is reported as
  an unused devDependency. Demonstrated by `packages/foo` (`cowsay`).
- **Symptom B — peer dependencies:** A package listed as a peerDependency of
  another (used) dep is reported as unused, because knip never reads the host
  dep's manifest to discover the peer relationship. Demonstrated by
  `packages/bar` (`chai`, peer of the imported `chai-as-promised`).

## Reproducing

```bash
git clone <this repo>
cd playground-knip-load-package-manifest-bug
yarn install
```

### Symptom A — script binaries (`packages/foo`)

`packages/foo/package.json` declares `cowsay` as a devDependency and uses its
binary in a script (`"moo": "cowsay hello"`). Valid, used dep.

```bash
# Bug: run from inside the workspace — cowsay reported unused
cd packages/foo
yarn knip --include dependencies
# → "Unused devDependencies (1)  cowsay  package.json:8:6"

# No bug: run from root in monorepo mode — no issues
cd ../..
yarn knip --workspace packages/foo --include dependencies
# → exit 0, no output
```

### Symptom B — peer dependencies (`packages/bar`)

`packages/bar/src/index.ts` directly imports `chai-as-promised`.
`chai-as-promised` declares `chai` as a non-optional peer dependency, and
`packages/bar/package.json` lists `chai` in devDependencies to satisfy that
peer requirement. Valid, used dep.

```bash
# Bug: run from inside the workspace — chai reported unused
cd packages/bar
yarn knip --include dependencies
# → "Unused devDependencies (1)  chai  package.json:5:6"

# No bug: run from root in monorepo mode — no issues
cd ../..
yarn knip --workspace packages/bar --include dependencies
# → exit 0, no output
```

## Why this matters

Some monorepos run knip per-package for performance, e.g. via lage / nx /
turborepo task runners. From inside the workspace dir, knip enters
single-project mode and the bug bites every dep whose manifest knip needs to
read in order to build its metadata caches — under yarn workspaces and
pnpm-shamefully-hoist that's *most of them*. In practice this surfaces as
false-positive unused-dep reports for both script-invoked binaries
(symptom A) and peers-of-installed-deps (symptom B).

The workaround users hit on is `ignoreDependencies: ['cowsay', 'chai', ...]`
in `knip.config.ts`. That suppresses the false positives but is unsatisfying
— it also masks genuine unused-dependency reports if they ever appear, and
it papers over a structural bug rather than fixing it.

## Root cause analysis

> ⚠️ **Author's note:** I have verified that the above reproduction works as described.
> But the root cause analysis and proposed fix was drafted by Claude.
> I can verify that the proposed code change fixes this reproduction,
> but I'm currently uncertain about whether it is the "right" fix all-up.
> If you are open to a fix PR, I am happy to take ownership of that and dive deeper.

### How knip uses installed packages' manifests

When knip processes a workspace, it iterates over each declared dependency
and reads the dep's installed `package.json` to extract metadata
(`packages/knip/src/manifest/index.ts`):

- The `bin` field, used to build a `binary-name → package-name` map (drives
  symptom A — resolving binaries invoked from `scripts`).
- The `peerDependencies` field, used to build a `peer-name → host-package`
  map. A package listed as a peer of an *imported* host is then treated as
  referenced via the host (drives symptom B —
  `DependencyDeputy.isReferencedDependency` walks this map).
- Whether the dep ships its own types (`types`/`typings`).

All of these are populated by `loadPackageManifest`
(`packages/knip/src/manifest/helpers.ts`), called once per declared dep:

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

In single-project mode (no `workspaces` config), knip computes the
workspace's `dir` as `join(cwd, '.')` — i.e., **`dir === cwd`** by
construction (`packages/knip/src/ConfigurationChief.ts`). So:

1. First lookup: `<workspace>/node_modules/<dep>/package.json` — does not
   exist, because yarn hoisted the dep to the repo-root `node_modules/`.
2. Fallback: skipped, because `dir !== cwd` is false.
3. → The dep's manifest is never loaded.
4. → None of the metadata derived from that manifest gets registered:
   - the `bin → package` map is missing the dep's binaries → **symptom A**;
   - the `peer → host` map is missing the dep's peer relationships →
     **symptom B**;
   - the dep is wrongly assumed not to ship its own types.
5. → Downstream checks see no evidence the dep is in use, and flag it as
   unused.

In **monorepo mode** (knip invoked from the repo root), `dir =
<root>/packages/foo` and `cwd = <root>` are different. The fallback fires,
finds `<root>/node_modules/<dep>/package.json`, and resolution succeeds.
That's why neither symptom shows up in monorepo mode.

## Suggested fix (AI-generated)

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
  tests should continue to pass. New test fixtures mirroring this repro
  (hoisted dep + single-project run) — one for binaries, one for peer
  dependencies — would lock in the fix for both symptoms.
