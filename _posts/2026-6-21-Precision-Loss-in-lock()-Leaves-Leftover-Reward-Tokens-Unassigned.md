---
title: "Precision Loss in lock() Leaves Leftover Reward Tokens Unassigned"
date: 2026-06-21
categories: [Bug Bounty]
tags: [solidity, "smart contracts", "precision loss"]
---

# Precision Loss in `lock()` Leaves Leftover Reward Tokens Unassigned

## Brief

A precision loss in the division operation inside the VestingMaster.lock() function leads to permanent freezing of the remaining funds, making them unclaimable.

## Vulnerability Details

In VestingMaster.sol, the lock() function receives an amount of reward tokens to lock for a user. It divides this amount by lockedPeriodAmount to compute lockedAmount per period

```solidity
locked: amount / lockedPeriodAmount,
```

However, this division operation performs integer division and discards the remainder (amount % lockedPeriodAmount). The loop then creates lockedPeriodAmount vesting entries, each containing lockedAmount.

The remaining dust amount (amount % lockedPeriodAmount) is never added to any vesting period and cannot be claimed by the user. Meanwhile, the global state variable totalLockedRewards is increased by the full amount (totalLockedRewards = totalLockedRewards.add(amount)).

## Scenario (included in PoC)

lockedPeriodAmount is set to 59 (as in the deployed contract).

A farming pool calls VestingMaster.lock(user, 1005) to lock 1005 reward tokens.

lockedAmount is computed as:

1005 / 59 = 17

The loop pushes 59 entries, each with locked = 17. The user's total vesting amount is recorded as:

59 * 17 = 1003

totalLockedRewards is increased by 1005.

The 2 remainder tokens (1005 % 59 = 2) are never assigned to the user.

When the user claims all rewards over time, they receive 1003 tokens.

The 2 remainder tokens are trapped in the contract, and totalLockedRewards remains at 2 instead of 0.

## Impact Details

Permanently trapped reward tokens for users.
The totalLockedRewards counter becomes overstated over time since it includes unassignable remainder tokens.
