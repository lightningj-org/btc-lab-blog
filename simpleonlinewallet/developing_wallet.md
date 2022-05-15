---
layout: default
title: Developing the Wallet
parent: Simple Online Wallet
nav_order: 4
---

# Developing the Wallet

In this section I will go through all tips and tricks I learnt developing this simple wallet and hopefully
helps you to get started. Using the BDK library didn't always feel straight forward, at least if you
are not too familiar will all Bitcoin terminology.

## Overall Structure of the Project

The structure of the project follows the RUST conventions with all source code in _src_ directory and 
integration tests in _tests_ directory.

The _src_ directory contains the main.rs file, which contains the main function and definitions for
parsing CLI arguments using the crate [clap](https://crates.io/crates/clap). It helps a lot with sorting out the 
options when using the tool. 

Then there are the submodules _src/core_ containing helper functions and structures in common between
commands and _src/cmd_ where each command is separated into it own rust source file.

### Automated Testing

Most of the code is tested either by using unit test, in the end of each module file or by running
the integration tests. The integration tests executes a CLI command with options and verifies that
the output to standard out and output file is correct. To help verifying integration test output I used the crate
[assert_cmd](https://crates.io/crates/assert_cmd).

To execute all tests use the command:

    cargo test

## Creating a Wallet

The hardest part when writing a bitcoin wallet software using BDK was creating the actual wallet.

Before doing so I had to dig into how Wallet Descriptors and Derivation Paths worked before being
able to generate the wallet. So I will give some tips and tricks on creating both
descriptors and derivation path before going through the actual code of creating a wallet.

### Wallet Descriptors

The first thing to understand when creating the wallet using BDK is the figuring out how Wallet Descriptors work. BDK
uses a very flexible way of defining the Bitcoin scripts created for spending output and addresses and provides an 
extended set of Mini-script for encoding the generated bitcoin script language. BDK have a description of
its extended mini-script support at its [descriptor page](https://bitcoindevkit.org/descriptors/).

In the simple wallet I choose to use the simple mini-script _wpkh(ext_key)_ which stands for Pay-to-Witness-Public-Key-Hash
(P2WPKH) which is used to create addresses using Segwit native addresses, i.e. generate addresses that start with _bc1_ 
or _tb1_ for testnet. 

But with BDKs flexible design it is possible to create more advanced wallet structures by
using a different mini-script.  Reading more about mini-scripts and also compiling and analyzing can be done with the 
[mini script website](https://bitcoin.sipa.be/miniscript/).


### Using Derivation Paths

The next thing to understand is how DerivationPaths work. BDK uses a Hierarchical Deterministic (HD) wallet used to derive 
a specific key within a tree of keys that was defined in [BIP 32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki). 

When creating a wallet you need at least to define one, but recommended two, derivation path strings. The first is an 
external one, used to generate external addresses while the other one is for internal addresses, such as change 
addresses etc. The simple wallet uses [BIP 84](https://github.com/bitcoin/bips/blob/master/bip-0084.mediawiki) 
descriptors by default. BIP 84 has the following structure:

    m / purpose' / coin_type' / account' / change / address_index

So the descriptor used for external usage becomes:

    m/84'/1'/0'/0/0

Which means descriptor is according to _m/ BIP 84 / Bitcoin Testnet / Account 0 / external chain / first key by index_

The descriptor for internal usage then becomes:

    m/84'/1'/0'/1/0

The code I used to calculate derivation paths look like the code below, the method _get_derivation_path_
checks the parameter external and type of chain to generate the expected str value:

```
/// Help method to get the coin type in the derivation path.
fn get_coin_type(chain : &Chain) -> &str {
    match chain {
        Chain::Testnet=> "1'",
        Chain::Mainnet=> "0'",
    }
}

/// Help method to get the change type in the derivation path
fn get_change(external : bool) ->  &'static str{
    if external {
        return "0";
    }
    return "1";
}

/// Method to calculate derivation path for given type (external or internal) and chain.
fn get_derivation_path(external : bool, chain : &Chain) -> DerivationPath{
    return DerivationPath::from_str(format!("m/84'/{}/0'/{}/0",
                                     get_coin_type(chain),
                                     get_change(external)).as_str()).unwrap();
}
```

Some details on BDKs support for Wallet Descriptors can be found [here](https://bitcoindevkit.org/descriptors/)

### Create Wallet Code

So after figuring out the Descriptor used and descriptor paths the code the create wallet code
can be created.

Create Wallet command first prompts for the password used as 13:th seed word and key for the
encrypted wallet data. Then it generates Mnemonic seed words in English. Then calls the method
_create_online_wallet()_ (explained below) which is used by both the create and import wallet commands.

```
    fn execute(self : &Self) -> Result<(), Box<dyn Error>>{
        let app_dir = get_or_create_app_dir()?;
        if wallet_exists(&self.name)? {
          return Err(into_err(format!("Error wallet {} already exists, remove files {}{} and {}{} in directory {}.",
                                      &self.name,&self.name,WALLET_DATA_POSTFIX,
                                      &self.name,WALLET_DB_POSTFIX, app_dir.to_str().unwrap())));
        }
        println!("You are about to generate a new wallet with name {}.",&self.name);
        println!();
        println!("First select a password to protect the wallet.");
        println!("It is *VERY IMPORTANT* to remember this password in order recreate this wallet later");
        println!("using the seed phrases.");

        let password = read_verified_password()?;

        println!("New Seed generated:");
        let mut rng = rand::thread_rng();
        let mnemonic = Mnemonic::generate_in_with(&mut rng,
                                                  Language::English,
                                                  12)
            .map_err(|_| bdk::Error::Generic("Mnemonic generation error".to_string()))?;
        let words : Vec<&'static str> = mnemonic.word_iter().collect();

        print_stdout(gen_seed_word_table(&words))?;

        println!("\nNote down seed phrase and keep it somewhere safe.");

        create_online_wallet(&self.name, &self.chain,
                             mnemonic, password,
                             &self.settings , &app_dir)?;
        return Ok(());
    }
```

The code for actually creating the wallet, first converts the mnemonic to a seed value. Then generates the root key and
external derivation path to create the external key.

Then the root key is used with the internal derivation path to create the internal key. The trick when generating
these descriptors was to use the to_string_with_secret in order to store the private key and not just the public data.

Finally, the wallet is created using the BDK _Wallet::new_ function and is stored encrypted to file
using the projects WalletData struct.

```
/// Help method to create an online wallet that is in common for create and import commands.
pub(crate) fn create_online_wallet(name : &String, chain : &Chain,
                                   mnemonic : Mnemonic, password : String,
                                   settings : &Settings, app_dir : &PathBuf) -> Result<(), Box<dyn Error>>{
    let network = get_chain_name(chain);
    let ext_path = get_derivation_path(true, chain);
    let seed = mnemonic.to_seed(&password);
    let root_key = ExtendedPrivKey::new_master(network, &seed).unwrap();

    let ext_key = (root_key, ext_path);
    let (ext_descriptor, ext_key_map, _) = bdk::descriptor!(wpkh(ext_key)).unwrap();

    let ext_descriptor_with_secret = ext_descriptor.to_string_with_secret(&ext_key_map);

    let int_path = get_derivation_path(false, chain);
    let int_key = (root_key, int_path);
    let (int_descriptor, int_key_map, _) = bdk::descriptor!(wpkh(int_key)).unwrap();
    let int_descriptor_with_secret = int_descriptor.to_string_with_secret(&int_key_map);
    let mut wallet_db_path = app_dir.clone();
    wallet_db_path.push(format!("{}.db",name));

    let database = settings.get_wallet_database(name)?;

    let wallet: Wallet<AnyDatabase> = Wallet::new(
        &ext_descriptor_with_secret,
        Some(&int_descriptor_with_secret),
        network,
        database
    )?;

    let wallet_data = WalletData::new(name,&wallet,
                                      &ext_descriptor_with_secret,
                                      &int_descriptor_with_secret,
                                      &root_key.private_key);
    let _ = wallet_data.save(&password)?;
    let wallet_path = get_wallet_path(name)?;
    println!("Wallet created and stored in {}", wallet_path.to_str().unwrap());
    return Ok(())
}
```

The code for creating a wallet can be found in _src/cmd/nowallet/mod.rs_ and _src/cmd/nowallet/createwalletcmd.rs_.

### Encrypting the Wallet Data

A wallet produces two files, one database file that is created and maintained by the Sled Database.

The other one is created by the wallet itself. It is a simple data structure WalletData that is
serialized to YAML using [serde_yaml](https://crates.io/crates/serde_yaml) crate, This is done by adding the 
Serialize and Deserialize annotation on the struct.

```
#[derive(Debug, Serialize, Deserialize)]
pub struct WalletData {
// Name of Wallet
pub name: String,
// Private Key in WIF String format
pub xpriv: String,
// External Descriptor
pub external_descriptor: String,
// Internal Descriptor
pub internal_descriptor: String,
// The related network
pub network : Network,
// IF wallet is an offline or online wallet
pub online: bool,
}
```

The WalletData implementation contains functions for loading and storing to file but since you would
want this sensitive information encrypted I also added encrypt and decrypt code.

The encryption key is the same password used as the 13th seed phrase to keep it simple. The encryption
then performs a PBKDF2 derivation function to generate a symmetric key used to encrypt the data using AES256-GCM.

```
/// Help method encrypt serialized wallet data with given password.
fn encrypt(data : String, password : &String) -> Result<Vec<u8>,Box<dyn std::error::Error>>{

    let salt = SaltString::generate(&mut OsRng);
    let gen_key = Pbkdf2.hash_password(password.as_bytes(), &salt).map_err(|err| into_err(format!("Error generating wallet encryption key from password: {}",err)))?;
    let key = gen_key.hash.unwrap();

    let mut nonce_data : [u8 ; 12] = [0;12];
    OsRng.fill_bytes( &mut nonce_data);

    let cipher = Aes256Gcm::new(Key::from_slice(key.as_bytes()));
    let nonce = Nonce::from_slice(&nonce_data);

    let ciphertext = cipher.encrypt(nonce, data.as_bytes()).map_err(|err| into_err(format!("Error encrypting wallet data: {}",err)))?;

    let result =  [salt.as_bytes().to_vec(),
        nonce_data.to_vec(),
        ciphertext].concat();

    Ok(result)
}
```

Decryption is about the same code with minor modifications:

```
/// Help method decrypt serialized wallet data with given password.
fn decrypt(data : Vec<u8>, password : &String) -> Result<String,Box<dyn std::error::Error>>{
    if data.len() < 35 {
        return Err(new_err("Invalid length of encrypted data"));
    }
    let salt_string = str::from_utf8(&data[0..22])?;
    let salt = Salt::new(salt_string).map_err(|err| into_err(format!("Error generating wallet encryption key from password: {}",err)))?;

    let hash = Pbkdf2.hash_password(password.as_bytes(), &salt).map_err(|err| into_err(format!("Error generating wallet decryption key from password: {}",err)))?;
    let key = hash.hash.unwrap();

    let nonce = Nonce::from_slice(&data[22..34]);
    let cipher = Aes256Gcm::new(Key::from_slice(key.as_bytes()));
    let enc_data = &data[34..];
    let plaintext = cipher.decrypt(nonce, enc_data.as_ref()).map_err(|err| into_err(format!("Error decrypting wallet data, was password correct?: {}",err)))?;
    Ok(String::from_utf8(plaintext)?)
}
```

The source code of the WalletData is in _src/core/walletdata.rs_ file.

### Using Configurable Network and Database Settings

One think I found quite tricky was making the used Database and Blockchain (blockchain sync implementation) dynamic by
making it configurable in a settings file. I solved this by using the AnyDatabase and AnyBlockchain structs from BDK.

Here is a section of code how to create a AnyDatabase from a Sled database from the _settings.rs_.

```
    /// Method To return the configured Wallet Database to use.
    pub fn get_wallet_database(self : &Self, name : &String) -> Result<AnyDatabase, Box<dyn std::error::Error>> {
        let mut wallet_db_dir = get_or_create_app_dir().map_err(|_| ConfigError::Message("Error reading application home directory".to_string()))?;
        let wallet_db_name = format!("{}{}",name,WALLET_DB_POSTFIX);
        wallet_db_dir.push(&wallet_db_name);
        let sled_config = SledDbConfiguration{
            path: wallet_db_dir.to_str().unwrap().to_string(),
            tree_name: wallet_db_name
        };
        let sled_tree = sled::open(&sled_config.path)?.open_tree(&sled_config.tree_name)?;
        let any_database = AnyDatabase::Sled(sled_tree);
        Ok(any_database)
    }
```

Another rather complicated thing was to have the possibility to have an offline wallet and online wallet used by the
same code since they are different struct definitions. For this I created WalletContainer to abstract the this behaviour.
BDK from 0.17 have simplified this behaviour so creating this WalletContainer might no longer be necessary.

## Importing a Wallet

Importing a wallet using existing seed words is practically the same when creating a new one. The only
difference is that the user is prompted for the seed words on the console.

To read a word and check that it is a valid word according to the english wordlist I used the following
code. This method is called repeatedly 12 times.

```
fn get_word(n : i32) -> Result<String, Box<dyn Error>>{
    let mut retval;
    loop {
        println!("Enter word {}: ", n);
        let mut word = String::new();
        let _ = stdin().read_line(&mut word)?;
        retval = word.to_lowercase().trim().to_string();
        let matching_words = Language::English.words_by_prefix(retval.as_str());
        if matching_words.len() == 1 && matching_words[0].eq(retval.as_str()){
            break;
        }else{
            println!("Invalid word {} entered, try again",n)
        }
    }

    return Ok(retval);
}
```

The full code to create an imported wallet is and can be found _src/cmd/nowallet/importwalletcmd.rs_:

```
    fn execute(self : &Self) -> Result<(), Box<dyn Error>>{
        let app_dir = get_or_create_app_dir()?;
        if wallet_exists(&self.name)? {
            return Err(into_err(format!("Error wallet {} already exists, remove files {}{} and {}{} in directory {}.",
                                        &self.name,&self.name,WALLET_DATA_POSTFIX,
                                        &self.name,WALLET_DB_POSTFIX, app_dir.to_str().unwrap())));
        }

        println!("You are about to recreate a new wallet with name {}.",&self.name);
        println!("The wallet will be recreated with your seed phrases in combination");
        println!("with your wallet password.");
        println!();
        let mut words: Vec<String>;
        loop {
            println!("Enter your seed phrases (Use Ctrl-C to abort): ");

            words = vec![];
            for n in 1..13 {
                let word = get_word( n)?;
                words.push(word)
            }

            println!();
            println!("You have entered the following seed phrases:");
            print_stdout(gen_seed_word_table(&words))?;
            if get_confirmation("Is this correct? (yes,no):")? {
                break;
            }
        }
        let password = read_verified_password()?;

        let word_string = words.join(" ");
        let mnemonic = Mnemonic::from_str(word_string.as_str())?;

        create_online_wallet(&self.name, &self.chain,
                             mnemonic, password,
                             &self.settings , &app_dir)?;

        return Ok(());
    }
```

## Displaying the Balance

To display the current wallet balance is really simple. Just call _get_balance()_ to
get the number of available satoshis in the wallet. A real wallet would probably
convert it into different denominations (BTC, bits, USD etc) but that is pure calculations.

```
    fn execute(self : &Self) -> Result<(), Box<dyn Error>>{
        let (wallet, _) = get_wallet(&self.name, &self.settings)?;
        let _ = sync_wallet(&wallet)?;
        println!("Current balance: {} SAT", wallet.get_balance()?);
        Ok(())
    }
```

The code can be found in _src/cmd/wallet/getbalancecmd.rs_.

## Generating Addresses

The code for generating a new receive address using BDK is very simple.
The address type generated is depending on the descriptor string used when creating
the wallet. In this case is a Bech32 (native Segwit) generated since the wallet descriptor 
started with 'm/84'.

To generate a new address just call _get_address()_. The code can
be found at _src/cmd/wallet/newaddresscmd.rs_.

```
    fn execute(self : &Self) -> Result<(), Box<dyn Error>>{
        let (wallet, _) = get_wallet(&self.name, &self.settings)?;
        let new_address = wallet.get_address(AddressIndex::New)?;
        println!("New address: {}", new_address.to_string());
        Ok(())
    }
```

## Listing Transactions

Listing all transactions of a wallet is quite easy. Just call wallet.list_transactions()
to retrieve all transactions. Then I used the [cli-table](https://crates.io/crates/cli-table) crate to 
format the table.

In the example I just display the data in the BDK transaction struct. A real wallet would probably
recalculate some fields to make the UX a bit clearer.

The code for listing transactions i quite straight forward and can be found
at _src/cmd/wallet/listtransactionscmd.rs_:

```
    fn execute(self : &Self) -> Result<(), Box<dyn Error>>{
        let (wallet, _) = get_wallet(&self.name, &self.settings)?;
        let _ = sync_wallet(&wallet)?;
        // In future include raw transactions in list
        let transactions = wallet.list_transactions(false)?;

        let _ = print_stdout(gen_transaction_table(&transactions));

        Ok(())
    }
```

The help method to create a nice CLI table presented on the console:

```
/// Help method to generate a seed word table with justified columns.
pub fn gen_transaction_table(transactions : &Vec<TransactionDetails>) -> TableStruct {
    let mut rows : Vec<Vec<CellStruct>> = vec![];
    for transaction in transactions{
        rows.push(vec![
            transaction.txid.to_string().cell(),
            transaction.sent.to_string().cell(),
            transaction.received.to_string().cell(),
            match &transaction.fee {
                None => "None".to_string(),
                Some(fee_value) => format!("{}", fee_value)
            }.cell(),
            match &transaction.confirmation_time {
                None => "None".to_string(),
                Some(confirm_time) => format!("{}", confirm_time.height)
            }.cell(),
        ])
    }
    return rows.table().title(vec![
        "TransactionId".cell().bold(true),
        "Sent".cell().bold(true),
        "Received".cell().bold(true),
        "Fee".cell().bold(true),
        "Confirmation Block".cell().bold(true),
    ])
}
```

## Sending Funds

To create and send a transaction is done in four steps, after synchronizing the wallet.

First step is to parse and validate the address. This is done with:
```
let address = Address::from_str(self.to_address.as_str()).map_err(|e| into_err(format!("Invalid address specified: {}",e)))?;
```

In the second step, I build up a transaction with the TxBuilder. I set the recipient address and amount (in SATs) and enable Replace-By-Fee flag.
Then I check if a custom fee was set as CLI argument and in that case use that as SAT per Virtual Byte.

Finally, I finish the transaction and generates an ASCII table out put to the user.

```
        let mut tx_builder = online_wallet.build_tx();
        tx_builder
            .add_recipient(address.script_pubkey(), self.amount)
            .enable_rbf();
        if self.fee != 0.0{
            tx_builder.fee_rate(FeeRate::from_sat_per_vb(self.fee));
        }
        let (mut psbt, tx_details) = tx_builder.finish()?;

        let _ = print_stdout(gen_transaction_table(&vec![tx_details]));
```

Third and fourth I sign the transaction with the wallet Signer and broadcasts
the raw transaction to the network with:

```
        let finalized = online_wallet.sign(&mut psbt, SignOptions::default())?;
        if finalized {
            let raw_transaction = psbt.extract_tx();
            blockchain.broadcast(&raw_transaction)?;
            let txid = &raw_transaction.txid();
            println!(
                "Transaction sent to Network.\nExplorer URL: https://blockstream.info/testnet/tx/{txid}",
                txid = txid
            );
        }else{
            println!("Transaction could not be signed.")
        }
```

The code can be found at _src/cmd/wallet/sendcmd.rs_.