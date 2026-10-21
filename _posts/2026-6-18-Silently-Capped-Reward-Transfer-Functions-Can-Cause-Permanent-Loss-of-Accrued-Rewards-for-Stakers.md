---
title: "Silently Capped Reward Transfer Functions Can Cause Permanent Loss of Accrued Rewards for Stakers"
date: 2026-06-18
categories: [Bug Bounty]
tags: [solidity, "smart contracts", "staking", "rewards"]
---

# Silently Capped Reward Transfer Functions Can Cause Permanent Loss of Accrued Rewards for Stakers

## Brief

This vulnerability affects both LiquidityMiningMaster and LiquidityMiningMaster V1.1 deployed contracts. Both contracts distribute mining rewards to users during deposit and withdraw calls, and both implement a silent capping mechanism that can cause permanent reward loss.

## Vulnerability Details

### LiquidityMiningMaster.sol
The contract caps the reward transfer via the safeTokenTransfer function to prevent transferring staked tokens. It calculates the safe withdrawable reward balance as balance - totalSupply(). If the user's pending reward _amount exceeds this available balance, safeTokenTransfer silently caps the transfer amount to the available balance (balance - totalSupply()) and proceeds with the transfer.

```solidity
    function safeTokenTransfer(address _to, uint256 _amount)
        internal
        returns (uint256)
    {
        uint256 balance = tokenBalance();
        uint256 amount;
        uint256 floorAmount = totalSupply();
        if (balance > floorAmount) {
            if (_amount > balance.sub(floorAmount)) {
                amount = balance.sub(floorAmount);
            } else {
                amount = _amount;
            }
        }
```

### LiquidityMiningMaster v1.1
The v1.1 contract has the same vulnerability in safeTokenTransfer. It calculates the safe withdrawable reward balance as rewardToken.balanceOf(address(this)) - floorAmount, where floorAmount = rewardToken == stakedToken() ? totalSupply() : 0. If the user's pending reward exceeds this available balance, safeTokenTransfer silently caps the transfer and proceeds.

```solidity
   function safeTokenTransfer(address _to, uint256 _amount)
        internal
        virtual
        returns (uint256)
    {
        uint256 balance = rewardToken.balanceOf(address(this));
        uint256 amount;
        uint256 floorAmount = rewardToken == stakedToken() ? totalSupply() : 0;
        if (balance > floorAmount) {
            if (_amount > balance.sub(floorAmount)) {
                amount = balance.sub(floorAmount);
            } else {
                amount = _amount;
            }
        }
```

## Impact Details

In both contracts, the caller of the safe transfer function ignores the returned actual transfer amount. It updates the user's rewardDebt as if the full pending reward amount was successfully paid out.

```solidity
safeTokenTransfer(msg.sender, pending.sub(locked));
```

Consequently, the unpaid portion of the user's accrued rewards is permanently lost. There is no tracking or refund mechanism, and the user's reward debt cannot be rolled back. Even if the governor funds the contract with additional reward tokens later, the user cannot reclaim their lost rewards.

## Likelihood

The vulnerability can be triggered whenever the contract's available reward balance becomes lower than the rewards owed to users:

```text
_amount > balance - totalSupply()
```

(or, in V1.1)

```text
_amount > rewardToken.balanceOf(address(this)) - floorAmount
```

## Scenario (included in PoC)

Pool State: pid:9

```text
{
  lpToken (address) : 0x...
  allocPoint (uint256) : 0
  lastRewardBlock (uint256) : 31272476
  accRewardPerShare (uint256) : 911075967026
}
```

1.
User stakes: 1,000,000 TOKEN ($230)
Pending rewards: 1,000,000 × 0.911 = 911,000 TOKEN

2.
User calls deposit(pid, 0) to claim rewards
Pending calculated: 1,000,000 × 0.911 = 911,000 TOKEN
safeTokenTransfer called with 911,000 TOKEN
Cap triggers: 911,000 > 100,000 available
Only 100,000 TOKEN transferred (return value ignored)
rewardDebt updated to 911,000 (assumes full payment)

3.
User received: 100,000 TOKEN
rewardDebt set to: 911,000 TOKEN
Difference: 811,000 TOKEN ($186) permanently lost
Future rewards subtract inflated rewardDebt, compounding the loss
