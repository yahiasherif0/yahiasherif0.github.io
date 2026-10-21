---
title: "LPPlugin::deposit Function Does Not Refund Unconsumed Tokens"
date: 2026-04-13
categories: [Bug Bounty]
tags: [solidity, "smart contracts", "DeFi"]
---

# LPPlugin::deposit Function Does Not Refund Unconsumed Tokens

## Summary

The LPPlugin::deposit function does not refund unconsumed tokens, causing permanent loss of user funds when depositing into out-of-range tick positions.

## Vulnerability Details

The deposit() function takes tickLower, tickUpper, amount0, and amount1 as parameters. It transfers both token amounts from the user into the LPPlugin contract, then calls ILPCallback.mint() to add liquidity at the specified tick range.

When the tick range is out of range relative to the current pool price, the Algebra pool only consumes one token — the other stays untouched. ILPCallback.mint() returns amount0Used and amount1Used indicating exactly how much was consumed, but deposit() completely ignores these return values. No refund calculation is performed and no transfer of the surplus is made back to the user.

## Impact

permanent loss of user funds. Any user who calls deposit() with a tick range outside the current pool price will lose the entirety of their unconsumed token amount. The tokens become permanently trapped in the LPPlugin contract with no recovery mechanism

## Recommended Mitigation

Apply the same refund pattern already used in onERC721Received() (lines 318–339) to the deposit() function.

```diff
-       ILPCallback(callback).mint(
+       (uint256 amount0Used, uint256 amount1Used, ) = ILPCallback(callback).mint(


+       uint256 refund0 = actual0 - amount0Used;
+       uint256 refund1 = actual1 - amount1Used;
+       if (refund0 > 0) IERC20(IAlgebraPool(pool).token0()).safeTransfer(msg.sender, refund0);
+       if (refund1 > 0) IERC20(IAlgebraPool(pool).token1()).safeTransfer(msg.sender, refund1);
```

## Validation steps

Step 0: Pool is initialized at tick 0 (price 1:1). User calls deposit() with tick range [-12000, -6000], which is entirely below the current price — meaning the pool will only consume token1 and ignore token0.

Step 1: User calls deposit() and tokens are transferred to LPPlugin:

```solidity
token0.transferFrom(msg.sender, address(this), amount0);
token1.transferFrom(msg.sender, address(this), amount1);
```

Step 2: LPPlugin approves callback for full amounts:

```solidity
token0.approve(callback, actual0);
token1.approve(callback, actual1);
```

Step 3: Callback mints liquidity to pool:

```solidity
ILPCallback(callback).mint(
    recipient,
    tickLower,
    tickUpper,
    actual0,
    actual1
);
```

Step 4: Pool only requests what it needs via callback:

```solidity
function algebraMintCallback(uint256 amount0Owed, uint256 amount1Owed, bytes calldata data) external override {
    CallbackStructs.MintCallbackData memory decoded = abi.decode(data, (CallbackStructs.MintCallbackData));
    require(msg.sender == pool, "Invalid caller of callback");
    IAlgebraPool algebraPool = IAlgebraPool(pool);

    if (amount0Owed > 0) _pay(algebraPool.token0(), decoded.payer, msg.sender, amount0Owed);
    if (amount1Owed > 0) _pay(algebraPool.token1(), decoded.payer, msg.sender, amount1Owed);
}
```

Step 5: No code returns unused tokens to user:

```solidity
    ILPToken(lpTokenByTicks[tickLower][tickUpper]).transfer(
        recipient,
        lpReceived
    );
}
```

```bash
Place DepositPoc.spec.ts in /test/ and run
npx hardhat test test/DepositPoc.spec.ts
```
