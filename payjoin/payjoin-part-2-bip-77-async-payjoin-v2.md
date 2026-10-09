# Payjoin Part 2 — BIP 77: Payjoin Without a Server

> Previously: [Part 1 — BIP 78: When the Receiver Chips In](./payjoin-part-1-bip-78-synchronous-payjoin-v1.md)

## Objective

By the end of this article you should be able to:

- explain how BIP 77 lets a phone wallet receive a Payjoin
- read the extra pieces inside a BIP 77 QR code
- explain how a middleman server passes messages it can't read, from senders it can't identify
- say what happens when someone drops out halfway, and what still leaks

## Same payment, different Bob

We're keeping the numbers from Part 1: Alice pays Bob 50,000 sats, Bob adds a 30,000-sat coin, and the final transaction is 209 vB with a 2,090-sat fee. The transaction doesn't change at all in BIP 77. What changes is how the two wallets reach each other.

This time Bob isn't a shop. He's a friend Alice owes for dinner, and his wallet is an app on a phone that's locked in his pocket. Under BIP 78, Alice's wallet would try to contact Bob's server, find nothing, and fall back to a plain payment.

BIP 77, written by Dan Gould and Yuval Kogman, fixes this by giving Bob a **mailbox** on a server called the **Payjoin Directory**. Alice leaves her draft there, and Bob picks it up whenever his app next wakes up. Bob's reply goes into a second mailbox that Alice checks. Neither of them runs a server or needs to be online at the same time as the other.

Here's what that looks like over an actual evening:

```text
             Alice                 Directory                  Bob
Mon 20:55                                              makes request,
                                                       locks phone
Mon 21:04    sends draft  ──────▶  [Bob's box: 1]
Mon 21:06    closes app
Tue 07:31                          [Bob's box: 1]  ──▶ opens app, adds
                                   [Alice's box: 1] ◀── coin, replies
Tue 08:10    checks box   ◀──────  [Alice's box: 1]
             signs, broadcasts
```

Eleven hours passed between Alice tapping Send and the payment going out, and at no point were both of them online.

## What's hiding in the QR code

Bob's QR code looks like this, shortened here:

```text
BITCOIN:BC1QBOB...?amount=0.0005&pjos=0&pj=HTTPS://PAYJO.IN/TXJCGKTKXLUUZ%23EX1...-OH1...-RK1...
                                           └───────────────┘└───────────┘└─┘└──────────────────┘
                                               directory       mailbox    #       fragment
```

The part after the `#` (written `%23`) holds three values:

```text
EX1...   when this request expires (say, 24 hours from now)
OH1...   the directory's encryption key, for the privacy relay (below)
RK1...   Bob's public key for this one request
```

That `#` isn't there by accident. Your wallet never sends the part of a web address after `#` to the server, so Bob's key travels straight from his screen to Alice's camera and never passes through the directory. Bob also makes a new key, and so a new mailbox, for every request, so two of his requests can't be linked together.

Everything is in capitals because QR codes pack uppercase letters and digits much more tightly, which makes the code smaller and easier to scan.

## A mailbox nobody can read

A server in the middle sounds worse than Part 1, where Alice talked directly to Bob. BIP 77 deals with that using two layers of protection.

**Layer one: sealed envelopes.** Alice locks her draft with Bob's key using **HPKE**, a standard way to encrypt a message to someone using only their public key. Inside the envelope she also puts a fresh key of her own, which tells Bob how to seal his reply and which mailbox to put it in. The directory only ever holds sealed envelopes. It doesn't even know Alice's reply key, only the mailbox number derived from it, so it couldn't forge a reply that Alice would accept.

**Layer two: a forwarding service.** Encryption hides what's inside the message, but not who sent it. If Alice connected to the directory directly, it would see her IP address. So she sends everything through an **OHTTP relay**, which is run by a different operator from the directory. The relay can see who's sending but can't read anything. The directory can read the address on the envelope but only ever sees the relay as the sender.

Finally, every envelope is padded to the same fixed size, so a draft with one input looks the same as one with ten.

Put together, here's who can see what:

```text
                        Relay    Directory    Bob
Alice's IP address       yes        no         no
Which mailbox            no         yes        yes
The transaction          no         no         yes
Message size             no         no         n/a
```

## When someone vanishes

Because this all happens over hours, someone might never come back. Every case still ends with Bob getting paid:

```text
Bob never shows up      -> after expiry, Alice broadcasts her Original
Alice never comes back  -> Bob broadcasts the Original she sent him
Both show up            -> the Payjoin goes out
```

The Original, the plain signed payment from Part 1, is what makes this safe. Because it's always a valid transaction, Bob never has to trust Alice to finish. It also blocks the free-coin-scouting trick from Part 1, because anyone who sends an Original can end up paying.

## Older wallets

A wallet that only knows BIP 78 can still pay a BIP 77 request. It sends a plain draft to the mailbox address, and the directory keeps that connection open until Bob replies. That comes with two costs. Bob has to be online within that short window, and without the sealed envelope the directory can read, and potentially change, the drafts.

That second cost is why the QR code includes `pjos=0`. It stops Bob from changing the address his output pays to. If the directory tried to swap in its own address, Alice's wallet would spot the change and refuse to sign.

## What still leaks

A few things are still visible:

- **Timing.** The directory sees when envelopes arrive and when they're collected. If only a few Payjoins happen each hour, it could try matching those times against new transactions appearing on the network.
- **Collusion.** If the relay operator and the directory operator work together, they can link IP addresses to mailboxes. They still can't read the transactions.
- **Refusal.** The directory can simply refuse to deliver messages. The worst outcome is a normal payment.
- **Coming back.** Both people still have to come back online before the request expires, or there's no Payjoin.

None of these put Alice's or Bob's funds at risk.

## Why it's worth it

Part 1 showed that a Payjoin makes an analyst confidently wrong. That only has a big effect if Payjoins are common. With BIP 78 they mostly happened at shops. With BIP 77, anyone's phone can receive one. And every Payjoin makes the "all inputs belong to one person" guess a little less reliable, even for transactions that were never Payjoins.

## Further reading

- [BIP 77 — Async Payjoin](https://github.com/bitcoin/bips/blob/master/bip-0077.md)
- [RFC 9180 — HPKE](https://www.rfc-editor.org/rfc/rfc9180)
- [RFC 9458 — Oblivious HTTP](https://www.rfc-editor.org/rfc/rfc9458)
- [payjoin.org](https://payjoin.org/)
- [rust-payjoin](https://github.com/payjoin/rust-payjoin)
- Inspired by [Carlos Santos's Payjoin write-up](https://www.carlossantos.sh/posts/payjoin-bip77-bip78/), which is well worth reading for a different angle.
