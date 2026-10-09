# Objective

The goal of this article is to help you

- Get a rough view of what a cryptocurrency is.
- Gain some foundational understanding of What Bitcoin is
  - make sense of the name `Satoshi Nakomoto` (unfortunately the name of an anime protagonist)
  - understand some Bitcoin terminologies
- understand the difference between Bitcoin and bitcoin
- understand Bitcoin wallets
- understand Bitcoin addresses
  - Hashing
  - Legacy (P2PKH)
  - Nested SigWit (P2SH)
  - Native SegWit (Bech32)
  - Taproot (P2TR)

# Introduction

I know you must have heard a lot about Bitcoin and how it has made a lot of people filthy rich over the years.

But beyond all the hype, do you understand the technology and infrastructure Bitcoin is layered upon? My job in this series is not to overload you with too many technicalities but to give you enough dumb down knowledge to help lean your curiosity in the right direction.

# What is a cryptocurrency

The word `cryptocurrency` can be broken down into two parts

- `crypto`: a short form for the word `cryptography` which is a mathematical domain of knowledge that deals with the encryption and decryption of data. remember how Batman uses his `wrist computer thingy` to decipher passwords for locked doors? now that process is data decryption which is a vital fragment of cryptography.

# What is `bitcoin (BTC))`

> Quick note: Bitcoin was released in 2009 by some anonymous nerd or group of nerds under the name Satoshi Nakamoto.

Who is Satoshi Nakamoto?

After asking Google the same question I got this:

> _"Satoshi Nakamoto is the name used by the presumed pseudonymous person or persons who developed Bitcoin, authored the bitcoin white paper, and created and deployed bitcoin's original reference implementation."_[Wikipedia](https://en.wikipedia.org/wiki/Satoshi_Nakamoto)

I believe the developer or developers of bitcoin maintained anonymity for a good cause, seeing as the Bitcoin infrastructure promotes heavy anonymity by obfuscating various data on the network via encryption it maybe wouldn't have made sense to have attached a face to the invention. But don't take my word for it, I'm also a dummy trying to understand Bitcoin.

# A definition of sorts

bitcoin (BTC) is a cryptocurrency, a virtual currency designed to act as money and a form of payment outside the control of a centralized system say your typical bank, so there's no need for third-party involvement in transactions. bitcoin (BTC) is also rewarded to `blockchain miners` for `verifying transactions` and can be purchased in specialized markets called exchanges.

# What is a cryptocurrency exchange?

Well, it's a market that provides a platform for customers to swap one cryptocurrency into another and to also buy cryptocurrencies.

# The difference between `Bitcoin` and `bitcoin`

Owing to my explanation above, `bitcoin (BTC)` is simply the currency or `native token` used in transactions on a cool type of network that connects computers/nodes to form peers called a `peer-to-peer network`. What makes this type of network cool is that all participating nodes have the liberty to verify transactions and add them into a `book of records` or you can call it a `ledger` as part of a collection of other transactions called a `block` which is then append it to the end of another `Block` to form `a chain of blocks`. hence the name `blockchain`.

> Quick Note: for clarity, `Transactions` are grouped into `Blocks` which are then appended to the previous block to form a `Blockchain`

- A `node` is any computer on the Bitcoin network which by default has the exact copy of the entire blockchain data
- A `block` is a record made up of a collection of transactions and meta-data grouped together as a single entity

  ![block image](https://www.researchgate.net/profile/Khaled-Salah-8/publication/321017113/figure/fig2/AS:614287290679309@1523468911795/Blockchain-design-structure-showing-chained-blocks-with-header-and-body-fields.png)

A compendium of all these concepts and technologies that facilitate the execution of activities on the `blockchain` can be referred to as `Bitcoin`.

> Quick note: Going forward, i will be referring to the bitcoin cryptocurrency as BTC for ease of identification and understanding and it's shorter to write.

# Bitcoin Wallets

Just like with conventional wallets, A Bitcoin wallet stores BTC you have `ownership` of. It is a single source of truth for determining your BTC balance and spending from the balance and also adding to the balance via a special set of alphanumeric characters called an `Address`. Lastly, a bitcoin wallet can either be a physical device like a USB drive, SSD, or HDD, or it can also be a computer program.

# Bitcoin Addresses

Let's quickly talk about `hashing`. Hashing is a cryptographic process of passing a set of data through a secure hash algorithm to produce an encrypted value called a `Digest`.

![Hashing](https://www.simplilearn.com/ice9/free_resources_article_thumb/hashing1.PNG)
_source: simplilearn_

On a more technical level, a Bitcoin wallet is a database file that contains a set of digital keys mathematically related to each other. they are key pairs that serve as

- A public id that serves as what you would call a bank account number in the centralized space. it is called a `Public key` and is used to receive and send BTC on the network
- A private ID (`Private Key`) which is more or less a `PIN`. It is a secret set of alphanumeric characters that is used to generate `digital signatures` for the sake of transaction validation.

> Quick Note: the public/private key value pairs are generated through a cryptographic algorithm called an `Elliptic curve cryptographic function`. if you care to read more about it and lose a little bit of your sanity then check out [Elliptic Curve Cryptography Explained](https://learn.saylor.org/mod/book/view.php?id=36353#:~:text=to%20generate%20addresses.-,Elliptic%20Curve%20Cryptography%20Explained,-Elliptic%20curve%20cryptography).

Bitcoin Addresses are generated from a public key via a `one-way cryptographic hashing algorithm` and are about 26 to 35 characters long.

The algorithms mentioned above leverages on `double-hashing`:

- Secure Hash Algorithm (SHA256)
- RACE Integrity Primitives Evaluation Message Digest (RIPEMD160)

And the final `digest` is passed through Base58Check encoding to improve the human readability of the final address

Mathematically:

```md
// one-way cryptographic hashing algorithm

Digest = SHA256(PublicKey)
DigestTwo = RIPEMD160(Digest)
finalAddress = Base58Check(DigestTwo) //encoding
```

Now that we have gone over how to create a Bitcoin address, below is are the different types of Bitcoin address:

- Legacy (P2PKH)
- Nested SigWit (P2SH)
- Native SegWit (Bech32)
- Taproot (P2TR)

# Conclusion

This is the end of the first part of the series, I hope you were able to gain some surface-level understanding of Bitcoin at least. In the next part, we'll dive deep into the different types of Bitcoin addresses mentioned above. Thank you.

# Useful links

- [Bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)
- [Soren Azorian | Demystifying Bitcoin Address Types: Legacy, SegWit, Native SegWit, and Taproot](https://www.linkedin.com/pulse/demystifying-bitcoin-address-types-legacy-segwit-native-soren-azorian/)
