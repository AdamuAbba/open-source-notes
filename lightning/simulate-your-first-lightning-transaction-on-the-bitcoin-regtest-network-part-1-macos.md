# Objective


The goal of this article is to help you

- configure the Bitcoin environment
- create a Bitcoin wallet
- create an address, give it a label
- mine some blocks to that address
- configure two Lightening nodes

# Introduction

When I recently started my journey in Bitcoin development, I felt so overwhelmed by the resources online mainly because I found out that certain information where either left out or now deprecated. so I was inspired to write my take on this topic because it confused me the most.

# Requirements

To be able to follow along, you must have both [bitcoind](https://github.com/bitcoin/bitcoin) and [lnd](https://github.com/lightningnetwork/lnd) installed.

Below are two articles that helped me through this topic's installation, setup, and research phases.

Their work heavily inspires this article and has made use of references appropriately.

| Author                                                                                                              | Article                                                                                                                                                                              | Study flow   |
| ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ |
| [Peter Tyonum](https://dev.to/tvpeter)                                                                              | [How to Setup Bitcoin Core and Lightning Network Node Developer Environment](https://dev.to/tvpeter/how-to-setup-bitcoin-core-and-lightning-network-node-developer-environment-3lil) | Start here   |
| [Michael Goldstein](https://medium.com/@bitstein?source=post_page-----ab967167594a--------------------------------) | [Setting Up a Bitcoin/Lightning Network Test Environment](https://medium.com/@bitstein/setting-up-a-bitcoin-lightning-network-test-environment-ab967167594a)                         | Go here next |

**My system specs**

- Machine: MacBook Pro
- Operating System: macOS Monterey
- terminal: kitty
- shell: ZSH
- Editor: Neovim
- Package Manager: [Homebrew](https://brew.sh/)

**The boring but very important stuff**

Before we get to copying and pasting stuff into our terminal(the fun part), let's look at what the internet has to say about the following:

1. The lightning network and how it interacts with the Bitcoin network
2. The various Bitcoin network modes.

> "The Lightning Network is a second-layer protocol built on top of the Bitcoin blockchain. It enables faster and cheaper transactions by creating off-chain payment channels between users. These channels allow multiple transactions to occur without needing to be recorded on the main Bitcoin blockchain. Instead, only the opening and closing transactions of the channel are settled on the blockchain. This interaction reduces congestion on the Bitcoin network and enables microtransactions with almost instant settlement times."

**The Bitcoin testnet network mode**

> _"It’s a public test blockchain where the bitcoin does not have a real-world value that works similarly to the mainnet blockchain. In this way, it is safe to test out some functionality."_ **_source:_** [**_https://studygroup.moralis.io/_**](https://studygroup.moralis.io/t/bitcoin-testnet-vs-regtest-reading-assignment/7914/2#:~:text=Apr%20%2719-,What%20is%20testnet%20in%20Bitcoin,-%3F%0Ait%E2%80%99s%20a)

**The Bitcoin regtest network mode** (this is the mode we are going to be using)

> \*"Regtest (regression test) mode creates a local private blockchain where you can adjust the parameters to what you want. You usually use this when it is not needed to communicate with other peers and blocks. An example of what you can do with the parameters is you can create blocks instantly"\***_Source:_** [**_https://studygroup.moralis.io/_**](https://studygroup.moralis.io/t/bitcoin-testnet-vs-regtest-reading-assignment/7914/2#:~:text=Apr%20%2719-,What%20is%20testnet%20in%20Bitcoin,-%3F%0Ait%E2%80%99s%20a)

**The Bitcoin signet network mode**

> \*"Signet is a proposed new test network parallel to the Bitcoin network. Like testnet and regtest, developers would use signet as a testing environment. Unlike the Bitcoin mainnet or the other test networks, signet would use digital signatures to validate blocks, not a Proof-of-Work system."\***_Source:_** [**_https://river.com/_**](https://river.com/learn/terms/s/signet/#:~:text=Signet%20is%20a%20proposed%20new%20test%20network%20parallel%20to%20the%20Bitcoin%20network.%20Like%20testnet%20and%20regtest%2C%20signet%20would%20be%20used%20by%20developers%20as%20a%20testing%20environment.%20Unlike%20the%20Bitcoin%20mainnet%20or%20the%20other%20test%20networks%2C%20signet%20would%20use%20digital%20signatures%20to%20validate%20blocks%2C%20not%20a%20Proof%2Dof%2DWork%20system.)

**Now to the good stuff:**

**Configuring our Bitcoin environment**

**Step 1a**

This part is very important. if you have gone over the [Setting Up a Bitcoin/Lightning Network Test Environment](https://medium.com/@bitstein/setting-up-a-bitcoin-lightning-network-test-environment-ab967167594a) as I suggested above you would realize that the author manually created a `bitcoin.conf` but we'll be using the one at `/Users/user/Library/Application Support/Bitcoin/bitcoin.conf` auto-generated while installing bitcoin core.

now let's open up the autogenerated `bitcoin.conf` file with our text editor

```plaintext
nvim /Users/user/Library/Application Support/Bitcoin/bitcoin.conf
```

Add these settings below

```plaintext
# Daemon Options
server=1
daemon=1
fallbackfee=0.00072
txindex=1
mintxfee=0.0001
txconfirmtarget=6

# Network Options
regtest=1
#signet=1
#testnet=1

[regtest]
# RPC Options
rpcport=18443
rpcuser=bitcoin
rpcpassword=bitcoin

# ZMQ Options
zmqpubrawblock=tcp://127.0.0.1:28332
zmqpubrawtx=tcp://127.0.0.1:28333
zmqpubhashtx=tcp://127.0.0.1:28332
zmqpubhashblock=tcp://127.0.0.1:28332
```

Now, you can run your Bitcoin node service and the following commands from the terminal. Ensure your Bitcoin backend is all set and ready to run your lightning network.

Take note of the corresponding outputs to make sure you're on the right track. please note that I have truncated some outputs.

_Start bitcoin service_

```plaintext
$ bitcoind
```

**Output:**

```plaintext
Bitcoin Core starting
```

_ensure you are on regtest_

```plaintext
$ bitcoin-cli  getblockchaininfo
```

**output:**

```json
{
  "chain": "regtest",  // just making sure we are on the 'regtest' network
  "blocks": 526,
  "headers": 526,
...
}
```

_create a Bitcoin wallet_

```plaintext
$ bitcoin-cli createwallet "my_first_regtest_wallet"
```

**output:**

```json
{
  "name": "my_first_regtest_wallet"
}
```

_Create an address (we'll be generating a_ `legacy` address)

> \*\*Quick note: \*\* A legacy address refers to a format for cryptocurrency addresses that was used in older versions of the blockchain protocol. For example, in Bitcoin, legacy addresses typically start with the number "1". These addresses are still functional but are being gradually phased out in favor of newer address formats like Segregated Witness (SegWit) addresses, which offer benefits like lower transaction fees and improved scalability.

```plaintext
$ bitcoin-cli getnewaddress -addresstype legacy
```

**output**

```plaintext
mhF5zQHNn7wzaaQTpzRmtsNgPgFF94fxeY  // yours will be different
```

_Assign a label to the address generated above to make it easy to identify it_

```plaintext
 $ bitcoin-cli setlabel "mhF5zQHNn7wzaaQTpzRmtsNgPgFF94fxeY" "my_first_address"
```

_get a list of all our generated addresses_

```plaintext
 $ bitcoin-cli listreceivedbyaddress 1 true
```

**output**

```json
[
  {
    "address": "mhF5zQHNn7wzaaQTpzRmtsNgPgFF94fxeY",
    "amount": 0.0,
    "confirmations": 0,
    "label": "my_first_address",
    "txids": []
  }
]
```

_Mine some block to the address (This is important)_

> **Quick note:** I faced a sticky issue when trying to run my `lncli commands` and so after doing some research found a solution which was basically to generate some blocks check here to know more about the issue [https://github.com/lightningnetwork/lnd/issues/1177#issuecomment-1100012559](https://github.com/lightningnetwork/lnd/issues/1177#issuecomment-1100012559)

```plaintext
 $  bitcoin-cli generatetoaddress 101 "mhF5zQHNn7wzaaQTpzRmtsNgPgFF94fxeY"
```

**output**

```json
[
   "461c729e6afecd371a5dac52aa06955fbcaa41688b67d1130887a761ab972064",
  "793c537c37a6cea487b762fdb2ca0af6bf15797f5f8f9847fe07dcf982cdc6b4",
  "422a76fd34cbf65743416f1e256129935c04cd40cb81a80c891e28425f78974d",
  "1a987221c802116c70c18b2cdd1233ec906c0800835eac3fe9e489b17d911ce8",
  ...
]
```

_Check our wallet info_

```plaintext
$ bitcoin-cli getwalletinfo
```

**Output**

```json
{
  "walletname": "my_first_regtest_wallet",
  "walletversion": 169900,
  "format": "sqlite",
  "balance": 137.5,
  "unconfirmed_balance": 0.0,
  "immature_balance": 471.875,
  "txcount": 122,
  "keypoolsize": 4000,
  "keypoolsize_hd_internal": 4000,
  "paytxfee": 0.0,
  "private_keys_enabled": true,
  "avoid_reuse": false,
  "scanning": false,
  "descriptors": true,
  "external_signer": false,
  "blank": false,
  "lastprocessedblock": {
    "hash": "0da2df1e8591e6ac00c1390fe397580b267c5b63a729f802e0bdd458aafb46f0",
    "height": 648
  }
}
```

**Configuring two LND nodes**

**Step 1a**

we'll be creating a base or root directory called `.lnd` using the [mkdir](https://phoenixnap.com/kb/create-directory-linux-mkdir-command#:~:text=The%20mkdir%20command%20in%20Linux/Unix%20is%20a%20command%2Dline%20utility%20that%20allows%20users%20to%20create%20new%20directories.%20mkdir%20stands%20for%20%22make%20directory.%22) Linux command. please note that this directory can be named anything you wish same

```plaintext
$ mkdir ~/.lnd/.lnd1
$ mkdir ~/.lnd/.lnd2
```

Create a configuration file and add the settings below then save it as lnd.conf in the .lnd1 directory.

```plaintext
[Bitcoin]

bitcoin.active=1
bitcoin.regtest=1
bitcoin.node=bitcoind

[Bitcoind]

bitcoind.rpchost=127.0.0.1
bitcoind.rpcuser=bitcoin
bitcoind.rpcpass=bitcoin
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333
```

**Step 1b**

Now just like in step 1a above, Create a configuration file add the settings below, and then save it as lnd.conf in the .lnd2 directory.

```plaintext
[Application Options]

listen=0.0.0.0:9734
rpclisten=127.0.0.1:11009
restlisten=0.0.0.0:8180

[Bitcoin]

bitcoin.active=1
bitcoin.regtest=1
bitcoin.node=bitcoind

[Bitcoind]

bitcoind.rpchost=127.0.0.1
bitcoind.rpcuser=bitcoin
bitcoind.rpcpass=bitcoin
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333
```

> \*\*Quick note: \*\* Notice that the config above has different ports and settings for network, RPC, and REST connections configured. well, to mention a few reasons:

- Each LND instance needs to listen on different ports to avoid conflicts.
- You might want `lnd1` and `lnd2` to use different wallets or have different wallet configurations. \_ You might configure different network settings for each LND instance. For example, one might be configured to run on the `Bitcoin mainnet` while the other runs on the `Bitcoin testnet` but note that it is required that you run the two LN nodes on the same network for the sake of running transactions between them

**Step 2**

lastly, let's configure some aliases so let's open up `~/.zshrc` with our favourite text editor once again

```plaintext
nvim ~/.zshrc
```

Now add this to the bottom.

```plaintext
# local variables
export LND1_DIR="$HOME/.lnd/.lnd1"
export LND2_DIR="$HOME/.lnd/.lnd2"

# aliases
#lnd1
alias lnd1="lnd --lnddir=$LND1_DIR";
alias lncli1="lncli -n regtest --lnddir=$LND1_DIR"

#lnd2
alias lnd2="lnd --lnddir=$LND2_DIR";
alias lncli2="lncli -n regtest --lnddir=$LND2_DIR --rpcserver=localhost:11009"
```

---

_This is the end of the first part of the series. we'll be going over some preliminary setup and configuration for a successful transaction between 2 lightning nodes in the next part_

---

**Conclusion**

you feel rushed, don't you? Don't worry It took me close to a week to understand this so it's fine if you don't entirely get what's going on for now. check out some of these useful links below [chainquery: bitcoin-cli documentation](https://chainquery.com/bitcoin-cli)
