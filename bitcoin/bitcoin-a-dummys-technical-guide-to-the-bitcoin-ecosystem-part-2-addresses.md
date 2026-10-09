### Objective

The goal of this article is to help you

- understand the various types of Bitcoin addresses and their impact on transactions
  - Legacy (P2PKH)
  - Nested SigWit (P2SH)
  - Native SegWit (Bech32)
  - Taproot (P2TR)

### Introduction

Welcome to the second part of the series. incase if you haven't already gone over the first part, please ensure you check it out here [Bitcoin - A dummy's technical guide to the bitcoin ecosystem part 1](https://dev.to/shytypes1028/bitcoin-a-dummys-technical-guide-to-the-bitcoin-ecosystem-part-1-oje). So you would have a better understanding of this part.

In this part we'll be talking about the different bitcoin addresses and what forms of improvement they bring into the Bitcoin ecosystem. we may get a little technical in this article but hey it's nothing we can't handle right?

Now let's get to it shall we?

### Understanding Bitcoin addresses

Now we know that the Bitcoin address is a fundamental part of the Bitcoin ecosystem that ensures transaction validations, generation of digital signatures and a host of important functionalities. Owing to the importance of Bitcoin addresses, over the years there have been multiple updates via soft forks to the Bitcoin network that have added improvements to Bitcoin addresses and their impact on the Bitcoin network.

### Why do we have different Bitcoin address types?

Have you ever wondered why there are multiple Bitcoin addresses and what improvements they bring into the Bitcoin ecosystem?. Keep reading for a brief breakdown.

Well around 2016 and 2017, there was a discussion or debate around the Bitcoin `Block size` which was `1 mb` per block and so only a limited amount of transactions could be added to a block. The accepted solution was to take a part of the transaction called the `witness data` and move it to a second layer on the blockchain there by allowing more space for transactions on the main block and increasing the block size naturally allowing more transactions to fit into a block.

Now Let's do a quick walkthrough of the various Bitcoin addresses with some quick facts.

### Legacy Address a.k.a Pay-to-Public-Key-Hash (P2PKH)

This is the oldest of all Bitcoin addresses. In fact, it is the original Bitcoin address and has a prefix of `1`.

Quick Facts

- They are accountable, stable and universally supported.
- Transactions via Legacy addresses have are large in size, meaning they take up more block space.
- transactions from Legacy addresses have high transaction fees due to the size of the transactions
- Legacy addresses are `not` SegWit compatible what this means is that on the Bitcoin network, only Legacy addresses can carry out transactions with with each other i'e send and and receive BTC.
- Legacy addresses have long transaction processing times.

### Nested SegWit a.k.a Pay-to-Script-Hash (P2SH)

This address is an improvement over `Legacy addresses` the implements better scalability, security and smaller block size their by enabling multiple transactions on the blockchain without having too much of an impact on the the Bitcoin blockchain size. it has a prefix of `3`.

Nested SegWit was launched around 2017 and it basically reduced the size of each Bitcoin transaction's data by isolating part of the transaction signature data from the main transaction data.

Quick Facts

- They have cheeper transaction fees and enhanced security over `Legacy addresses` thy achieve this through multi-signature capabilities.
- Nested SegWit addresses have cross compatibility meaning transactions can be carried out between Nested SegWit and other types of Bitcoin Addresses.
- The introduction and implementation of Nested SegWit gave birth to the Lightning Network

### Native SegWit a.k.a pay-to-witness-public-key-hash (Bech32)

This type of Bitcoin address has a prefix of "bc1q" and it is an upgrade to the Nested SegWit. It is also compatible with most wallets and exchanges by default. Native SegWit has a primary focus on weight efficiency, which brings about a reduction in the weight of Bitcoin blocks and this weight reduction naturally translates to faster transaction speeds, which enhances the network’s scalability and further lowers the transaction fees.

Quick Facts

- Native SegWit Bitcoin addresses are in lowercase for enhanced readability and better error detection

### Taproot a.k.a Pay-to-Taproot (P2TR)

`Taproot` is by far one of the coolest things to have ever happened to the Bitcoin ecosystem.

This Bitcoin Address type has a prefix pf "bc1p" and has the lowest transaction fees of all existing Bitcoin Address types. Taproot (P2TR) is a new address format that was introduced in 2021 as an update to the Segwit address type. One major shift in the dynamics of bitcoin addresses via Taproot is that it implements the `Schnorr digital signature` which is a shift from the conventional `Elliptic curve digital signature algorithm` previous bitcoin address types have of used for years.

Quick Facts

- Taproot aims to make bitcoin transactions cheeper, more efficient, private and also unlocks the potential of smart contracts on the bitcoin network via a couple of new implementations one of which is a system called the `Merklized Abstract Syntax Trees (MAST)` which enables conditions placed on a transaction to be condensed and hashed into one script.
- The `Schnorr digital signatures` which Taproot makes use of allows for transaction grouping. what this means is that multiple transactions from a bitcoin wallet can be hashed together and assigned a unique key. Simple and complex transactions containing multiple signatures would naturally be indistinguishable via this hashing.

Quick Link: [https://tara-annison.medium.com/what-are-taproot-mast-and-schnorr-signatures-b737dae20681](https://tara-annison.medium.com/what-are-taproot-mast-and-schnorr-signatures-b737dae20681)

### Conclusion

The best part about an open blockchain system is how it is community driven and so over time improvements are welcomed and accepted to further scale the network. So Which of the Bitcoin addresses would you be making use of?

Thanks for staying till the end, Goodbye and see you in the next one.

### Useful links

Check out these links for further reading

- [Bitcoinbook](https://github.com/bitcoinbook/bitcoinbook)
- [Soren Azorian | Demystifying Bitcoin Address Types: Legacy, SegWit, Native SegWit, and Taproot](https://www.linkedin.com/pulse/demystifying-bitcoin-address-types-legacy-segwit-native-soren-azorian/)
- [Bitcoin - Native SegWit vs Taproot: A Comprehensive Guide for Beginners](https://trustwallet.com/blog/bitcoin-native-segwit-vs-taproot-a-comprehensive-guide-for-beginners)
- [What are Taproot, MAST and Schnorr Signatures?](https://tara-annison.medium.com/what-are-taproot-mast-and-schnorr-signatures-b737dae20681)
