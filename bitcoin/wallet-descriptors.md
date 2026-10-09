# Objective

The goal of this article is to help you

- Understand what a wallet descriptor is and why Bitcoin wallets use them
- Break down the anatomy of a descriptor, piece by piece
  - Script type
  - Key origin
  - Extended key and derivation path
  - Checksum
- Decode a sample wallet descriptor with a small Rust app

# Requirements

- Some understanding of HD wallets. If you haven't already, check out my article on HD wallets, since descriptors build directly on top of BIP32 and BIP44
- Rust installed and configured on your machine
- Some basic knowledge of the Rust language

# Introduction

In the HD wallets article we saw that a single seed phrase can produce an unlimited number of addresses. But there's a catch nobody tells you about. The seed alone isn't enough to find your money.

Say you restore your 12 words into a new wallet app. Which derivation path should it walk, `m/44'/0'/0'` or `m/84'/0'/0'`? Should it build legacy addresses starting with `1`, or native SegWit addresses starting with `bc1q`? Is it a single-key wallet or a 2-of-3 multisig? Pick the wrong combination and the app shows a balance of zero, even though your coins are sitting right there on the blockchain.

Wallet descriptors fix this by writing all of those answers down in one line of text.

# What is a wallet descriptor

A wallet descriptor is a short, human-readable string that describes exactly which scripts, and therefore which addresses, belong to a wallet. It was introduced in Bitcoin Core and later standardised in BIP380 through BIP386.

A seed phrase is a key. A descriptor is the map that tells a wallet what to do with that key. Two wallets given the same descriptor will always produce the exact same addresses, in the same order.

You've actually already used one. Back in Part 1 of my Lightning series, the output of `bitcoin-cli getwalletinfo` included the line `"descriptors": true`. Modern Bitcoin Core wallets are descriptor wallets, and you can see yours by running `bitcoin-cli listdescriptors`.

> Quick note: a descriptor that contains only public keys (`xpub`) can produce addresses and watch balances but can't spend. That makes it safe to hand to a watch-only wallet, but it still reveals every address you'll ever use, so treat it as private information.

# Anatomy of a wallet descriptor

Here's the descriptor we'll be working with for the rest of the article:

```text
wpkh([73c5da0a/84'/0'/0']xpub6CatWdiZ...C7PW6V/0/*)#wc3n3van
```

It looks like a jumble, so let's pull it apart:

```text
wpkh( [73c5da0a/84'/0'/0'] xpub6CatWdiZ...C7PW6V /0/* ) #wc3n3van
└┬─┘  └────────┬─────────┘ └─────────┬─────────┘ └┬─┘   └───┬───┘
 │             │                     │            │         │
 │             │                     │            │         └─ checksum
 │             │                     │            └─ what to derive next:
 │             │                     │               receive chain 0, every index
 │             │                     └─ extended public key for the account
 │             └─ key origin: master fingerprint + path used to reach this key
 └─ script type: pay-to-witness-public-key-hash (native SegWit)
```

## Script type

The outer function says what kind of script, and so what kind of address, to build. The common ones line up with the address types from Part 2 of my Bitcoin series:

```text
pkh(KEY)             legacy (P2PKH)            addresses start with 1
sh(wpkh(KEY))        nested SegWit (P2SH)      addresses start with 3
wpkh(KEY)            native SegWit (P2WPKH)    addresses start with bc1q
tr(KEY)              taproot (P2TR)            addresses start with bc1p
wsh(multi(2,A,B,C))  2-of-3 multisig (P2WSH)   addresses start with bc1q
```

Notice how they nest. `sh(wpkh(...))` literally reads as "a SegWit script wrapped inside a P2SH script", which is exactly what nested SegWit is.

## Key origin

The part in square brackets, `[73c5da0a/84'/0'/0']`, records where the key came from. `73c5da0a` is the fingerprint of the master key, the first four bytes of a hash of its public key. `84'/0'/0'` is the path walked from the master key to reach this account. The apostrophe marks a hardened step, just like in the BIP44 paths from the HD wallets article.

The wallet doesn't strictly need this to generate addresses. Hardware wallets do need it, though. When a hardware wallet is asked to sign, it uses the fingerprint and path to work out which of its own keys to use.

## Extended key and derivation path

`xpub6CatWdiZ...` is the extended public key of the account. After it, `/0/*` tells the wallet what to derive next. `0` is the receiving chain, and `*` is a wildcard meaning "every index: 0, 1, 2, and so on". A descriptor with a `*` in it is called a ranged descriptor. Your change addresses get a second descriptor that is identical except it ends in `/1/*`.

