---
title: "Attacker bypass of the PriceVelocityGuard circuit breaker"
date: 2026-07-22
categories: [Bug Bounty]
tags: [solidity, "smart contracts", oracle, "circuit breaker"]
---

# Attacker bypass of the PriceVelocityGuard circuit breaker

## Summary

The PriceVelocityGuard extension is designed to protect the pool from market crashes by limiting the price drop per block to a percentage set by the admin. However, the wrong implementation will cause a bypass of this protection for the pool as an attacker will execute many small swaps in the same block.

## Root Cause

In `PriceVelocityGuardExtension.sol` lines 57-58:

```solidity
s.lastMidPriceX64 = midPrice;
```

The extension updates `lastMidPriceX64` after every successful swap, causing subsequent swaps within the same block to validate against the previous swap's price instead of the block's opening reference price. This effectively enforces a per-swap limit rather than the intended per-block limit.

## Internal Pre-conditions

- Admin needs to call `setMaxChangePerBlock()` to set the velocity limit (e.g., to 2%).
- The pool must have `PriceVelocityGuardExtension` active in its `beforeSwap` hook.

## External Pre-conditions

- The pool must be using a pull-based oracle that allows submitting intermediate price attestations alongside swap transactions.
- The external market price experiences a significant crash.

## Attack Path

1. The admin sets the max change per block limit to 2%.
2. Token X crashes from $100 to $50.
3. The attacker submits a $98 price update (a 2% drop) with a tiny swap. The Guard passes and resets the anchor to $98.
4. In the same block, the attacker submits a $96 price update with another tiny swap. The Guard passes and resets the anchor to $96.
5. The attacker chains these small steps until reaching $50, then executes a massive swap to buy the pool's remaining Token X at the fully crashed price.

## Impact

## Mitigation

The anchor must only be updated when moving to a new block. This ensures that all subsequent swaps within the same block are evaluated against the same starting price, strictly enforcing the block-level velocity constraint.
