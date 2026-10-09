# Payjoin Part 1 — BIP 78: When the Receiver Chips In

> Next up: [Part 2 — BIP 77: Payjoin Without a Server](./payjoin-part-2-bip-77-async-payjoin-v2.md)

## Objective

By the end of this article you should be able to:

- explain what an ordinary Bitcoin payment gives away about you
- explain how a Payjoin makes chain analysts draw the wrong conclusions
- follow a BIP 78 payment from QR code to broadcast
- explain the one requirement that keeps BIP 78 out of most people's hands

## Your payment leaves a receipt

Let's say Alice buys a pair of headphones from Bob's online shop for 50,000 sats. Her wallet holds one coin worth 120,000 sats, and fees are sitting at 10 sat/vB. Her wallet does the obvious thing:

```text
   120,000  (Alice)  ──┬──▶   50,000   Bob
                       └──▶   68,590   Alice's change
                              ──────
                       fee:    1,410   (141 vB x 10 sat/vB)
```

Anyone can read this transaction on-chain and make two very reasonable guesses. First, **every input belongs to the same person**, because you need a coin's key to spend it. Second, **one output is the payment and the other is change**. The round 50,000 is almost certainly the payment, so the 68,590 must be Alice's change. Now an analyst knows what Alice paid, how much she has left, and which coin to follow next.

Neither guess requires any hacking. They're just right most of the time, and that's enough to build a map of who pays whom.

## Two people, one transaction

A Payjoin flips this by having **Bob add one of his own coins** to Alice's payment. Say Bob throws in a 30,000-sat coin he already owns:

```text
   120,000  (Alice)  ──┐ ┌──▶   80,000   Bob      (50,000 + his own 30,000)
                       ├─┤
    30,000  (Bob)    ──┘ └──▶   67,910   Alice's change
                                ──────
                       fee:      2,090   (209 vB x 10 sat/vB)
```

From the outside this still looks like one person spending two coins. So the analyst's first guess now lumps Bob's coin in with Alice's. Their second guess flags the round 80,000 as the payment, when the real amount was 50,000, a number that appears nowhere on-chain. They aren't just confused. They're **confidently wrong**, and that bad data spreads into everything they conclude afterwards.

Bob gets something out of it too. Each Payjoin lets him fold an old coin into a new, bigger one, which saves him fees later. Imagine he's sitting on forty 5,000-sat coins and fees hit 20 sat/vB. Each coin then costs 1,360 sats just to spend, more than a quarter of its value. Merging one into each sale, with the customer's fee contribution covering it, clears them out without a costly sweep.

## How the two wallets talk

Bob's checkout page shows a normal Bitcoin QR code with one extra parameter:

```text
bitcoin:bc1qbob...?amount=0.0005&pj=https://bob-shop.example/payjoin
                                 └──────────────┬──────────────────┘
                                 "I speak Payjoin, send me a draft here"
```

A wallet that doesn't understand `pj=` just ignores it and pays normally, so nothing breaks. A wallet that does understand it passes the transaction back and forth as a **PSBT**, a partially signed transaction that each person can sign only their own part of:

```text
 Alice's wallet                                   Bob's server
       │  1. builds + signs a normal payment           │
       │     (the "Original")                          │
       │──── POST Original PSBT ──────────────────────▶│
       │                                               │ 2. checks it
       │                                               │ 3. adds his coin,
       │                                               │    signs only that
       │◀─── Proposal PSBT ────────────────────────────│
       │  4. checks Bob's changes                      │
       │  5. signs again, broadcasts                   │
```

Notice that Alice's Original is a complete, valid transaction. If Bob's server is down or sends back nonsense, Alice broadcasts the Original and Bob still gets paid. The worst a failed Payjoin can do is fall back to a normal payment.

She has to sign twice because a standard signature covers every input and output. Once Bob adds his coin, her first signature no longer matches the transaction.

## The fine print: fees and what Bob can touch

Bob's extra coin makes the transaction 68 vB bigger, which means 680 more sats in fees at 10 sat/vB. Alice tells Bob up front how much of that she'll cover, using parameters on her request:

```text
POST /payjoin?v=1&additionalfeeoutputindex=1&maxadditionalfeecontribution=680
```

In plain English: "you may take up to 680 sats from output #1, which is my change." That's exactly why her change dropped from 68,590 to 67,910. Bob could also choose to cover the 680 himself by shrinking his own output instead.

Before signing again, Alice's wallet checks that Bob stayed inside the lines:

- **Her side is untouched:** her inputs are all still there, and nothing else about them changed.
- **The payment is the same:** she's still paying 50,000.
- **The fee stayed in bounds:** her change dropped by no more than 680 sats.
- **Nothing was added:** there are no surprise outputs.
- **Bob's coin blends in:** it's the same script type as hers. Mixing, say, SegWit and Taproot inputs is a giveaway that two wallets were involved.

If anything fails, she broadcasts the Original instead. Alice never has to trust Bob, because she checks his work.

## Traps and how BIP 78 closes them

Here's a sneaky one. Mallory opens Bob's checkout twenty times and sends twenty Originals, but never actually pays. Each time Bob's server helpfully adds a coin, so Mallory learns twenty of his coins for free.

BIP 78 makes that expensive. Bob only accepts Originals that are valid and broadcastable, and if no Payjoin shows up within a minute or so, **Bob broadcasts the Original himself**. Twenty probes now means twenty real payments to Bob. Bob can also refuse any input he's seen before.

The other trap is redirection. If an attacker could tamper with Bob's reply, they might swap his output for their own address. That's why the `pj=` endpoint must be HTTPS or a Tor onion address. Alice can also switch off **output substitution**, which is Bob's permission to change the address his output pays to.

## The catch

Look back at the conversation diagram. Bob's side is a **live server** that has to be:

- reachable from the internet at the exact moment Alice pays
- able to answer within seconds
- holding keys that can sign automatically

A merchant running BTCPay Server can do that. Bob getting paid back by a friend on his phone can't: he has no public address, the app is asleep, and he's not going to leave his keys on a server. So BIP 78 Payjoins mostly happen at shops. Everyday payments between people, where privacy arguably matters most, get left out.

Part 2 covers how BIP 77 removes that requirement.

## Further reading

- [BIP 78 — A Simple Payjoin Proposal](https://github.com/bitcoin/bips/blob/master/bip-0078.mediawiki)
- [BIP 174 — Partially Signed Bitcoin Transactions](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki)
- [payjoin.org](https://payjoin.org/)
- [rust-payjoin](https://github.com/payjoin/rust-payjoin)
- Inspired by [Carlos Santos's Payjoin write-up](https://www.carlossantos.sh/posts/payjoin-bip77-bip78/), which is well worth reading for a different angle.
