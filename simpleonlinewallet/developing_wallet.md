---
layout: default
title: Developing the Wallet
parent: Simple Online Wallet
nav_order: 4
---

# Developing the Wallet

In this section I will go through all tips and tricks I learnt developing this simple wallet and hopefully
helps you to get started. Using the BDK library didn't always feel straight forward, at least if you
are not to familiar will all concept of Bitcoin terminology.

## Overall Structure of the Project

The structure of the project follows the RUST conventions with all source code in _src_ directory and 
integration tests in _tests_ directory.

The _src_ directory contains the main.rs file which contains the main function and definitions for
parsing CLI arguments using the crate [clap](https://crates.io/crates/clap), that helps a lot with sorting out the 
options. Then there are the submodules _src/core_ containing helper functions and structures in common between
commands and _src/cmd_ where each command is separated into it own rust source file.

### Automated Testing

Most of the code is tested either by using unit test, in the end of each module file or by running
the integration tests. The integration tests executes a CLI command with options and verifies that
the output to standard out and output file is correct. To help verifying integration test output I used the crate
[assert_cmd](https://crates.io/crates/assert_cmd).

To execute all tests use the command:

    cargo test

## Creating a Wallet

TODO

## Importing a Wallet

TODO

## Displaying the Balance

TODO

## Generating Addresses

TODO

## Listing Transactions

TODO

## Creating and Sending Funds

TODO
