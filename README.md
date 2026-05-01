# Knip `loadPackageManifest` bug — preview-release fix verification

This branch verifies that the preview release published by the knip
maintainer (`https://pkg.pr.new/knip@18a917e`, version `6.9.0`) fixes the
`loadPackageManifest` bug described in
[webpro-nl/knip#1711](https://github.com/webpro-nl/knip/issues/1711).

For full background on the bug — root cause, the two symptoms, and why it
only surfaces in single-project mode within a hoisting monorepo — see the
README on the [`main`](https://github.com/astegmaier/playground-knip-load-package-manifest-bug/tree/main)
branch.

## What changed on this branch

`knip` was bumped from `6.7.0` to the preview release at all three places it
was declared (root + both workspaces):

```jsonc
"knip": "https://pkg.pr.new/knip@18a917e"
```

## How to verify

```bash
git clone <this repo>
cd playground-knip-load-package-manifest-bug
git checkout private-release-fix
yarn install
```

### Symptom A — script binaries (`packages/foo`)

```bash
cd packages/foo
yarn knip --include dependencies
# → exit 0, no output  (previously: "Unused devDependencies (1)  cowsay …")
```

### Symptom B — peer dependencies (`packages/bar`)

```bash
cd packages/bar
yarn knip --include dependencies
# → exit 0, no output  (previously: "Unused devDependencies (1)  chai …")
```

### Monorepo-mode regression check

```bash
cd ../..
yarn knip --workspace packages/foo --include dependencies   # exit 0
yarn knip --workspace packages/bar --include dependencies   # exit 0
```

Both symptoms are gone, and monorepo-mode runs continue to pass.
