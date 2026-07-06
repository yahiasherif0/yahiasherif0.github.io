---
title: "Unsafe transfer() Causes DoS for Multisig and Smart Contract Wallets"
date: 2026-06-10
categories: [Bug Bounty]
tags: [solidity, "smart contracts", "DoS",]
---

# Unsafe `transfer()` Causes DoS for Multisig and Smart Contract Wallets

## Brief

The `syncUserWeight()` function uses the unsafe payable `transfer()` method to refund excess ETH to users.

## Vulnerability Details

The `syncUserWeight()` function refunds excess ETH using payable `transfer()`. Since `transfer()` forwards only 2300 gas, refunds can fail when the recipient is a multisig or smart contract wallet with complex `receive()` or `fallback()` logic, causing the entire transaction to revert and resulting in a denial of service.

## Code Snippet
```solidity
        if ((msg.value - fee) != 0) {
            payable(_msgSender()).transfer(msg.value - fee);
        }
```

## Impact Details

Transactions will revert when:

- The recipient contract does not implement a payable function.
- The recipient's payable `receive()` or `fallback()` function requires more than 2300 gas.
- The recipient is a proxy-based smart contract wallet where the forwarded gas is insufficient.

As a result, multisig and smart contract wallet users may be unable to successfully execute the affected functions.

## Mitigation

```diff
- payable(_msgSender()).transfer(msg.value - fee);

+ (bool success, ) = payable(_msgSender()).call{value: msg.value - fee}("");
+ require(success, "ETH refund failed");

```

