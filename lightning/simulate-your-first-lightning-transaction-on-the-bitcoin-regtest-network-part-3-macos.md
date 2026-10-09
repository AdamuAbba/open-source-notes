### Objective

The goal of this article is to help you

- Connect the lightning nodes to form a peer
- Open a payment channel between `lnd1` and `lnd2` nodes
- Add some confirmations to the block that the channel opening transaction was mined on
- Check the list of channels to ensure our channel is ready and funded
- Create an invoice on `lnd2`
- inspect the invoice created above from `lnd1`
- Pay or satisfy the invoice created by `lnd2` via `lnd1`
- Close the payment channels, kill both Bitcoin and lightning daemons
- Celebrate the success of this tutorial with a cold bottle of beer.

### Introduction

Hey there, welcome to the final part of this 3 part series on simulating your first lightning transaction.

If you are as happy as I am to finally wrap up the series, let's get right to it....

### Prerequisites

To be successful in this tutorial in case if you haven't already gone through the previous parts leading up to this one, ensure you;

- checkout [Part 1](https://shyxperience.hashnode.dev/simulate-your-first-lightning-transaction-on-the-bitcoin-regtest-network-part-1-macos)
- checkout [Part 2](https://shyxperience.hashnode.dev/simulate-your-first-lightning-transaction-on-the-bitcoin-regtest-network-part-2-macos)
- Have a running Bitcoin network daemon (on regtest)
- Have two active and running lightning node instances as lnd1 and lnd2

If the above prerequisites list has been satisfied then you can proceed beyond this point.

### Connect the lightning nodes to form a peer

Now that both of our nodes are ready for some action, we need to connect them to form a `peer-to-peer` network this would enable our nodes to communicate with each other on the network by opening and closing channels.

From `lnd1` run

**command:**

```zsh
lncli1 listpeers
```

**output:**

```json
{
  "peers": [] // no peers for now...let's fix that.
}
```

From `lnd2` run the command below to get its `identity_pubkey` which we would use in connecting the node to `lnd1`

**command:**

```zsh
lncli2 getinfo
```

**output:**

```json
{
    "version":  "0.17.0-beta commit=fn/v1.0.1-85-ge31d15989",
    "commit_hash":  "e31d1598932a814c2d44b774e5110746295df711",
    "identity_pubkey":  "024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579",// copy this
    "alias":  "024baa5e16c118f42cdd",
...
}
```

Now let's connect `lnd2` to `lnd1` by passing the `identity_pubkey`, `IP address` and `port`

**command:**

```zsh
 lncli1 connect 024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579@localhost:9734
```

**output:**

```json
{} // don't worry this is the correct output.
```

Let's check for the list of peers once again at `lnd1`

**command:**

```zsh
  lncli1 listpeers
```

**output:**

```json
{
    "peers":  [
        {
            "pub_key":  "024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579",
            "address":  "127.0.0.1:9734",
            "bytes_sent":  "394",
            "bytes_recv":  "394",
            "sat_sent":  "0",
            "sat_recv":  "0",
            "inbound":  false,
            "ping_time":  "-1",
            "sync_type":  "ACTIVE_SYNC",
            .....
}]}
```

Both Nodes are now successfully connected as a `peer` on our local lightning network.

### Open a payment channel between `lnd1` and `lnd2` nodes

Now let's open up a `payment channel` between `lnd1` and `lnd2`

> Quick note: A payment channel on the lightning network is a two-way connection between two parties (lightning nodes) that enables them to exchange bitcoin.

To create a payment channel between `lnd1` and `lnd2`, we'll be making use of the `pub_key` of `lnd2` and we'll then be funding the channel with `100,000 SAT`. the `100,000 SAT` is `lnd1's` contribution to the channel

**command:**

```zsh
 lncli1 openchannel 024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579 100000
```

**output:**

```json
{
  "funding_txid": "0e5e4d4f8f793cddd9370eb81140d5bc168b0e32200a3a17177d9e4da26607b0"
}
```

### Mine some blocks to increase the confirmations for the Channel opening transaction

Once the channel has been created and `lnd1's` BTC contribution has been added to the channel, we need to mine/generate some blocks to increase confirmations for the Channel opening transaction.

LND won't use a channel until its funding transaction has a few confirmations (three by default). Until then, the channel only shows up under `lncli1 pendingchannels`.

**command:**

```zsh
   bitcoin-cli -generate 10
```

**output:**

```json
{
  "address": "bcrt1q5ltxhvn3essvq5ldzpfrvlhul73auec4ekg4fk",
  "blocks": [
    "518ab6ab9e6e93ef03be2da8a8f728358cb0d4270126d1770cdb223df52f9f11",
    "01f8a4a87c0c8c98697458e42bdb459af095ba90ad9ae165ac3c31be9addf447",
    "3880fe05eaf1cff356d7b035d619704a6b93893b225fb31a938ca7275488229c",
    "11e52d12e2d7547dd1f47a44d088bd59fd73c7c436b4076c03e51d78a3fba25e",
    ...
  ]
}
```

### Check the list of channels

Run the command below to check the list of active channels and to confirm that the channel between both nodes is well-funded and ready.

**command:**

```zsh
    lncli1 listchannels
```

**output:**

```json
{
    "channels":  [
        {
            "active":  true,
            "remote_pubkey":  "024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579",
            "channel_point":  "0e5e4d4f8f793cddd9370eb81140d5bc168b0e32200a3a17177d9e4da26607b0:0",
            "chan_id":  "735573279047680",
            "capacity":  "100000",
            "local_balance":  "96530",
            "remote_balance":  "0",
            "commit_fee":  "3140",
            "commit_weight":  "772",
            "fee_per_kw":  "2500",
            "unsettled_balance":  "0",
            "total_satoshis_sent":  "0",
            "total_satoshis_received":  "0",
            ...
        }
    ]
}
```

Take a look at the balances. The channel holds 100,000 sats, but `local_balance` is only 96,530. `lnd1` opened the channel, so it pays the 3,140-sat `commit_fee`. The last 330 sats go to an anchor output, a tiny output that lets the commitment transaction be fee-bumped later if needed. Also notice that `remote_balance` is 0. All the money starts on `lnd1`'s side, so right now `lnd1` can pay `lnd2`, but `lnd2` can't pay `lnd1` anything yet.

### Create an invoice on `lnd2`

Now that we have confirmed that our channel is ready for transactions, let's go ahead and create a payable invoice for an amount of `50,000 SAT` on `lnd2`

**command:**

```zsh
   lncli2 addinvoice --amt 50000
```

**output:**

```json
{
  "r_hash": "28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e",
  "payment_request": "lnbcrt500u1pja5y06pp59re4enmeely9g5634cmzkn7mp7z5xlwlg5kp467nny38vplqadhqdqqcqzzsxqyz5vqsp58p5xllcxwt76rt7hjtjwuec2n5sfgv2t9kkc6vplrhsk2edj6ndq9qyysgqs8aysfmfz60jexh03kq27cmpym5qw8hw55wac4mqmj6t684t48u89kqaw0m7y0xuyygh5uuj95643gde3vse7zxgtu4aycfwfwk5z3cqfwmtp7",
  "add_index": "1", // total amount of invoices created.
  "payment_addr": "38686fff0672fda1afd792e4ee670a9d2094314b2dad8d303f1de16565b2d4da"
}
```

The `payment_request` is the invoice you'd normally see as a QR code, and its first few characters already tell you a lot. `ln` means lightning, `bcrt` means Bitcoin regtest, and `500u` is the amount: 500 micro-bitcoin, which is 50,000 sats.

Let's quickly inspect or confirm the details of this newly created invoice from `lnd1` by making use of the invoice's `payment_request`

**command:**

```zsh
   lncli1 decodepayreq lnbcrt500u1pja5y06pp59re4enmeely9g5634cmzkn7mp7z5xlwlg5kp467nny38vplqadhqdqqcqzzsxqyz5vqsp58p5xllcxwt76rt7hjtjwuec2n5sfgv2t9kkc6vplrhsk2edj6ndq9qyysgqs8aysfmfz60jexh03kq27cmpym5qw8hw55wac4mqmj6t684t48u89kqaw0m7y0xuyygh5uuj95643gde3vse7zxgtu4aycfwfwk5z3cqfwmtp7
```

**output:**

```json
{
    "destination":  "024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579",
    "payment_hash":  "28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e",
    "num_satoshis":  "50000",
    "timestamp":  "1708790266",
    "expiry":  "86400",
    "description":  "",
    "description_hash":  "",
    "fallback_addr":  "",
    "cltv_expiry":  "80",
    "route_hints":  [],
    "payment_addr":  "38686fff0672fda1afd792e4ee670a9d2094314b2dad8d303f1de16565b2d4da",
    "num_msat":  "50000000",
    "features":  {
        "8":  {
            "name":  "tlv-onion",
            "is_required":  true,
            "is_known":  true
        },
        ...
    }
    ...
}
```

### Pay or satisfy the invoice created by `lnd2` via `lnd1`

Time for `lnd1` to pay up

Run the command below and follow the prompts to confirm payment

**command:**

```zsh
  lncli1 payinvoice lnbcrt500u1pja5y06pp59re4enmeely9g5634cmzkn7mp7z5xlwlg5kp467nny38vplqadhqdqqcqzzsxqyz5vqsp58p5xllcxwt76rt7hjtjwuec2n5sfgv2t9kkc6vplrhsk2edj6ndq9qyysgqs8aysfmfz60jexh03kq27cmpym5qw8hw55wac4mqmj6t684t48u89kqaw0m7y0xuyygh5uuj95643gde3vse7zxgtu4aycfwfwk5z3cqfwmtp7
```

**output:**

```md
Payment hash: 28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e
Description:
Amount (in satoshis): 50000
Fee limit (in satoshis): 2500
Destination: 024baa5e16c118f42cddd95fd828b32fa8dfcfea55fa384e89e2da0f3b96fc4579
Confirm payment (yes/no): yes
+------------+--------------+--------------+--------------+-----+----------+-----------------+----------------------+
| HTLC_STATE | ATTEMPT_TIME | RESOLVE_TIME | RECEIVER_AMT | FEE | TIMELOCK | CHAN_OUT | ROUTE |
+------------+--------------+--------------+--------------+-----+----------+-----------------+----------------------+
| SUCCEEDED | 0.086 | 0.565 | 50000 | 0 | 761 | 735573279047680 | 024baa5e16c118f42cdd |
+------------+--------------+--------------+--------------+-----+----------+-----------------+----------------------+
Amount + fee: 50000 + 0 sat
Payment hash: 28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e
Payment status: SUCCEEDED, preimage: 071ae531770166a26d7b42a82a6215776a4f90f1e9935c1f225077158b3782c9
```

How does `lnd1` know the payment really arrived? When `lnd2` created the invoice, it made up a secret called the preimage and put its hash, the `r_hash`, in the invoice. The payment only completes once `lnd2` reveals the preimage, and anyone can check that hashing it gives the `r_hash`. The preimage works as the receipt.

The fee was 0 because the payment went straight through our own channel, with no other nodes to pay along the way. Nothing touched the blockchain either. `lnd1`'s side of the channel dropped by 50,000 sats and `lnd2`'s side went up by 50,000.

Now that we have made payment, let's look up the invoice on `lnd2` we'll be using the `payment hash` above

**command:**

```zsh
    lncli2 lookupinvoice 28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e
```

**output:**

```json
 {
    "memo":  "",
    "r_preimage":  "071ae531770166a26d7b42a82a6215776a4f90f1e9935c1f225077158b3782c9",
    "r_hash":  "28f35ccf79cfc8545351ae362b4fdb0f85437ddf452c1aebd399227607e0eb6e",
    "value":  "50000",
    "value_msat":  "50000000",
    "settled":  true,// the invoice has been fulfilled
    "creation_date":  "1708790266",
    "settle_date":  "1708793723",
    "payment_request":  "lnbcrt500u1pja5y06pp59re4enmeely9g5634cmzkn7mp7z5xlwlg5kp467nny38vplqadhqdqqcqzzsxqyz5vqsp58p5xllcxwt76rt7hjtjwuec2n5sfgv2t9kkc6vplrhsk2edj6ndq9qyysgqs8aysfmfz60jexh03kq27cmpym5qw8hw55wac4mqmj6t684t48u89kqaw0m7y0xuyygh5uuj95643gde3vse7zxgtu4aycfwfwk5z3cqfwmtp7",
    "description_hash":  "",
    "expiry":  "86400",
    "fallback_addr":  "",
    "cltv_expiry":  "80",
    "route_hints":  [],
...
}
```

### Close the payment channel, kill both Bitcoin and lightning daemons

_"Eventually, all good things must come to an end"_

We are finally done with all lightning transactions so let's go ahead and close the payment channel and kill all running lightning and Bitcoin daemons as we close for the day.

Run the command below from any of the two nodes

**command:**

```zsh
    lncli2 closeallchannels
```

**output:**

```json
{
  "remote_pub_key": "02e0810d836946eebaeb9186c45ebb0c2d17294890ec67f488c9127de4d51dd3cd",
  "channel_point": "0e5e4d4f8f793cddd9370eb81140d5bc168b0e32200a3a17177d9e4da26607b0:0",
  "closing_txid": "9b9e88eec1d3708d97588088e6e40cd0d3cc65163baabb5215445785268f5b78",
  "error": ""
}
```

Closing is the one step that goes back on-chain. The `closing_txid` is a normal Bitcoin transaction that pays each node its final balance. `lnd2` gets its 50,000 sats, and `lnd1` gets the rest of the 100,000, minus the closing fee. Until a block confirms it, the channel sits under `pendingchannels` as waiting to close.

Check the list of channels

**command:**

```zsh
    lncli2 listchannels
```

**output:**

```json
{
  "channels": [] // no active channel
}
```

Run the command below to kill all running daemons

**command:**

```zsh
     lncli1 stop && lncli2 stop && bitcoin-cli stop
```

**output:**

```zsh
Bitcoin Core stopping
```

### Conclusion

Finally, an amazing end to a three part journey.

I hope this article helped you gain some hands-on experience in working with the lightning network. Remember, Lightning payments are faster, cheaper and less complex than the conventional bitcoin transaction. see you in the next one
