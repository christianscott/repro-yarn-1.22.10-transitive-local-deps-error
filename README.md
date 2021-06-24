# Reproducing errors with local transitive dependencies with yarn 1.22.10

> [Link to issue in yarnpkgs/yarn](https://github.com/yarnpkg/yarn/issues/8653)

I can't manage to make a chain of local packages depending on one another work in yarn 1.22.10. First I tried installing the packages into one another via `yarn add ../path/to/package`, but that leads to an error similar to the following:

```
$ yarn add ../nested/two
yarn add v1.22.5
info No lockfile found.
[1/4] 🔍  Resolving packages...
error Package "three" refers to a non-existing file '"/code/repro-yarn-1.22.10-transitive-local-deps-error/packages_plain_paths/three"'.
info Visit https://yarnpkg.com/en/docs/cli/add for documentation about this command.
```

After that I tried installing using the "file" prefix, e.g. `yarn add ../path/to/package`. This doesn't fail but produces an invalid lock file:

```
"two@file:../nested/two":
  version "1.0.0"
  dependencies:
    three "file:../../../../../../Library/Caches/Yarn/v6/npm-two-1.0.0-6b33cef9-3d3e-4e34-8187-895d54f9451d-1624546610982/node_modules/three"
```

I would expect this to be something like this instead:

```
"two@file:../nested/two":
  version "1.0.0"
  dependencies:
    three "file:../nested/three"
```

## Reproducing

This repo reproduces both of the errors outlined above. It contains three packages, structured as follows:

```
packages/
├── nested
│   ├── three
│   │   ├── index.js
│   │   └── package.json
│   └── two
│       ├── package.json
│       └── yarn.lock
└── one
    └── package.json

4 directories, 5 files
```

With the following dependency graph:

```
nested/three <- nested/two <- one
```

To reproduce the second issue, run the following commands:

```sh
cd packages_plain_paths/one
yarn add ../nested/two
```

I believe the nesting is essential – my guess is that yarn tries to resolve `three` using the relative path that `two` uses. Since they're not in the same directory, the package cannot be found. This snippet looks a little suspicious: https://github.com/yarnpkg/yarn/blob/master/src/resolvers/exotics/file-resolver.js#L35-L37
