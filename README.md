# Knip `loadPackageManifest` bug — patched-fix branch

This branch applies a candidate fix to **knip 6.7.0** via `yarn patch`,
addressing the bug reproduced on the [`main`
branch](https://github.com/astegmaier/playground-knip-load-package-manifest-bug/tree/main).
See `main` for the full bug description, root-cause analysis, and rationale.

## What changed

A single yarn patch against
`knip/dist/manifest/helpers.js` (see
`.yarn/patches/knip-npm-6.7.0-*.patch`). `loadPackageManifest` now walks up
the directory tree from the workspace `dir`, looking for the dependency in
each ancestor `node_modules/`, mirroring Node's own module-resolution
algorithm. This handles hoisting monorepos (yarn workspaces, pnpm
`shamefully-hoist`) where the dep lives one or more levels above the
workspace `dir`. The original `cwd` fallback is kept as a final attempt for
the case where `cwd` is not an ancestor of `dir` (e.g. `--directory`
pointing somewhere unrelated).

## Validating the fix

```bash
git clone <this repo>
cd playground-knip-load-package-manifest-bug
git checkout test-fix
yarn install   # materializes the patched knip into node_modules
```

### Symptom A — script binary (`cowsay` in `packages/foo`)

```bash
cd packages/foo
yarn knip --include dependencies
# Expected: exit 0, no output (was: "Unused devDependencies (1) cowsay")
```

### Symptom B — peer dependency (`chai` in `packages/bar`)

```bash
cd packages/bar
yarn knip --include dependencies
# Expected: exit 0, no output (was: "Unused devDependencies (1) chai")
```

### Sanity check — monorepo mode still works

```bash
cd ../..
yarn knip --workspace packages/foo --include dependencies   # exit 0
yarn knip --workspace packages/bar --include dependencies   # exit 0
```
