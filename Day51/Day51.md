**Creating an NFT TokenURI On-Chain**

Back in our NFT and now we know all about abi.encoding stuff and what it does.svgToImageURI function till is great to get an image but we don't want just an image.We're going to need the metadata.We need to to be the JSON object not just an image URL.We need to stick base64 encoded image into the image field of our JSON.We know that ERC721 code comes with a tokenURI and it's that tokenURI points to the JSON object which tells us what our code is going to look like.We can actually base64 encode our JSON as well to turn into a JSON token URI.

```solidity
function tokenURI(uint256 tokenId) public view override returns (string memory) {
        if (!(_exists(tokenId))) {
            revert DynamicSvgNft__NonExistentToken();
        }
    }
```

_exists function comes in ERC721.

Now we want to figure out how to make the tokenURI return a base64 encoded version of JSON.So first we know how to concatenate a string.That's going to be the first thing that we're going to do. 

```solidity
string memory imageURI = "hi!";
        abi.encodePacked(
            '{"name":"',
            name(),
            '", "description": "An NFT that changes based on the Chainlink Feed", ',
            '"attributes":[{"trait_type":"coolness", "value":100}], "image":"',
            imageURI,
            '"}'
        );
```

Doing abi.encodePacked is going to concatenate this all together.

How do we turn this into a base64 encoded tokenURI so that other people can read it?

We're going to typecast this whole thing to bytes and now the whole thing is in bytes we can do exactly what we did with the SVG.Now we can base64 encode it.

```solidity
Base64.encode(
            bytes(
                abi.encodePacked(
                    '{"name":"',
                    name(),
                    '", "description": "An NFT that changes based on the Chainlink Feed", ',
                    '"attributes":[{"trait_type":"coolness", "value":100}], "image":"',
                    imageURI,
                    '"}'
                )
            )
        );
```

This here is going to give us the URL but it's not going to give us `data:image/svg+xml;base64,` this part.So we just need to append the first bit now and we should be good to go.This `data:image/svg+xml;base64,` is the prefix for base64 svg images.The prefix for base64 JSON is going to be `data:application/json;base64,`.So we're going to do it like this instead.

Now the ERC721 has something called baseURI that we're going to override and that we're going to use.

```solidity
function _baseURI() internal pure override returns (string memory) {
        return "data:application/json;base64,";
    }
```

Now we can use the baseURI to append to our base64 encoded JSON.So in order to append them, once again, we're going to do abi.encodePacked.

```solidity
abi.encodePacked(
            _baseURI(),
            Base64.encode(
                bytes(
                    abi.encodePacked(
                        '{"name":"',
                        name(),
                        '", "description": "An NFT that changes based on the Chainlink Feed", ',
                        '"attributes":[{"trait_type":"coolness", "value":100}], "image":"',
                        imageURI,
                        '"}'
                    )
                )
            )
        );
```

We're going to encode baseURI to the encoded JSON.This is obviously a bytes object but we want a string.So all we gotta do is typecast it as a string and return it.

```solidity
return
            string(
                abi.encodePacked(
                    _baseURI(),
                    Base64.encode(
                        bytes(
                            abi.encodePacked(
                                '{"name":"',
                                name(),
                                '", "description": "An NFT that changes based on the Chainlink Feed", ',
                                '"attributes":[{"trait_type":"coolness", "value":100}], "image":"',
                                imageURI,
                                '"}'
                            )
                        )
                    )
                )
            );
```

We're creating a JSON string, we encoded in bytes that way we could encode in base64.Once we've encoded in base64, we then just append the prefix for JSON objects, we do abi.encodePacked and cast it to string.Now we've a tokenURI.All we have to do is update our imageURI with what we get from `svgToImageURI` then we'll be good to go.

**Making the NFT Dynamic**

So in our constructor, we're passing the lowSvg and the highSvg.What are these lowSVG and highSvg? We're basically saying when the price of the asset is too low, show a frown.svg and when the price of the asset is high, show a smily face.So we're going to give it a frown.svg and happy.svg as input parameters.We probably want to save those but we don't necessarily want to save them in their svg format.We just want to store the imageURI instead of the actual svg.

```solidity
constructor(string memory lowSvg, string memory highSvg) ERC721("Dynamic SVG NFT", "DSN") {
        s_tokenCounter = 0;
        i_lowImageURI = svgToImageURI(lowSvg);
        i_highImageURI = svgToImageURI(highSvg);
    }
```

Now svgToImageURI is going to return the imageURI and we're going to store the imageURI on-chain.Now we've the two of those, we can use them in our tokenURI function.When somebody calls tokenURI of tokenId 0, we're going to stick into our JSON either the lowImageURI or the highImageURI and we're actually going to base that off of the Chainlink price feed.So let's go and add chainlink contracts.

`yarn add --dev @chainlink/contracts`

```solidity
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
```

We're going to call pricefeed to figure our what the price is and then show the high image or the low image based off that.So in order to get the priceFeed, let's just add pricefeed address in our constructor and then we'll make another variable  of type AggregatorV3Interface.

```solidity
AggregatorV3Interface internal immutable i_priceFeed;

constructor(
        address priceFeedAddress,
        string memory lowSvg,
        string memory highSvg
    ) ERC721("Dynamic SVG NFT", "DSN") {
        s_tokenCounter = 0;
        i_lowImageURI = svgToImageURI(lowSvg);
        i_highImageURI = svgToImageURI(highSvg);
        i_priceFeed = AggregatorV3Interface(priceFeedAddress);
    }
```

