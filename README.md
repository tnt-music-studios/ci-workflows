This repository contains reusable GitHub Actions workflows used by
multiple C++ projects in this organization. The workflows provide a
consistent, centralized CI setup for building projects with CMake and
the Conan package manager across Linux, macOS, and Windows.

## Products

`cpp-cmake-conan-build.yml` configures and builds one tree per platform, and its `products` input
names what that tree packages and releases. Each entry gives a product's `cpack-config` relative to
the build tree `build/<configure-preset>` and the `tag-prefix` that identifies it, naming both its
release tags and its folder of the platform's packages artifact. Every product is packaged on each
platform; a master push refreshes every product's
`<tag-prefix>dev-build` release, and a `<tag-prefix>v<SemVer>` tag releases the one product it
names. The default is the single package of a tree whose top-level project packages:
`CPackConfig.cmake` at the tree's root, released on `v<SemVer>` tags and at `dev-build`. The
workflow states each input's full contract where it declares it.

A repository with two products, each packaged by its own project below a top-level project that
packages nothing (PackageBuilder 4.0.0 or later), declares both in every platform's caller and
triggers on both tag patterns:

```yaml
name: Build - Windows

on:
  push:
    branches: [master]
    tags: ['app-v*', 'tool-v*']

concurrency:
  group: ${{ github.workflow }}-${{ github.event_name }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  windows:
    uses: tnt-music-studios/ci-workflows/.github/workflows/cpp-cmake-conan-build.yml@v<version>
    with:
      os: windows-latest
      package: true
      release: true
      products: >-
        [
          {"cpack-config": "app/CPackConfig.cmake", "tag-prefix": "app-"},
          {"cpack-config": "tool/CPackConfig.cmake", "tag-prefix": "tool-"}
        ]
    permissions:
      contents: write
```

An `app-v1.0.0` tag then fills the draft release `app-v1.0.0` with the app's installers alone.

## Compiler cache

The build job of `cpp-cmake-conan-build.yml` compiles through ccache, keeping one entry per platform
in the caller's GitHub cache, and prints its hit rate in the `Report compiler cache` step. Link time
is not cached, so a project built with link-time optimization still pays for every link.

GitHub lets a run restore entries from its own ref and the default branch only, and removes an entry
after 7 days without access. A tag run therefore starts warm only from a default-branch run within
the last 7 days, so dispatch one on each platform before tagging. Tag runs save nothing, since no
other ref could restore it; every other run saves, even when its build fails.

## Self-hosted runs

`cpp-cmake-conan-build.yml` runs on hosted images unless its `runner` input names a self-hosted
runner label. That label then takes the pre-commit gate, the build, the lint and the test-package
jobs, so a whole run needs no hosted minutes, and `os` still names the platform. The release job
stays on hosted Linux, since it runs only on push events. A caller makes this an on-demand choice
with a dispatch input, so hosted stays the default and a tag push always builds hosted:

```yaml
on:
  workflow_dispatch:
    inputs:
      local:
        description: Run on the self-hosted runner
        type: boolean
        default: false

jobs:
  windows:
    uses: tnt-music-studios/ci-workflows/.github/workflows/cpp-cmake-conan-build.yml@v<version>
    with:
      os: windows-latest
      runner: ${{ inputs.local && 'my-windows-runner' || '' }}
```

The machine must carry what the hosted image would have supplied: on Windows, Visual Studio with
the C++ workload, Git (its `bin` folder ahead of WSL's `bash`), PowerShell 7, Chocolatey, jq,
Graphviz through Chocolatey, NSIS, Doxygen, and the Python that `setup-python` installs, already
in the runner's tool cache, since installing it there needs administrator rights.

## Releases

### 2.0.0

`cpp-cmake-conan-build.yml` packages and releases several products from one build tree, declared by
the `products` input. Upgrading from 1.x, where a caller passing neither `products` nor the removed
`tag-prefix` is affected only by the last two of these changes:

- `tag-prefix` is removed. A caller that passed it declares a `products` entry instead, with that
  prefix and the product's `cpack-config`.
- The build job checks `package` against each product's CPack configuration in both directions,
  naming the path. Under `package: true` a missing configuration fails; 1.x skipped packaging
  silently, and only the release job noticed. Under `package: false` an existing configuration
  fails; 1.x packaged it and failed only a release push's release job.
- A tag push whose tag is no product's `<tag-prefix>v<SemVer>`, such as `v1.2`, fails the release
  job. 1.x published nothing and passed.
