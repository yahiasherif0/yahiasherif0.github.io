---
title: "Dust Token Transfers Permanently Prevent Asset Removal"
date: 2026-06-15
categories: [Bug Bounty]
tags: [solidity, "smart contracts", "DoS"]
---

## Brief

Transferring dust amounts to vaults leads to Dos of the removeAsset() function

## Vulnerability Details

removeAsset() is a function callable by the contract owner that, by calling it on a specific ERC20 token, leads to removing the token from the storage of the contract and blocking users from using it

```solidity
    function removeAsset(address token) external onlyOwner override {
        (uint256 inVault, uint256 inStrategy) = balances(token);
        require(inVault == 0, "OTV1B: can't remove token with vault balance > 0");
        require(inStrategy == 0, "OTV1B: can't remove asset with strategy balance > 0");
        require(assetSet.exists(token), "OTV1B: unknown token");
        if(collateralTokenSet.exists(token)) collateralTokenSet.remove(token, "OTV1B: ERR removing collateral token");
        if(otherTokenSet.exists(token)) otherTokenSet.remove(token, "OTV1B: ERR removing MEM token");
        assetSet.remove(token, "OTV1B: ERR removing asset");
        delete assets[token];
        emit AssetRemoved(msg.sender, token);
    }
```

However, there's a check that prevents the asset from being removed if its vault balance is not equal to 0

```solidity
        require(inVault == 0, "OTV1B: can't remove token with vault balance > 0");
```

So if an attacker transferred a dust amount 1 WEI to the vault, the function removeAsset() will fail to execute

## Scenario

Owner try to remove specific asset from the contract so he calls removeAsset()
Attacker catches the transaction at the memepool, then he frontruns it and transfers dust amount 1 Wei to the vault
The owner transaction will fail and the asset will still be existed

## Impact Details

Impact Category: Governance DoS / Griefing
Direct Losses: None (no fund theft)
Operational Impact:

- Governance cannot remove assets from the vault registry
- Deprecated or delisted tokens remain permanently registered
- Registry becomes polluted with unwanted assets
- Attack cost: 1 wei of token (negligible)
