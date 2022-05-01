---
layout: default
title: Using the Wallet CLI Tool
parent: Simple Online Wallet
nav_order: 3
---

# Using the Wallet CLI Tool

This section gives a short usage guide of the tool to given an overview how the tool works
before describing the implementation details for each command.

After building the tool it is possible get at list of available commands by running the command
without any option (or with 'help'). 

    ./btc-tool help

    btc-tool 0.1.0

    USAGE:
    btc-tool <SUBCOMMAND>

    OPTIONS:
    -h, --help       Print help information
    -V, --version    Print version information

    SUBCOMMANDS:
    create               Create new BTC wallet
    get-balance          Get Current Balance of Wallet
    help                 Print this message or the help of the given subcommand(s)
    import               Import existing wallet from Seed Phrases
    list-transactions    List all transactions using wallet
    new-address          Get Current Balance of Wallet
    send                 Sends funds to specified address

To handle CLI options I used the library clap
(https://docs.rs/clap/latest/clap/) which is quite powerful to handle all command line functionalities.

## Creating a Wallet

The first step is to create a new wallet. This is done with the _create_ command.

    Create new BTC wallet

    USAGE:
    btc-tool create [OPTIONS]

    OPTIONS:
    -c, --chain <CHAIN>    Target Chain of Wallet [default: testnet] [possible values: testnet,
    mainnet]
    -h, --help             Print help information
    -n, --name <NAME>      The name of the wallet [default: default]

Sample output of the command:

    ./btc-tool create
    You are about to generate a new wallet with name default

    First select a password to protect the wallet.
    It is *VERY IMPORTANT* to remember this password in order recreate this wallet later
    using the seed phrases.
    Enter Password :
    Verify Password:
    New Seed generated:
    +-----------+-------------+----------+------------+------------+-----------+
    |  1: bonus |  2: supreme |  3: race |  4: ugly   |  5: tragic |  6: arrow |
    +-----------+-------------+----------+------------+------------+-----------+
    |  7: wrist |  8: raccoon |  9: lock | 10: foster | 11: doll   | 12: pair  |
    +-----------+-------------+----------+------------+------------+-----------+

    Note down seed phrase and keep it somewhere safe.
    Wallet created and stored in ~/.btc-tool/default.wallet

If no wallet name is specified is the 'default' wallet created. It is possible to have
multiple different wallets, but in that case its name have to be specified as a parameter for
each subsequent command when using that wallet.

The tool generates the seed phrases and lets the user enter a 13:th passphrase used with the
seed. This passphrase or password is used both to encrypt the wallet data file and to protect
the seed values. This means that this password is required when restoring the wallet. 

Many other Bip39 wallets use an empty password as 13:th passphrase in order to always let 
the user restore the wallet using the 12 seed phrases only. And for some wallets this could
be a better approach, and I might change this in this test wallet in the future.

It is also possible to select which chain that should be used with the wallet, testnet of mainnet.
But since this tool is not intended to be used in production with any real funds, just to elaborate
with developing Bitcoin tools using RUST it is _strongly_ recommended to use the default testnet chain
only.

The source code for the _create_ command can be found in src/cmd/nowallet/createwalletcmd.rs.

## Configurating the Wallet

The first time the tool is used it will create a configuration file in ~/.btc-tool/btc-tool.yml

The configuration file contains the following data:

    #Configuration for btc tool

    #if debug mode should be used for more verbose output
    debug: false

    #The Electrum Connect URL to connect to.
    electrum_url: ssl://electrum.blockstream.info:60002

The wallet uses the Electum API to synchronize wallet data and it is possible to specify
your own Electrum server by changing the _electrum_url_ setting.

The directory _HOME_DIR/.btc-tool_ is also the directory where all encrypted wallet and 
blockchain syncronization database files are located.

To override the default directory for storage if files set the environment variable BTC_TOOL_HOME
before running the command.

    export BTC_TOOL_HOME=/tmp/somelocation
    ./btc-tool create

### Avoid entering the password

When developing and testing it might be annoying to enter the password every command. To avoid
this set the environment variable BTC_TOOL_PWD. This is obviously not safe and only when testing.

    export BTC_TOOL_PWD=foo123
    ./btc-tool create

## Restoring a Wallet

To restore a wallet from its seeds phrases use the _import_ command. It lets the user
enter the 12 seed phrases, (checking that each word phrase is in the english BIP 39 word list.)

    Import existing wallet from Seed Phrases

    USAGE:
    btc-tool import [OPTIONS]

    OPTIONS:
    -c, --chain <CHAIN>    Target Chain of Wallet [default: testnet] [possible values: testnet,
    mainnet]
    -h, --help             Print help information
    -n, --name <NAME>      The name to wallet to import from seed [default: default]

Sample output of the tool:

     ./btc-tool import  -n import_wallet1
    You are about to recreate a new wallet with name import_wallet1.
    The wallet will be recreated with your seed phrases in combination
    with your wallet password.
    
    Enter your seed phrases (Use Ctrl-C to abort):
    Enter word 1:
    bonus
    Enter word 2:
    supreme
    Enter word 3:
    race
    Enter word 4:
    ugly
    Enter word 5:
    tragic
    Enter word 6:
    arrow
    Enter word 7:
    wrist
    Enter word 8:
    raccoon
    Enter word 9:
    lock
    Enter word 10:
    foster
    Enter word 11:
    doll
    Enter word 12:
    pair
    
    You have entered the following seed phrases:
    +-----------+-------------+----------+------------+------------+-----------+
    |  1: bonus |  2: supreme |  3: race |  4: ugly   |  5: tragic |  6: arrow |
    +-----------+-------------+----------+------------+------------+-----------+
    |  7: wrist |  8: raccoon |  9: lock | 10: foster | 11: doll   | 12: pair  |
    +-----------+-------------+----------+------------+------------+-----------+
    Is this correct? (yes,no):
    yes
    Enter Password :
    Verify Password:
    Wallet created and stored in ~/.btc-tool/import_wallet1.wallet

The source code for the _create_ command can be found in src/cmd/nowallet/importwalletcmd.rs.

## Check Balance

To check the current balance of your wallet, use the _get-balance_ command. It synchronizes the wallet using
electrum and displays the current amount in SAT.

    ./btc-tool get-balance 
    Enter Password
    Synchronizing Blockchain...
    Sync Complete.

    Current balance: 4859 SAT

## Generate New Address

To generate a new address that can be used to send funds to the wallet use the _new-address_ command.

    ./btc-tool new-address
    Enter Password
    New address: tb1qq775n5ummkxdxzhw2ayxqfv8jalpwvxj28tzag

It generates a new Bech32 Segwit address specified in BIP 173.

## List Transactions

To list all transactions performed by a wallet use the command list-transactions.

Example output:

    ./btc-tool list-transactions
    Enter Password
    Synchronizing Blockchain...
    Sync Complete.
    
    +------------------------------------------------------------------+-------+----------+-----+--------------------+
    | TransactionId                                                    | Sent  | Received | Fee | Confirmation Block |
    +------------------------------------------------------------------+-------+----------+-----+--------------------+
    | efcee94876e3aa5e18049efcfdbb2c370c54a72f12ea797ff1981d556c7b48b2 | 10000 | 4859     | 141 | 2196329            |
    +------------------------------------------------------------------+-------+----------+-----+--------------------+
    | b97ca0dd8359fe56f41a7cf417a1a1e48ddbe392694ec0009ea93161616645f3 | 0     | 10000    | 141 | 2196328            |
    +------------------------------------------------------------------+-------+----------+-----+--------------------+

## Send Transaction

To send funds to another wallet use the _send_ command. The command takes two parameters (with an optional third fee 
parameter), the amount in satoshis and the address to send to.

    ./btc-tool send -h
    btc-tool-send
    Sends funds to specified address
    
    USAGE:
    btc-tool send [OPTIONS] --address <ADDRESS> --amount <AMOUNT>
    
    OPTIONS:
    -a, --address <ADDRESS>    Address to send to
    -f, --fee <FEE>            Optional fee in sats/vbyte [default: 0.0]
    -h, --help                 Print help information
    -n, --name <NAME>          The name of the wallet [default: default]
    -s, --amount <AMOUNT>      Amount of to send in satoshis

Example usage:

    ./btc-tool send -a tb1ql7w62elx9ucw4pj5lgw4l028hmuw80sndtntxt -s 4000
    Enter Password
    Synchronizing Blockchain...
    Sync Complete.
    
    +------------------------------------------------------------------+------+----------+-----+--------------------+
    | TransactionId                                                    | Sent | Received | Fee | Confirmation Block |
    +------------------------------------------------------------------+------+----------+-----+--------------------+
    | 3a49ad5b5d1f5a2a25f9363bdadd541961ff548afde779348c3563dcd74d5c7c | 4859 | 718      | 141 | None               |
    +------------------------------------------------------------------+------+----------+-----+--------------------+
    Transaction sent to Network.
    Explorer URL: https://blockstream.info/testnet/tx/3a49ad5b5d1f5a2a25f9363bdadd541961ff548afde779348c3563dcd74d5c7c

After the transaction have been sent it is possible to copy and paste the link to check that the transaction have been 
confirmed.