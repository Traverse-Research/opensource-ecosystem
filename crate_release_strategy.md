# How we release open-source crates at Traverse Research

How we make it straightforward and accessible to publish new crate releases from our repositories.

### The local process

We use [cargo-release](https://github.com/crate-ci/cargo-release) to help us bump versions in various places, create the "Release {{version}}" commit and tag and push them upstream. This process happens locally on a developer machine and requires push access to GitHub, but no access to a https://crates.io token/account. Publishing is disabled locally and happens on the CI, which is made possible by this `cargo-release` config:

`release.toml`:
```toml
pre-release-commit-message = "Release {{version}}"
tag-message = "Release {{version}}"
tag-name = "{{version}}"
sign-commit = true
sign-tag = true
publish = false

pre-release-replacements = [
  {file="README.md", search="your_crate_name = .*", replace="{{crate_name}} = \"{{version}}\""},
]
```

If your crate consists of a `[workspace]` with multiple crates, but you only wish to publish the root crate or keep all crate versions in sync, add the following configuration to make sure `cargo-release` substitutes `{{version}}` in the templates above:

```toml
# cargo-release only allows using {{version}} in the commit title when creating one
# commit across all released packages in this workspace (we only release one package
# though), or by using the same version for all packages.
# https://github.com/crate-ci/cargo-release/issues/540#issuecomment-1328769105
# https://github.com/crate-ci/cargo-release/commit/3af94caa4b9bbee010a5cf3f196cc4afffbaf192
consolidate-commits = false
shared-version = true
```

With that, a developer performs a release on their local machine with:

```console
$ cargo install cargo-release
$ cargo release "<next version>"
```

(Only add `-x` if you are okay with the changes!)

#### Creating a release on GitHub

While [the CI process](###the-automated-ci-process) runs, click `Draft a new release` on the `releases` page on GitHub. Select the tag matching the version just pushed by `cargo-release` above, and click `Generate release notes` to automatically populate the title and description. Modify both as necessary. In particular, release notes convey user-facing changes: remove internal changes such as clippy fixes and other cleanups. When breaking changes are made or new features are added, spend an extra paragraph at the top detailing how users should migrate to or make use of the new release 🥳!

### The automated CI process

This process is complemented by a CI workflow to perform the actual `cargo publish` to https://crates.io, and makes it so that not every developer needs to have direct access to the crates account nor a token for it. This workflow is triggered by tag pushes only (but could also be invoked on every push to `main`, which is a no-op if the version wasn't bumped) and simply performs the publishing step using a secret token stored with GitHub:

`.github/workflows/publish.yaml`:
```yaml
name: Publish

on:
  push:
    tags:
    paths: "/Cargo.toml"

jobs:
  Publish:
    if: github.repository_owner == 'Traverse-Research'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          override: true
      - name: Publish
        uses: actions-rs/cargo@v1
        with:
          command: publish
          args: --token ${{ secrets.cratesio_token }}
```

### Future improvements

The `cargo-release` process requires push access to `main` and push access for tags. Over time it'd be great if a `cargo-release` result (together with perhaps any other relevant changes) can be PR'd, and automatically released (+ pushing a new tag) directly frm the CI as soon as it is merged.
