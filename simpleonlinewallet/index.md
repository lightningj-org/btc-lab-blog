---
layout: default
title: Simple Online Wallet
nav_order: 2
has_children: true
---

# Simple wallet into

My first project in learning RUST/Bitcoin/BDK development was creating a simple wallet
that have the following features:

* Create wallet with BIP 39 seed phrase
* Recover a wallet from seed phrases
* Generate a new address (in bech32 address format).
* Show current balance
* List transactions in a table
* Create and send a transaction.


The wallet is a light wallet that uses electrum API that by default operates on Testnet. Main-net
is supported by newer tested. 

*IMPORTANT* This is only for fun test project and never intended for any real funds.

First part of the guide is how to get starting with the project with a few tips in using Cargo. Then
some overview of using the wallet before going through some tips and tricks on developing the functionality.