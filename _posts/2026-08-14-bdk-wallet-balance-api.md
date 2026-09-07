---
layout: post
title: Modifying the bdk_wallet Balance API for My Summer of Bitcoin Project
date: 2026-08-14
description: How I changed the way bdk_wallet decides which unconfirmed coins count as trusted, by following where the money came from instead of which keychain it landed on.
tags: ["Bitcoin", "BDK", "Rust", "Wallets", "Summer of Bitcoin", "Open Source"]
categories:
---

Let me start with a brief story.

Imagine you want to buy a Coke in a supermarket that accepts bitcoin. You just started using a wallet built on the BDK libraries, and half an hour ago someone sent you a few thousand sats. The transaction is still unconfirmed, but the balance says you can spend it.

So you grab the bottle, walk to the till, scan the QR code and... the payment fails. Not enough funds.

What happened is that the sender replaced their transaction with a higher-fee one ([RBF](https://bitcoinops.org/en/topics/replace-by-fee/)) that no longer pays you. The original never confirmed, so the coin you were about to spend never existed. The supermarket is fine, the network is fine, the sender did nothing the protocol forbids. The only thing that failed is how your wallet classified that coin.

That is what I worked on this summer for [Summer of Bitcoin](https://www.summerofbitcoin.org/): fixing how BDK decides which of your unconfirmed funds you can actually trust. It closes two long-standing bugs in bdk_wallet:

- [#16](https://github.com/bitcoindevkit/bdk_wallet/issues/16): the wallet trusted any unconfirmed coin that landed on its change keychain. But nothing stops a stranger from paying to a change address they spotted on-chain, and that money was counted as trusted even though the sender could still double-spend it.
- [#273](https://github.com/bitcoindevkit/bdk_wallet/issues/273): the mirror image. Consolidating your own coins into a receive address was marked untrusted, while the same transaction sent to a change address was trusted. Same wallet, same coins, different bucket.

## BDK Balance

BDK reports your balance as four numbers:

- **Confirmed**: money already in a block.
- **Immature**: freshly mined coins the protocol locks for 100 blocks.
- **Trusted pending**: unconfirmed money the wallet is fairly sure will go through.
- **Untrusted pending**: unconfirmed money someone else could still pull back.

The last two are the interesting ones. Pending means the transaction isn't in a block yet, so it could still be replaced before it confirms.

### Old Logic

The old code decided trust from the keychain the coins landed on. The whole rule was one closure:

```rust
|&(k, _), _| k == KeychainKind::Internal
```

Which means: "did this land on a change address? then trust it."  

The point here is that trust has nothing to do with which of your addresses received the coin. It depends on where the money came from, so the correct approach would be to check who actually sent these coins.  

### Ancestry-Based Trust

To classify the coins correctly, that classification should be determined by the ancestry of the coin.

The main assumption would be: *only trust owned inputs*.  

| Case | Condition | Why |
| ---- | --------- | --- |
| **Trusted** | The coin's whole unconfirmed history only spends coins you own | You made every one of those transactions, so no outsider can replace them |
| **Untrusted** | Somewhere in that history it pulls in coins that aren't yours | Whoever controls those coins can still replace the transaction |
| **Unknown** | The wallet can't see far enough back | A history you can't verify isn't safe, so it falls back to untrusted |

That was difficult on two counts: first to arrive at this idea, and second to actually implement it in the code. We thought it could be a good idea to have it folded in the wallet, but we noticed there were some useful primitives to perform the walk in chain. Unfortunately we could not use them, so we ended up doing something "totally aside" from the project: a new API in `bdk_chain` that let us run complex closures over the chain's balance function, erasing generics and giving a clearer API to work with.

Next I'll explain every change I made in the chain layer and the wallet layer.

#### In `bdk_chain` ([#2246](https://github.com/bitcoindevkit/bdk/pull/2246))

I added `classify_outpoints`, which labels each coin with its spend eligibility. It walks back through the transactions that funded a coin and stops as soon as it hits something confirmed or tainted, memoizing what it has seen so shared history is never walked twice.  

Eligibility is the vocabulary that walk speaks. Every unspent output comes back as one of three things: `Settled`, when its chain position satisfies `is_settled`; `Immature`, when it is a coinbase output that has not aged its 100 blocks; or `Unsettled(Trust)`, when it is still pending. That last one carries the verdict from the ancestry walk, `Trust::Trusted` when the whole unconfirmed history is yours and `Trust::Untrusted` when anything in it is tainted or missing. Balance stops being a special case and becomes a tally over those labels, which is what leaves room for new buckets later.

The chain layer doesn't decide what counts as tainted, or as final. It takes two predicates:

- `does_taint(&tx)`: should this transaction be considered tainted?
- `is_settled(&pos)`: do we consider this chain position final?

Both of them replace something that was there before. `balance` used to take a `trust_predicate` and a `min_confirmations` number. The old predicate only saw an output's keychain and index, so it could tell you where a coin landed but never where it came from. `does_taint` takes the whole transaction instead, which is what makes it possible to look at the inputs. `min_confirmations` used to be a fixed number too.

While I was in there I also took a generic out. `balance` used to work over `(identifier, outpoint)` pairs, which dragged a type parameter through the signature without buying much, and it now takes plain `OutPoint`s. Together with the memoization, which guarantees `does_taint` runs at most once per transaction no matter how many of your coins trace back through it, the function ended up both simpler to call and cheaper to run than the one I set out to patch.

##### A Small Primitive ([#2263](https://github.com/bitcoindevkit/bdk/pull/2263))

Out of the confirmation logic came `ChainPosition::blocks_since_conf`, which returns how many blocks sit on top of a confirmed position. A tiny helper that avoids the usual off-by-one, reused for confirmation thresholds and relative timelocks.

#### In `bdk_wallet` ([#431](https://github.com/bitcoindevkit/bdk_wallet/pull/431))

The wallet is the only layer that knows which coins are yours, so it is the one that supplies `does_taint`. Its rule is one line: a transaction is tainted if any of its inputs spends an output that none of the wallet's descriptors produced.

`Wallet::balance` takes a `min_confirmations` argument now, and that is the breaking part of the signature. Before counting anything, BDK has to pick which of several competing versions of a transaction is the canonical one, and that decision now starts from `tip - min_confirmations` instead of from the chain tip. A coin that has not cleared your threshold is therefore not treated as confirmed at any point, including when the ancestry walk is deciding whether it can stop.

Finally, the wallet folds `classify_outpoints` itself rather than calling `bdk_chain`'s `balance`. The numbers come out the same, but folding it here means the wallet can add buckets that mean nothing to the chain layer, which is exactly what the next pull request needed.

##### Locked coins ([#538](https://github.com/bitcoindevkit/bdk_wallet/pull/538))

A confirmed coin can still be unspendable if its descriptor carries a timelock that has not matured yet, and until now the balance counted it as `confirmed` and overstated what you could actually move. So there is a `locked` bucket now, and `Wallet::balance` returns a `WalletBalance` to carry it.

A timelock lives in the descriptor, which is a wallet concept, so the check stays in the wallet. It folds `classify_outpoints` and reroutes the outputs that hold to `locked`, leaving `bdk_chain` timelock-agnostic.

Two limits worth saying out loud. Only height-based locks are handled, because time-based ones need median-time-past and BDK does not track it yet ([#183](https://github.com/bitcoindevkit/bdk_wallet/issues/183)). And a descriptor with several spending paths gets a default satisfaction rather than a full policy analysis, so a coin that is spendable through some other branch can still show up as locked.

## Memories from a newbie in open-source contribution

Maybe the biggest lesson of the summer is that a good fix is rarely the *first* fix. This one went through several complete redesigns in review with the BDK maintainers and other contributors.

It started as a self-contained walk inside the wallet, then built on an earlier chain-layer effort ([#2235](https://github.com/bitcoindevkit/bdk/pull/2235)), and through discussion it ended up as a smaller, faster, memoized walk that only ever inspects your unconfirmed coins and their ancestors, so it doesn't get slower as your wallet history grows.

A lot of the value came from other people poking holes in the approach: performance concerns, edge cases like missing history, and questions about what "trust" should even mean. Contributing to open source is much less about writing the patch than about defending, breaking and rebuilding it in public.

## Where it stands

The trust redesign and the wallet delegation are open and under review, the confirmation primitive is open, and the `locked` category is open as a draft follow-up. The one hard dependency is release ordering: `bdk_wallet` builds against a published `bdk_chain`, so the wallet pieces are gated on the chain change shipping first. Next up are time-based timelocks and a future frozen/reserved category for coins the user locks manually.

## What else?

The PRs and issues behind this project:

- [bdk_wallet#431](https://github.com/bitcoindevkit/bdk_wallet/pull/431) was my first attempt, a walk directly in the wallet, later reworked to delegate to the chain walk once #2246 existed, and gained `min_confirmations`.
- [bdk#2246](https://github.com/bitcoindevkit/bdk/pull/2246) is the ancestry-based trust and eligibility in `bdk_chain`, which then sent me back to rework #431 on top of it.
- [bdk#2263](https://github.com/bitcoindevkit/bdk/pull/2263) is the `blocks_since_conf` primitive.
- [bdk_wallet#538](https://github.com/bitcoindevkit/bdk_wallet/pull/538) is the `locked` balance category.

The issues that framed the work: [#16](https://github.com/bitcoindevkit/bdk_wallet/issues/16) and [#273](https://github.com/bitcoindevkit/bdk_wallet/issues/273) (the trust bugs), [#180](https://github.com/bitcoindevkit/bdk_wallet/issues/180) (locked coins), and [#183](https://github.com/bitcoindevkit/bdk_wallet/issues/183) (median-time-past, needed for time-based timelocks).

<sub>*Thanks to [nymius](https://github.com/nymius) for mentoring me through all of this, and to [Evan](https://github.com/evanlinjin) for the groundwork in [#2235](https://github.com/bitcoindevkit/bdk/pull/2235) and for sitting through round after round of review.*</sub>
