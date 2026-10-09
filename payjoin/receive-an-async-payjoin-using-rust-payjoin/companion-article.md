# Objective

The goal of this article is to help you

- Build a small Rust app that receives an async Payjoin (BIP 77) using the Payjoin Dev Kit (PDK)
- Understand each state the receiver moves through, and why the library makes you save every step
- Test the app end to end on regtest, with `payjoin-cli` playing the sender
- Know what changed between the original PDK receiver demo and `payjoin` 1.2.0

# Requirements

- Some understanding of how async Payjoin works. This is a companion to [Payjoin Part 2 — BIP 77: Payjoin Without a Server](../payjoin-part-2-bip-77-async-payjoin-v2.md), so read that first
- [Nigiri](https://github.com/vulpemventures/nigiri) and Docker. Nigiri runs regtest Bitcoin Core on port `18443` (login `admin1` / `123`), so stop any other bitcoind using that port
- Rust installed and configured on your machine
- An internet connection, since the app uses the public `payjo.in` directory even on regtest

# Introduction

In early 2025, spacebear built [pdk-receiver-demo](https://github.com/spacebear21/pdk-receiver-demo), a single-file app that receives an async Payjoin using a Bitcoin Core wallet. It was written against a pre-1.0 version of the library, and if you copy it into a new project today, it won't compile.

In this article we'll rebuild it step by step against `payjoin` 1.2.0. Every code block comes from a project I compiled and ran against the real directory, so pasting them in order gives you a working receiver.

# Setting up

First, start Nigiri, create two wallets, and fund them. The receiver needs a coin of its own, because contributing an input is what makes a Payjoin. `nigiri rpc` runs `bitcoin-cli` inside the container.

A freshly mined coin can't be spent until 100 more blocks sit on top of it, so we mine 101 blocks to the sender, send the receiver 0.5 BTC, then mine one more block to confirm it.

**command:**

```zsh
nigiri start
nigiri rpc createwallet receiver
nigiri rpc createwallet sender
nigiri rpc generatetoaddress 101 $(nigiri rpc --rpcwallet=sender getnewaddress)
nigiri rpc --rpcwallet=sender sendtoaddress $(nigiri rpc --rpcwallet=receiver getnewaddress) 0.5
nigiri rpc generatetoaddress 1 $(nigiri rpc --rpcwallet=sender getnewaddress)
```

> Quick note: if these wallets already exist from earlier Nigiri use, run `nigiri rpc loadwallet` with each name instead of `createwallet`.

Now bootstrap the project. `payjoin` is the PDK, and its `io` feature fetches the directory's keys for us. `corepc-client` talks to Bitcoin Core, replacing the unmaintained `bitcoincore-rpc` crate the original demo used. `reqwest` sends HTTP requests, and `tokio` runs them.

**command:**

```zsh
cargo new pdk-receiver-demo && cd pdk-receiver-demo
cargo add payjoin@1.2.0 --features io
cargo add corepc-client@0.17 --features client-sync
cargo add reqwest@0.13
cargo add tokio@1 --features full
```

Open `src/main.rs`, delete what's there, and start with the imports. The two constants are the OHTTP relay and the Payjoin directory we'll use.

```rust
use std::time::Duration;

use corepc_client::client_sync::v29::Client;
use corepc_client::client_sync::Auth;
use payjoin::bitcoin::consensus::encode::serialize_hex;
use payjoin::bitcoin::psbt::Input;
use payjoin::bitcoin::{Address, Amount, Network, OutPoint, Script, TxIn, TxOut};
use payjoin::persist::{InMemoryPersister, OptionalTransitionOutcome};
use payjoin::receive::v2::{ReceiverBuilder, SessionEvent};
use payjoin::receive::InputPair;
use payjoin::ImplementationError;

const OHTTP_RELAY: &str = "https://pj.benalleng.com";
const DIRECTORY: &str = "https://payjo.in";
```

# Step 1: Connect to the receiver's wallet

`main` connects to the `receiver` wallet and asks for a fresh address to be paid at. `#[tokio::main]` makes `main` async, so we can `.await` network calls. Everything up to step 7 goes inside `main`.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Connect to the "receiver" wallet on a regtest bitcoind
    let bitcoind = Client::new_with_auth(
        "http://localhost:18443/wallet/receiver",
        Auth::UserPass("admin1".into(), "123".into()),
    )?;
    let address = bitcoind.fresh_address()?;
```

`fresh_address` is one of four helpers we'll write in step 8, so ignore your editor's complaints until then.

# Step 2: Start a session

We fetch the directory's OHTTP keys through the relay, so the directory never learns our IP address.

This is also the biggest change from the original demo. Every state transition now returns a value you must save to a **persister** before you get the next state. The persister records each step as an event, so a real wallet can restart mid-session and replay the events to pick up where it left off. The library's `InMemoryPersister` keeps events in memory, which is fine for a demo, but a crash would lose the session.

```rust
    // Fetch the directory's OHTTP keys through the relay
    let ohttp_keys = payjoin::io::fetch_ohttp_keys(OHTTP_RELAY, DIRECTORY).await?;

    // Every state transition is saved to this persister before we can use the next state
    let persister = InMemoryPersister::<SessionEvent>::default();

    // Initialize the receiver session and print the BIP21 URI
    let mut receiver = ReceiverBuilder::new(address, DIRECTORY, ohttp_keys)?
        .with_amount(Amount::from_sat(1_000_000))
        .build()
        .save(&persister)?;
    println!("Payjoin URI:\n{}", receiver.pj_uri());
```

The amount now goes on the builder, rather than being set on the URI afterwards.

# Step 3: Wait for the sender

Next we poll the directory with OHTTP-encrypted requests through the relay. Each saved response is either `Progress`, meaning the sender's Original PSBT has arrived, or `Stasis`, meaning nothing yet, so we keep the same receiver and try again.

```rust
    // Poll the directory until the sender's Original PSBT arrives
    let http = reqwest::Client::new();
    let proposal = loop {
        println!("Polling receive request...");
        let (req, ctx) = receiver.create_poll_request(OHTTP_RELAY)?;
        let res = http
            .post(req.url)
            .header("Content-Type", req.content_type)
            .body(req.body)
            .send()
            .await?;
        match receiver.process_response(&res.bytes().await?, ctx).save(&persister)? {
            OptionalTransitionOutcome::Progress(proposal) => break proposal,
            OptionalTransitionOutcome::Stasis(same_receiver) => receiver = same_receiver,
        }
    };
```

# Step 4: Check the sender's proposal

Before adding anything, the receiver runs the BIP 78 checklist. The PDK enforces the order: each check returns a new type that only offers the next check. This typestate pattern makes skipping a step a compile error.

```rust
    // Validate the sender's proposal
    let proposal = proposal
        .check_broadcast_suitability(None, |tx| {
            let result = bitcoind.test_mempool_accept(&[tx.clone()]).map_err(ImplementationError::new)?;
            Ok(result.0.first().is_some_and(|r| r.allowed))
        })
        .save(&persister)?;

    println!(
        "Original PSBT received! Broadcast this fallback transaction if the Payjoin fails:\n{}",
        serialize_hex(&proposal.extract_tx_to_schedule_broadcast())
    );

    let proposal = proposal
        .check_inputs_not_owned(&mut |outpoint| bitcoind.owns_outpoint(outpoint))
        .save(&persister)?;
    let proposal = proposal.check_no_inputs_seen_before(&mut |_| Ok(false)).save(&persister)?;
    let proposal = proposal
        .identify_receiver_outputs(&mut |script| bitcoind.owns_script(script))
        .save(&persister)?;
```

In order, these checks:

- **Test the Original:** `testmempoolaccept` confirms it's broadcastable. We print it, because if the sender disappears we can broadcast it ourselves and still get paid.
- **Inputs not ours:** none of the sender's inputs should belong to us. This check now hands us an `OutPoint` instead of a script, so `owns_outpoint` looks up the funding transaction first.
- **Inputs not seen before:** this guards against probing. Returning `false` every time switches the protection off, so a real wallet must remember every input it has seen.
- **Find our outputs:** we tell the library which outputs pay us.

# Step 5: Add our coin

Now we turn the payment into a Payjoin. We keep the outputs, let the library pick a coin that preserves privacy, and add it as an input. `apply_fee_range` is new: it used to live in the finalize step, and `None, None` accepts the defaults.

```rust
    // Augment the proposal: keep our outputs, add one of our inputs, set the fee range
    let proposal = proposal.commit_outputs().save(&persister)?;
    let candidate_inputs = bitcoind.input_pairs()?;
    let selected_input = proposal.try_preserving_privacy(candidate_inputs)?;
    let proposal = proposal
        .contribute_inputs(vec![selected_input])?
        .commit_inputs()
        .save(&persister)?;
    let proposal = proposal.apply_fee_range(None, None).save(&persister)?;
```

# Step 6: Sign and reply

`finalize_proposal` hands us the PSBT for `walletprocesspsbt` to sign, and we post the encrypted result back to the directory for the sender.

```rust
    // Sign our input and send the Payjoin proposal back to the sender
    let proposal = proposal
        .finalize_proposal(|psbt| {
            let signed = bitcoind.wallet_process_psbt(psbt).map_err(ImplementationError::new)?;
            signed.into_model().map(|m| m.psbt).map_err(ImplementationError::new)
        })
        .save(&persister)?;

    let (req, ctx) = proposal.create_post_request(OHTTP_RELAY)?;
    let res = http
        .post(req.url)
        .header("Content-Type", req.content_type)
        .body(req.body)
        .send()
        .await?;
    let txid = proposal.psbt().clone().extract_tx_unchecked_fee_rate().compute_txid();
    let mut monitor = proposal.process_response(&res.bytes().await?, ctx).save(&persister)?;
    println!("Payjoin proposal sent. Waiting for the sender to broadcast {txid}...");
```

# Step 7: Watch for the transaction

The original demo stopped after replying. Version 1.2.0 adds a `Monitor` state that watches until the sender broadcasts the Payjoin or the fallback.

```rust
    // Watch our wallet until the Payjoin (or the fallback) shows up
    loop {
        let outcome = monitor
            .check_for_transaction(|txid| match bitcoind.get_transaction(txid) {
                Ok(tx) => Ok(Some(tx.into_model().map_err(ImplementationError::new)?.tx)),
                Err(_) => Ok(None),
            })
            .save(&persister)?;
        match outcome {
            OptionalTransitionOutcome::Progress(()) => break,
            OptionalTransitionOutcome::Stasis(same_monitor) => monitor = same_monitor,
        }
        tokio::time::sleep(Duration::from_secs(2)).await;
    }
    println!("Payjoin transaction found in the mempool. Done!");

    Ok(())
}
```

# Step 8: The wallet helpers

Finally, add these helpers below `main`. A trait lets us add methods to `corepc-client`'s `Client`, and `ImplementationError` carries our own errors back through the PDK's state machine.

```rust
/// Small helpers that answer the receiver's questions using bitcoind
trait ReceiverWallet {
    fn fresh_address(&self) -> Result<Address, Box<dyn std::error::Error>>;
    fn owns_script(&self, script: &Script) -> Result<bool, ImplementationError>;
    fn owns_outpoint(&self, outpoint: &OutPoint) -> Result<bool, ImplementationError>;
    fn input_pairs(&self) -> Result<Vec<InputPair>, Box<dyn std::error::Error>>;
}

impl ReceiverWallet for Client {
    fn fresh_address(&self) -> Result<Address, Box<dyn std::error::Error>> {
        Ok(self.get_new_address(None, None)?.into_model()?.0.assume_checked())
    }

    fn owns_script(&self, script: &Script) -> Result<bool, ImplementationError> {
        let Ok(address) = Address::from_script(script, Network::Regtest) else {
            return Ok(false);
        };
        let info = self.get_address_info(&address).map_err(ImplementationError::new)?;
        Ok(info.is_mine)
    }

    fn owns_outpoint(&self, outpoint: &OutPoint) -> Result<bool, ImplementationError> {
        // If our wallet doesn't know the funding transaction, the coin can't be ours
        let Ok(funding) = self.get_transaction(outpoint.txid) else {
            return Ok(false);
        };
        let tx = funding.into_model().map_err(ImplementationError::new)?.tx;
        match tx.output.get(outpoint.vout as usize) {
            Some(txout) => self.owns_script(&txout.script_pubkey),
            None => Ok(false),
        }
    }

    fn input_pairs(&self) -> Result<Vec<InputPair>, Box<dyn std::error::Error>> {
        let unspent = self.list_unspent()?.into_model()?.0;
        let pairs = unspent
            .into_iter()
            .map(|utxo| {
                let txin = TxIn {
                    previous_output: OutPoint { txid: utxo.txid, vout: utxo.vout },
                    ..Default::default()
                };
                let psbtin = Input {
                    witness_utxo: Some(TxOut {
                        value: utxo.amount,
                        script_pubkey: utxo.script_pubkey,
                    }),
                    ..Default::default()
                };
                InputPair::new(txin, psbtin, None).expect("bitcoind gives us valid inputs")
            })
            .collect();
        Ok(pairs)
    }
}
```

# Running the demo

Start the receiver.

**command:**

```zsh
cargo run
```

**output:**

```zsh
Payjoin URI:
bitcoin:bcrt1q7qx7p0t5uh4djz0gdx0hd8u07gcm288sd3qzak?amount=0.01&pjos=0&pj=HTTPS://PAYJO.IN/0STGRM66T5N3G%23EX1UADVV6S-OH1QYPNNJTQFK00ZQ99VKKNGF6M90PKRRPYY6QJHDNC0WLHWR9V74X4P9C-RK1Q2VJQZEWA0LZ7V6J7RES92G5YXWZH03HHNJCAD4JT22SAMGENY9N7
Polling receive request...
```

In a second terminal, install `payjoin-cli` as the sender and create a `config.toml` pointing at the `sender` wallet.

**command:**

```zsh
cargo install payjoin-cli --version 1.0.0
mkdir sender && cd sender
```

```toml
[bitcoind]
rpcuser = "admin1"
rpcpassword = "123"
rpchost = "http://localhost:18443/wallet/sender"

[v2]
pj_directories = ["https://payjo.in", "https://lets.payjo.in"]
ohttp_relays = ["https://pj.benalleng.com", "https://pj.bobspacebkk.com", "https://payjoin.achow101.com"]
```

Paste the receiver's URI into the send command.

**command:**

```zsh
payjoin-cli send "<PAYJOIN URI>" --fee-rate 2
```

**output:**

```zsh
[Sender   01a111ac-beae-70d4-8a3c-816bd8f87144] Session established
[Sender   01a111ac-beae-70d4-8a3c-816bd8f87144] Posted Original PSBT...
[Sender   01a111ac-beae-70d4-8a3c-816bd8f87144] Proposal received. Processing...
[Sender   01a111ac-beae-70d4-8a3c-816bd8f87144] Payjoin sent. TXID: 6f01da2cbbe54ccfdfe8de92eae41288197d5290b1f1042693dc66cf24f1b27f
```

Back in the first terminal, the receiver finishes on its own.

**output:**

```zsh
Original PSBT received! Broadcast this fallback transaction if the Payjoin fails:
02000000000101ffc59b311a04ee26619ad68a2619b0a94a09f2b682fbc3c42e...
Payjoin proposal sent. Waiting for the sender to broadcast 6f01da2cbbe54ccfdfe8de92eae41288197d5290b1f1042693dc66cf24f1b27f...
Payjoin transaction found in the mempool. Done!
```

Look the transaction up with `nigiri rpc getrawtransaction <TXID> 1` and you'll see two inputs, one from each wallet. The receiver's output is 0.51 BTC: the 0.01 BTC payment plus its own 0.5 BTC coin coming back.

# What changed since the original demo

```text
Original demo (pre-1.0)                     payjoin 1.2.0
──────────────────────────────────────────  ──────────────────────────────────────────
Receiver::new(addr, dir, keys, None)        ReceiverBuilder::new(addr, dir, keys)?
                                              .with_amount(..).build().save(&persister)?
pj_uri.amount = Some(..)                    .with_amount(..) on the builder
no persistence                              every step returns a transition to .save()
extract_req / process_res -> Option         create_poll_request / process_response
                                              -> Progress or Stasis
check_inputs_not_owned(|script| ..)         check_inputs_not_owned(&mut |outpoint| ..)
InputPair::new(txin, psbtin)                InputPair::new(txin, psbtin, None)
finalize_proposal(sign, min_fee, max_fee)   apply_fee_range(min, max), then
                                              finalize_proposal(sign)
extract_v2_req                              create_post_request
session ends after replying                 Monitor state: check_for_transaction
bitcoincore-rpc                             corepc-client
```

# Conclusion

The receiver flow is the one the BIPs describe: wait for the Original, check it, add a coin, sign, and reply. PDK 1.2.0 adds structure around it. Persistence lets a session survive a restart, and `Monitor` tells the receiver how the session ended. When you're done, `nigiri stop` shuts everything down. To go further, swap `InMemoryPersister` for a database, as `payjoin-cli` does with SQLite, and track seen inputs so the probing check means something.

# Useful links

- [spacebear21/pdk-receiver-demo](https://github.com/spacebear21/pdk-receiver-demo)
- [payjoin/rust-payjoin](https://github.com/payjoin/rust-payjoin)
- [payjoin crate documentation](https://docs.rs/payjoin/1.2.0)
- [payjoin-cli](https://github.com/payjoin/rust-payjoin/tree/master/payjoin-cli)
- [BIP 77: Async Payjoin](https://github.com/bitcoin/bips/blob/master/bip-0077.md)
