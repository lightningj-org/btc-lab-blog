---
layout: default
title: Setup Rust Project
parent: Simple Online Wallet
nav_order: 2
---

# Setup the simple online wallet project

Here is a brief introduction on how to start using the project if you want to test it out yourself.

Prerequisites are installed Git and some knowledge in the terminal. These instructions have
been tested on MacOS Monterey. 

Before starting with rust you need to install Rust and its package manager Cargo by following this
guide https://doc.rust-lang.org/cargo/getting-started/installation.html

Then fetch the source code with:

  git clone git@github.com:lightningj-org/btc-tool.git
  
Then to get started quickly build and run the client (to list available commands) with:

    cargo run -- help 
  
To just build a debug version use the command, (add --release for release version):

    cargo build

Then the generated binary is located in _target/debug/btc-tool_.

To run all tests (both unit and integration tests) run:

    cargo test

To generate and read API documentation use:

    cargo doc
        
    cd target/doc/btc_tool/
    open index.html

As IDE I have used Intellij Community Edition with Rust plugin and Visual Studio Code.