Then we'll get the latest price in our tokenURI function.

```solidity
(, int256 price, , , ) = i_priceFeed.latestRoundData();
```

Based on this price, we'll show the image. 

```solidity
string memory imageURI = i_lowImageURI;
        
if (price >= ??) {
        imageURI = i_highImageURI;
    }
```

All we gotta do is figure our the price.We'll let the minter choose the value that they want to use.For each NFT, their own high value.

```solidity
mapping (uint256 => int256) public s_tokenIdToHighValue;
```

We'll say that when they mint an NFT, we'll set that equal to highValue.So when they mint they choose the highValue that they want.

```solidity
function mintNft(int256 highValue) public {
        s_tokenIdToHighValue[s_tokenCounter] = highValue;
        _safeMint(msg.sender, s_tokenCounter);
        s_tokenCounter++;
    }
```

Then we'll say if the price is greater than or equal to the highValue of the tokenId, then we'll use the high one otherwise we'll just use the low one.

```solidity
if (price >= s_tokenIdToHighValue[tokenId]) {
            imageURI = i_highImageURI;
        }
```

Now the only thing that we want to add here is probably an event.We probably want to emit an event when we mint the NFT.

```solidity
event NFTMinted(uint256 indexed tokenId, int256 highValue);

function mintNft(int256 highValue) public {
        s_tokenIdToHighValue[s_tokenCounter] = highValue;
        s_tokenCounter++;
        _safeMint(msg.sender, s_tokenCounter);
        emit NFTMinted(s_tokenCounter, highValue);
    }
```

Let's just make sure everything compiles here `yarn hardhat compile`

First thing we need to do is write our deploy function.Now we're going to do a dynamic NFT that's hosted 100% on chain and it changes based off the price of as asset.

**Dynamic SVG On-Chain NFT Deploy Script**

Let's create a new file called "03-deploy-dynamic-nft.js" inside deploy folder and grab the boilerplate from the basic NFT.

```javascript
const { network } = require("hardhat")
const { developmentChains } = require("../helper-hardhat-config")
const { verify } = require("../utils/verify")

module.exports = async function ({ getNamedAccounts, deployments }) {
    const { deploy, log } = deployments
    const { deployer } = await getNamedAccounts()
}
```

What do we need for our constructor? well we need the priceFeedAddress, lowSvg and highSvg.So let's get all of those.Price feed address is something that we've already done before and we can add that in our helper-hardhat-config.If we're on local, we're going to use a mock and if we're on Rinkeby or actual network, we're going to use an actual address.So let's head to [chainlink docs](https://docs.chain.link/docs/ethereum-addresses/) and grab the price feed address.

```javascript
4: {
        name: "rinkeby",
        vrfCoordinatorV2: "0x6168499c0cFfCaCD319c818142124B7A15E857ab",
        ethUsdPriceFeed: "0x8A753747A1Fa494EC906cE90E9f37563A8AF630e",
    },
```

Since for local host we need to do a mock.So let's create a "MockV3Aggregator.sol" inside contracts/test.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "@chainlink/contracts/src/v0.6/tests/MockV3Aggregator.sol";
```

This is using 0.6.0 version of solidity so we're going to make sure that in our hardhat.config, we've atleast one 0.6.0 version.

```javascript
solidity: {
        compilers: [{ version: "0.8.8" }, { version: "0.4.19" }, { version: "0.6.12" }],
    },
```

It means in our deploy mocks, we're going to add code to deploy MockV3Aggregator.

```javascript
// outside of model.exports
const DECIMALS = "18"
const INITIAL_PRICE = ethers.utils.parseUnits("2000", "ethers") 

// inside of model.exports
await deploy("MockV3Aggregator", {
            from: deployer,
            log: true,
            args: [DECIMALS, INITIAL_PRICE],
        })
```

So we've waited to deploy mocks for that priceFeed.

```javascript
let ethUsdPriceFeedAddress

if (developmentChains.includes(network.name)) {
    const EthUsdAggregator = await ethers.getContract("MockV3Aggregator")
    ethUsdPriceFeedAddress = EthUsdAggregator.address
} else {
    ethUsdPriceFeedAddress = networkConfig[chainId].ethUsdPriceFeedAddress
}
```

We've the EthUsdPriceFeed, now we need the lowSvg and the highSvg.So we're going to create a new folder in our images folder called "dynamicNft" and put the frown and happy svg images there.Now we've those we want to read them into our scripts here.

```javascript
const lowSvg = await fs.readFileSync("./images/dynamicNft/frown.svg", { encoding: "utf8" })
const highSvg = await fs.readFileSync("./images/dynamicNft/happy.svg", { encoding: "utf8" })
```

When price is good, we're going to do happy.svg but if it's bad, we'll do frown.svg.
Now let's go ahead and let's deploy this contract.

```javascript
args = [ethUsdPriceFeedAddress, lowSvg, highSvg]

const dynamicSvgNft = await deploy("DynamicSvgNft", {
    from: deployer,
    args: args,
    log: true,
    waitConfirmations: network.config.blockConfirmations || 1,
})
```

So let's try to see the deploy script that we just created works.

`yarn hardhat deploy --tags dynamicsvg`
