---
layout: default
title: Creating Rust Github Workflow
parent: Miscellaneous Tips and Tricks
nav_order: 3
---

# Creating Rust Github Workflow

To create a workflow in Github create a directory in your Rust project _.github/workflows_ and
then create a YAML file there for each workflow.


## Automatically run All Tests on Push

To have Github automatically verify committed code is healthy I created the following workflow file 
in _.github/workflows/test_and_build.yml._ for my RUST projects.

The YAML file is quite self-explanatory. It triggers on Git Push and Pull Requests and
checks out the code and runs _cargo build_ and then _cargo test_.

```
name: Test and Build

on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: cargo build --verbose
      - name: Run tests
        run: cargo test --verbose
```

## Creating Automatic Release Builds

To automatically generate release builds when tagging a release I found the following workflow description
useful. It builds and generates archives for Windows, Linux and Mac OS automatically and packages them
with the README and License file. It uses the github action script [rust-build/rust-build.action](https://github.com/rust-build/rust-build.action). 

Just create a file in _.github/worflows/release.yml_ with the following content.

```
name: Release

on:
  release:
    types: [created]

jobs:
  release:
    name: release ${{ matrix.target }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - target: x86_64-pc-windows-gnu
            archive: zip
          - target: x86_64-unknown-linux-musl
            archive: tar.gz tar.xz
          - target: x86_64-apple-darwin
            archive: zip
    steps:
      - uses: actions/checkout@master
      - name: Compile and release
        uses: rust-build/rust-build.action@v1.3.2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          RUSTTARGET: ${{ matrix.target }}
          ARCHIVE_TYPES: ${{ matrix.archive }}
          EXTRA_FILES: "README.md LICENSE"
```