## Checksum

The eight characters after the `#` are a checksum over everything before it. If a single character gets mistyped or corrupted, the checksum no longer matches and the wallet refuses to load the descriptor, rather than quietly watching the wrong addresses.

# Decoding a sample wallet descriptor with rust

Enough theory, let's decode one. Create a new project and add the `miniscript` crate, which knows how to parse descriptors and comes with the `bitcoin` crate built in.

```zsh
cargo new descriptor-demo && cd descriptor-demo
cargo add miniscript@12
```

Now replace everything in `src/main.rs` with the code below

```rust
use std::str::FromStr;

use miniscript::bitcoin::Network;
use miniscript::{Descriptor, DescriptorPublicKey};

fn main() {
    let text = "wpkh([73c5da0a/84'/0'/0']xpub6CatWdiZiodmUeTDp8LT5or8nmbKNcuyvz7WyksVFkKB4RHwCD3XyuvPEbvqAQY3rAPshWcMLoP2fMFMKHPJ4ZeZXYVUhLv1VMrjPC7PW6V/0/*)";

    // Parse the string into a typed descriptor (fails on typos or bad keys)
    let descriptor = Descriptor::<DescriptorPublicKey>::from_str(text).expect("valid descriptor");

    println!("Type:     {:?}", descriptor.desc_type());
    println!("Ranged:   {}", descriptor.has_wildcard());
    println!("Full:     {descriptor}");

    // Replace the * with 0, 1, 2 and turn each result into an address
    for i in 0..3 {
        let address = descriptor
            .at_derivation_index(i)
            .expect("index is not hardened")
            .address(Network::Bitcoin)
            .expect("wpkh has an address form");
        println!("/0/{i}  ->  {address}");
    }
}
```

Run it with `cargo run`

```zsh
Type:     Wpkh
Ranged:   true
Full:     wpkh([73c5da0a/84'/0'/0']xpub6CatWdiZiodmUeTDp8LT5or8nmbKNcuyvz7WyksVFkKB4RHwCD3XyuvPEbvqAQY3rAPshWcMLoP2fMFMKHPJ4ZeZXYVUhLv1VMrjPC7PW6V/0/*)#wc3n3van
/0/0  ->  bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu
/0/1  ->  bc1qnjg0jd8228aq7egyzacy8cys3knf9xvrerkf9g
/0/2  ->  bc1qp59yckz4ae5c4efgw2s5wfyvrz0ala7rgvuz8z
```

Here's what happened:

- `from_str` parsed our string into a `Descriptor`, checking that every part is valid along the way. `Descriptor::<DescriptorPublicKey>` tells Rust which kind of key we expect inside, here a public key that may carry a path and a wildcard. The `::<...>` syntax is just Rust's way of filling in that type when it can't guess it.
- `desc_type()` confirms it's a `Wpkh` descriptor, and `has_wildcard()` confirms the `*` makes it ranged. The `{:?}` in `println!` prints a value in its debug form, which is handy for types like this one.
- When we printed the full descriptor, the library added the `#wc3n3van` checksum for us.
- `at_derivation_index(i)` swaps the `*` for a real number, and `.address(Network::Bitcoin)` builds the address for that index.

Remember the `abandon abandon ... about` test phrase from the HD wallets article? This descriptor is the native SegWit account from that same phrase. That's why the master fingerprint is `73c5da0a`, and why the first address, `bc1qcr8te4kr609gcawutmrza0j4xv80jy8z306fyu`, matches the official BIP84 test vector.

To see the checksum doing its job, add `#wc3n3van` to the end of `text`, then change `84'` to `48'`. The program now stops with:

```zsh
Invalid descriptor: Invalid checksum 'wc3n3van', expected 'y3t0v3qt'
```

One changed digit, and the whole thing is rejected instead of silently producing the wrong addresses.

# Conclusion

A seed phrase on its own tells a wallet nothing about which addresses you actually used. A wallet descriptor packs the script type, key origin, extended key, derivation path, and a checksum into one line that any descriptor-aware wallet can read the same way. If you ever back up a wallet, back up its descriptors along with the seed phrase, and future you will be thankful.

# Useful links

- [Bitcoin Core: Support for Output Descriptors](https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md)
- [BIP380: Output Script Descriptors General Operation](https://github.com/bitcoin/bips/blob/master/bip-0380.mediawiki)
- [BIP84: Derivation scheme for P2WPKH based accounts](https://github.com/bitcoin/bips/blob/master/bip-0084.mediawiki)
- [rust-miniscript](https://github.com/rust-bitcoin/rust-miniscript)
