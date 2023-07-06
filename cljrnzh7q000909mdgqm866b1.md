---
title: "The Developer's Guide to Chainlink VRF: Foundry Edition"
seoTitle: "The Developer's Guide to Chainlink VRF with  Foundry"
seoDescription: "In this article, you will learn how to build and test smart contracts powered by Chainlink's VRF service, all in Foundry."
datePublished: Thu Jul 06 2023 21:33:03 GMT+0000 (Coordinated Universal Time)
cuid: cljrnzh7q000909mdgqm866b1
slug: the-developers-guide-to-chainlink-vrf-foundry-edition
canonical: https://blog.developerdao.com/the-developers-guide-to-chainlink-vrf-foundry-edition
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1688679055738/082beddc-9cea-4817-9c11-4694fd7d1614.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1688679137374/a2fef86e-ed84-48d4-9474-1e9c3e6b24ef.png
tags: programming-blogs, blockchain, web3

---

Generating truly random numbers on a computer is a complex mathematical problem. However, most programming languages today have either native support for generating random numbers or come with supporting libraries that generate random numbers with an acceptable level of determinism mixed in.

While generating acceptably-random numbers is a mostly solved problem in traditional computing, it is nigh impossible to do so on blockchains.

Yes, you could hash together a bunch of block and transaction data with a `keccak256` function, but that is ad-hoc at best and is definitely not production-worthy code today.

If you've been in Web3 for any reasonable amount of time, you have undoubtedly heard of Chainlink.  
Chainlink is a decentralized oracle network that feeds a variety of data to blockchains in real-time so that smart contract developers can access reliable off-chain data without compromising the security of their contracts.

In this article, you will learn about Chainlink's VRF service, a powerful tool that you can use to integrate randomization into your smart contract securely.  
Plus, we will do everything with Foundry, one of the market's latest smart contract development frameworks.

## What will we build?

In this article, we will:

1. Set up a dev environment with [Foundry](https://book.getfoundry.sh/) to work with [Chainlink](https://github.com/smartcontractkit/chainlink) and [Openzeppelin's](https://www.openzeppelin.com/contracts) contracts.
    
2. Upload three images and their corresponding JSON metadata to [Pinata](https://www.pinata.cloud/), a pinning service for IPFS.
    
3. Set up an ERC1155 contract, without VRF, to mint multiple tokens from our limited collection of images.
    
4. Randomize the mint function by integrating Chainlink VRF into our contract.
    
5. Test our random NFT minting contract, now powered by Chainlink's VRF, using a local mock contract.
    
6. Deploy and verify our contract to the Mumbai testnet using Forge, to make it possible for anyone to mint an NFT from our contract.
    

> Note 📝: A few months ago, Patrick Collins posted a [tweet with pictures](https://twitter.com/PatrickAlphaC/status/1615049532827635713) that looked like they were part of a gym photoshoot.  
> These are the pictures that we will be turning into NFTs on the Mumbai testnet, with permission to do so from Patrick.
> 
> I figured that hardly anything could sell harder than Patrick in a Chainlink article. Also makes for a fun project with a cool end result.
> 
> Follow along, and by the end you will be the owner of a new, shiny Patrick Collins NFT.

## Before we start

This will be a no-holds-barred, technical, deep dive into Chainlink VRF. This article is, in fact, about 70% of what I originally intended to publish.

If you don't feel confident in your Solidity skills, I highly recommend you check out this [full blockchain development course with Foundry](https://www.youtube.com/watch?v=umepbfKp5rI&list=PL4Rj_WH6yLgWe7TxankiqkrkVKXIwOP42) on Youtube, also published by Patrick.

This is hands down the most up-to-date and in-depth course on blockchain development to exist, and you will definitely be able to follow along with the article if you could complete at least part 1 of the series.

[![](https://cdn.hashnode.com/res/hashnode/image/upload/v1688586229113/45d5f7be-c003-4190-b1d0-f10f6d7446bb.png align="center")](https://www.youtube.com/watch?v=umepbfKp5rI&list=PL4Rj_WH6yLgWe7TxankiqkrkVKXIwOP42)

I recommend you don't follow along with the article in the first read, primarily if you haven't used VRF before.  
Instead, try to digest the concepts I have worked hard to put into simple words.

As a reward for reading through to the end, you will be able to mint yourself a Patrick Collins NFT :)

## Setting up a dev environment with Foundry

Foundry is an increasingly popular smart contract development framework.

This is not an introductory course to Foundry. I recommend you check out the [Foundry book](https://book.getfoundry.sh/) for a detailed reference or [this repo I made](https://github.com/chainstacklabs/learnweb3dao-foundry-workshop) for a quick crash course.

Once you have Foundry installed, make sure all of its components are up to date using the following:

```bash
foundryup
```

Open a new terminal in a new directory, and initialize a new Foundry project using:

```bash
forge init
```

We use Chainlink and Openzeppelin's smart contract libraries as part of our code. To install [Openzeppelin contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) into your Foundry project, run:

```bash
forge install Openzeppelin/openzeppelin-contracts
```

We don't need to install the repo containing all of Chainlink's code alongside its node binary. We can install its [slimmed-down version](https://github.com/smartcontractkit/chainlink-brownie-contracts), containing only the contracts, by running:

```bash
forge install smartcontractkit/chainlink-brownie-contracts
```

By default, Forge manages dependencies via git submodules, and we don't need to change this behavior (even though we can). You can find all the dependencies for this project in the `lib` directory.

> Note 📝: [Foundry](https://github.com/foundry-rs/foundry) is modular in design and is a collection of four different CLI tools(for now). These are Forge, Cast, Anvil, and Chisel.  
> In this article, I'll mainly be using Forge and Anvil.

## Setting up IPFS metadata

1. Go to [Pinata](https://www.pinata.cloud/), and sign up for an account. We only need the free tier for our needs.
    
2. Gather all the images you want to tokenize into a single folder. I named Patrick's pictures `1.png`, `2.png`, and `3.png`. I highly recommend following a simplified naming convention.
    
3. Upload all these images to Pinata as a single folder. This means you'll receive a single content identifier (CID). An individual image can now be accessed as `ipfs://CID/1.png`. My folder of images can be accessed via [this link](https://gateway.pinata.cloud/ipfs/QmQCRiKqzirEUBkjpoYJBKCBG4ynpknAjqH4Cp6rLTSTik/).
    
4. Next, we will create three individual JSON files to store Opensea-compatible metadata. Again we'll name them as `1.json`, `2.json`, and `3.json`. You can read about Opensea's metadata standards in detail [on their docs](https://docs.opensea.io/docs/metadata-standards). For now, this is what 1.json will look like. You can check out all three of the JSON files through this [IPFS URL](https://gateway.pinata.cloud/ipfs/QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/).
    
    ```json
     {
      "name": "Patrick in the gym #1",
      "description": "Call the mint function from this contract to get one of the three images from Patrick's gym photoshoot. This contract has a randomized mint function powered by Chainlink's VRF service.",
      "image": "ipfs://QmQCRiKqzirEUBkjpoYJBKCBG4ynpknAjqH4Cp6rLTSTik/1.png",
      "edition": 1,
      "date": 1685971561,
      "attributes": [
        {
          "trait_type": "Probability of getting this image.",
          "value": "1%"
        }
      ]
    }
    ```
    
    The critical thing is to note the probability value. This means we want a minter to have only a `1%` to get *1.png*. These values are `33%` and `66%` for the second and third images and will be enforced via Chainlink VRF.
    
5. Finally, we upload these 3 JSON files to Pinata, again, **as part of a single folder.** This gives us a single CID to access all 3 of these files. Opensea only uses JSON metadata to display the NFT image and associated properties.
    

## A generic ERC1155 contract

> Pro Tip 💡: Before moving further, I highly recommend you know the differences between the 721 and 1155 NFT standards.

Before adding randomization to our smart contract, let us set up a generic ERC1155 smart contract.

1. Go to [Openzeppelin Contracts Wizard](https://docs.openzeppelin.com/contracts/4.x/wizard) and set up a boilerplate ERC1155 contract with the following configurations
    
    ![Openzeppelin Contracts Wizard](https://cdn.hashnode.com/res/hashnode/image/upload/v1686337431526/f2ef8307-5e3c-4f27-90aa-4db13566caee.png align="center")
    

> Note 📝: The IPFS metadata for our collection can be accessed as `ipfs://CID/{1 or or 2 or 3}.json`. These numbers will also be the token IDs of our pictures. Hence, we pass the generic CID of our metadata to the smart contract like this:
> 
> *"*`ipfs://QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/{id}.json`*"*
> 
> Any instance of `{id}` will be replaced by the `tokenID` by clients like Opensea.

1. Inside the `src` directory at the root of your Foundry project, create a file named `nft.sol`. Paste the code as it is inside.
    
2. We make a few changes.
    

* I removed the `mintBatch` function because I don't want anyone to have more than 1 NFT from this contract.
    
* Added a public string called `name` initialized with this value: `Patrick Through VRF`. We need to expose a public string called `name` for Opensea to be able to give a name to our collection. This variable is created automatically in the ERC721 standard but not in the ERC1155 standard.
    
* Next, I created a mapping called `_minted` that will keep track of all the addresses that have minted an NFT already minted an NFT.
    
* After this, I hardcoded all the parameters of the mint function except the `tokenID`.
    
* Lastly, I added a simple event that will be emitted every time our contract mints an NFT.
    

This is what our contract looks like at this point.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

contract PatrickInTheGym is ERC1155, ERC1155Burnable, Ownable, ERC1155Supply {
    
    mapping(address => bool) public _minted;
    string public name = "Patrick Through VRF";

    event TokenMinted(address indexed account, uint256 indexed id);

    constructor() ERC1155("ipfs://QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/{id}.json") {}

    function mint(uint256 id)
        public
    {
        require(!_minted[msg.sender], "You can only mint once");
        _minted[msg.sender] = true;
        _mint(msg.sender, id, 1, "");
        emit TokenMinted(msg.sender, id);
    }

    // The following functions are overrides required by Solidity.
    function _beforeTokenTransfer(address operator, address from, address to, uint256[] memory ids, uint256[] memory amounts, bytes memory data)
        internal
        override(ERC1155, ERC1155Supply)
    {
        super._beforeTokenTransfer(operator, from, to, ids, amounts, data);
    }
}
```

This contract will allow anyone to call the mint function from our contract **exactly once**, and that person can choose the image they want by passing in the `tokenID` of their choice.

Keep this in mind. I'll expand on this below.

## Remappings in Foundry

Let us compile our contract to make sure everything works smoothly up to this point. To compile contracts in Foundry, run:

```bash
forge build
```

But Forge won't be able to compile our contract right away since it doesn't understand the format our import statements are using. More precisely, Forge has no idea what `"@openzeppelin"` is.

Run the following command in your terminal:

```bash
forge remappings > remappings.txt
```

This command will create a new file named `remappings.txt` at the root of your project and will fill it with some remappings that Forge has automatically deduced for you. For now, make sure you add this line to the remappings file.

```bash
@openzeppelin/=lib/openzeppelin-contracts/
```

Save the changes in the remappings file, and rerun the build command. This time our contract should compile successfully.

## What do we want Chainlink VRF for?

Our contract allows anyone to mint one NFT from our collection by passing a `tokenID` of their choice. We want to integrate Chainlink VRF into our contract so that the mint function randomly mints one of the three pictures with varying probability levels.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686342227880/80fbcf22-8f10-4f44-9b38-62ebcd89a645.png align="center")

Here's the solution I came up with.

1. Ask VRF to generate a random number between 1 and 100, including both bounds.
    
2. If the returned number is 100, mint an NFT with the `tokenID` set to 1.
    
3. If the number returned is divisible by 3, mint an NFT with the `tokenID` set to 2.
    
4. Else, mint an NFT with the `tokenID` set to 3.
    

> Note 📝: If you can develop a more efficient solution, comment below.

## Creating a VRF subscription

Chainlink VRF currently offers us two methods for requesting randomness:

1. **Direct Funding**: This method entails maintaining an appropriate balance of LINK tokens in the consuming contract to pay for each randomness request.
    
2. **Subscription**: This method creates a particular 'subscription' containing the required LINK tokens. This account can then be used to fund multiple consuming contracts as per the owner's wishes.
    

We will go with the Subscription method in this tutorial.

1. Go to [faucets.chain.link](http://faucets.chain.link) and request a few LINK tokens to an EOA.
    
2. Go to [vrf.chain.link](http://vrf.chain.link) and create a new subscription on the Mumbai testnet.  
    The subscription id is what we will need soon.
    
3. Once your subscription has been created, add some LINK tokens.
    
4. We will add a consuming contract to our subscription once we have deployed one.
    

## VRF-powered randomization

Create a new file named `nftVRF.sol` inside the `src` directory.

Get ready. The real stuff starts now.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686342744182/54b8d45f-e86d-47c5-b2d6-31762fe4b639.gif align="left")

First, we need to import some Chainlink dependencies into our contract. Add these imports to `nftVRF.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

//Chainlink VRF imports
import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";
```

You might need to configure Chainlink imports the same way as before in the remappings file.

> Pro-tip 💡: Forge installed an outdated version of the Chainlink contracts repo for me. I don't understand why that happened.  
> If you face the same issue, run `forge update lib/chainlink-brownie-contracts` to update this library.

1. The `VRFCoordinatorV2Interface` is an interface used to interact with the VRFCoordinator contract deployed on the chain you are using. You can check out the coordinator contract for Mumbai testnet [here](https://mumbai.polygonscan.com/address/0x7a1BaC17Ccc5b313516C5E16fb24f7659aA5ebed#readContract).  
    An interface in Solidity is a collection of function declarations (**NOT** definitions) marked external.  
    Interfaces are useful when your smart contract needs to interact with another smart contract, and you only need to know the function signatures of the other contract.  
    To send a randomness request to Chainlink, we call the `requestRandomWords()` on this contract.  
    The subscription you created on [vrf.chain.link](http://vrf.chain.link) is just a UI that calls the `createSubscription()` function on the coordinator contract.  
    Please check out the interface code for a better understanding.
    
2. The `VRFConsumerBaseV2` is an abstract contract. The Chainlink Coordinator requires us to inherit this contract as a parent and implement a function named `fulfillRandomWords()`.  
    The Coordinator then calls the `fulfillRandomWords()` once the random values are generated.
    

> Note 📝: An abstract contract is like a regular contract, but it's not fully implemented. It may have some functions without a body (i.e., without implementation). A contract with at least one function without implementation is considered abstract.

Declare the main contract while importing all the dependencies:

```solidity
contract PatrickInTheGym is ERC1155,
                            ERC1155Burnable, 
                            Ownable, 
                            ERC1155Supply, 
                            VRFConsumerBaseV2
{
}
```

You will immediately see the whole thing go red in errors. This is because the `VRFConsumerBaseV2` contract requires a constructor to be initialized, and our contract won't compile till we provide constructor arguments for all base contracts.

Let us start configuring all the variables we need to call the `requestRandomWords()` function from the Coordinator contract. Take a look at all these variables:

```solidity
//Chainlink Variables
VRFCoordinatorV2Interface private immutable CoordinatorInterface;
uint64 private immutable _subscriptionId;
address private immutable _vrfCoordinatorV2Address;
bytes32 keyHash = 0x4b09e658ed251bcafeebbc69400383d49f344ace09b9576fe248bb02c003fe9f;
uint32 callbackGasLimit = 200000;
uint16 blockConfirmations = 10;
uint32 numWords = 1;
```

* The `CoordinatorInterface` is simply a new instance of the `VRFCoordinatorV2Interface` . This instance will be initialized in the constructor.
    
* **Subscription ID**: The unique ID for your subscription that holds the LINK to fund your contract's Randomness request.  
    This value must be initialized in the constructor.
    
* **Coordinator V2 Address**: The address of the Chainlink VRF Coordinator contract on that particular chain.
    
* **Key Hash/Gas Lane**: This hash value represents the maximum gas price you are willing to pay. Mainnets supported by Chainlink VRF typically have multiple supported '*gas lanes*'; the Mumbai testnet, however, has only one.  
    You can check out the value on [Chainlink's documentation](https://docs.chain.link/vrf/v2/subscription/supported-networks/#configurations).
    
* **Callback Gas Limit**: This value specifies the maximum amount of gas the Coordinator contract must use to call the `fulfillRandomWords()` to return the random values.
    
* **Block Confirmations**: This value sets the number of blocks the Coordinator will wait for before sending back our random values. The greater this value is, the more secure the generated random number is. The minimum and maximum block confirmations are specified for each network in Chainlink's documentation.
    
* **Number of words**: The number of random values to get back in one request. We will call back for one word per request.
    

Add those values right below the contract declaration.

### A conceptual detour

Let us take a detour to explore the new workflow more carefully. The mint function will undergo significant changes compared to the generic 1155 contract, and it is vital to understand the differences.

Here's what will happen in the new contract:

1. The user calls a function named `mint()` on the main contract, but this won't directly mint them an NFT. Instead, this mint function will internally call the `requestRandomWords()` that tells the VRF Coordinator:  
    "*Hey dude, I want a random number. Please wait for 10 blocks and then give me a random number*".
    
2. Invoking this function triggers an event called `RandomWordsRequested` from the Coordinator contract; an off-chain VRF node picks that up.
    
3. The VRF node will wait ten blocks (as we specified) before returning a random number to the Coordinator contract.
    
4. The Coordinator will then call the `fulfillRandomWords()` function from our contract and execute whatever logic is included.  
    We will mint NFTs from inside this function.
    

> Note 📝: Let me repeat this. The user will only call the `mint()` function, which triggers the `requestRandomWords()` function.  
> The coordinator contract calls the `fulfillRandomWords()` function, which makes it the 'Callback Function'.

Take a look at this rough diagram. This will become clearer as we write the rest of the code.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686669551812/5f132159-3215-4367-901a-d361607478fe.png align="center")

### Wrapping up the contract

Let us finally get our constructor set up. We need to do two things:

1. Pass our subscription id as a constructor parameter to the main contract.
    
2. Initialize the `VRFConsumerBaseV2` contract's constructor by passing the Coordinator's address.
    

This is how our constructor will look like:

```solidity
constructor(uint64 subscriptionId, address vrfCoordinatorV2Address)
ERC1155("ipfs://QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/{id}.json")
VRFConsumerBaseV2(vrfCoordinatorV2Address)
{
     _subscriptionId = subscriptionId;
     _vrfCoordinatorV2Address= vrfCoordinatorV2Address;
     CoordinatorInterface = VRFCoordinatorV2Interface(vrfCoordinatorV2Address);
}
```

> Note 📝: I also created a new instance of the Coordinator contract called the `CoordinatorInterface` . This will make it simpler to call functions using the interface.

Next, I will declare some state variables required for the contract. This is what our contract looks like right now.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

//Chainlink VRF imports
import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";

contract PatrickInTheGym is
    ERC1155,
    ERC1155Burnable,
    Ownable,
    ERC1155Supply,
    VRFConsumerBaseV2
{
    //Contract Variables and events
    mapping(address => bool) public _minted;
    string public name = "Patrick Through VRF";
    mapping(uint256 => address) public _requestIdToMinter;
    event RequestInitalized(uint256 indexed requestId, address indexed minter);
    event NftMinted(uint256 indexed tokenID, address indexed minter);

    //Chainlink Variables
    VRFCoordinatorV2Interface private immutable CoordinatorInterface;
    uint64 private immutable _subscriptionId;
    address private immutable _vrfCoordinatorV2Address;
    bytes32 keyHash = 0x4b09e658ed251bcafeebbc69400383d49f344ace09b9576fe248bb02c003fe9f;
    uint32 callbackGasLimit = 200000;
    uint16 blockConfirmations = 10;
    uint32 numWords = 1;

    constructor(
        uint64 subscriptionId,
        address vrfCoordinatorV2Address
    )
        ERC1155("ipfs://QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/{id}.json")
        VRFConsumerBaseV2(vrfCoordinatorV2Address)
    {
        _subscriptionId = subscriptionId;
        _vrfCoordinatorV2Address= vrfCoordinatorV2Address;
        CoordinatorInterface = VRFCoordinatorV2Interface(vrfCoordinatorV2Address);
    }
}
```

Create a new function `mint()` right below the constructor as follows:

```solidity
 function mint() public returns (uint256 requestId)
 {
        require(!_minted[msg.sender], "You can only mint once");

        //Calling requestRandomWords from the coordinator contract
        requestId = CoordinatorInterface.requestRandomWords(
            keyHash,
            _subscriptionId,
            blockConfirmations,
            callbackGasLimit,
            numWords
        );

        // map the caller to their respective requestIDs.
        _requestIdToMinter[requestId] = msg.sender;

        // emit an event
        emit RequestInitalized(requestId, msg.sender);
    }
```

* The function can only be called by an address that doesn't already hold an NFT from our contract.
    
* We call the `requestRandomWords()` function from the Coordinator contract here. The function returns a unique variable of type `uint256` that we will store as the `requestID`.
    
* The calling of the `requestRandomWords()` function will automatically start the random number generation off-chain process.
    

> Note 📝: Why do we use the `_requestIdToMinter` mapping?
> 
> Because many people worldwide may simultaneously call the mint function. In that case, it is helpful to assign `requestIDs` to the minters since we can keep track of the arriving results.

Create a `fulfillRandomWords()` function right below the `mint()` function like this:

```solidity
function fulfillRandomWords(
        uint256 requestId,
        uint256[] memory randomWords
    ) internal override {
        // get the minter address
        address minter = _requestIdToMinter[requestId];

        // To generate a random number between 1 and 100 inclusive
        uint256 randomNumber = (randomWords[0] % 100) + 1;

        uint256 tokenId;

        //manipulate the random number to get the tokenId with a variable probability
        if(randomNumber == 100){
            tokenId = 1;
        } else if(randomNumber % 3 == 0) {
            tokenId = 2;
        } else {
            tokenId = 3;
        }
        
        // Updating the mapping
        _minted[minter] = true;

        // Finally mint the token
        _mint(minter, tokenId, 1, "");

        // emit an event
        emit NftMinted(tokenId, minter);
    }
```

The piece of code above will be called by the Coordinator contract whenever it wants to return the results of a successful randomness request.

This function, whenever triggered, will mint a random NFT to someone who called the `mint()` function from our contract.

Lastly, add in the `_beforeTokenTransfer` function from the ERC1155 standard.

```solidity

  // The following functions are overrides required by Solidity.

    function _beforeTokenTransfer(
        address operator,
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) internal override(ERC1155, ERC1155Supply) {
        super._beforeTokenTransfer(operator, from, to, ids, amounts, data);
    }
```

This is what the contract finally looks like:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC1155/extensions/ERC1155Supply.sol";

//Chainlink VRF imports
import "@chainlink/contracts/src/v0.8/interfaces/VRFCoordinatorV2Interface.sol";
import "@chainlink/contracts/src/v0.8/VRFConsumerBaseV2.sol";

contract PatrickInTheGym is
    ERC1155,
    ERC1155Burnable,
    Ownable,
    ERC1155Supply,
    VRFConsumerBaseV2
{
    //Contract Variables and events
    mapping(address => bool) public _minted;
    string public name = "Patrick Through VRF";
    mapping(uint256 => address) public _requestIdToMinter;
    event RequestInitalized(uint256 indexed requestId, address indexed minter);
    event NftMinted(uint256 indexed tokenID, address indexed minter);

    //Chainlink Variables
    VRFCoordinatorV2Interface private immutable CoordinatorInterface;
    uint64 private immutable _subscriptionId;
    address private immutable _vrfCoordinatorV2Address;
    bytes32 keyHash = 0x4b09e658ed251bcafeebbc69400383d49f344ace09b9576fe248bb02c003fe9f;
    uint32 callbackGasLimit = 200000;
    uint16 blockConfirmations = 10;
    uint32 numWords = 1;

    constructor(
        uint64 subscriptionId,
        address vrfCoordinatorV2Address
    )
        ERC1155("ipfs://QmXN7twhiJF7pSttkvqxfok9o5p1QWJeCbwRTZvZ5RCzvz/{id}.json")
        VRFConsumerBaseV2(vrfCoordinatorV2Address)
    {
        _subscriptionId = subscriptionId;
        _vrfCoordinatorV2Address= vrfCoordinatorV2Address;
        CoordinatorInterface = VRFCoordinatorV2Interface(vrfCoordinatorV2Address);
    }

    function mint() public returns (uint256 requestId) {
        require(!_minted[msg.sender], "You can only mint once");

        //Calling requestRandomWords from the coordinator contract
        requestId = CoordinatorInterface.requestRandomWords(
            keyHash,
            _subscriptionId,
            blockConfirmations,
            callbackGasLimit,
            numWords
        );

        // map the caller to their respective requestIDs.
        _requestIdToMinter[requestId] = msg.sender;

        // emit an event
        emit RequestInitalized(requestId, msg.sender);
    }

    function fulfillRandomWords(
        uint256 requestId,
        uint256[] memory randomWords
    ) internal override {
        // get the minter address
        address minter = _requestIdToMinter[requestId];

        // To generate a random number between 1 and 100 inclusive
        uint256 randomNumber = (randomWords[0] % 100) + 1;

        uint256 tokenId;

        //manipulate the random number to get the tokenId with a variable probability
        if(randomNumber == 100){
            tokenId = 1;
        } else if(randomNumber % 3 == 0) {
            tokenId = 2;
        } else {
            tokenId = 3;
        }
        
        // Updating the mapping
        _minted[minter] = true;

        // Finally mint the token
        _mint(minter, tokenId, 1, "");

        // emit an event
        emit NftMinted(tokenId, minter);
    }

    // The following functions are overrides required by Solidity.

    function _beforeTokenTransfer(
        address operator,
        address from,
        address to,
        uint256[] memory ids,
        uint256[] memory amounts,
        bytes memory data
    ) internal override(ERC1155, ERC1155Supply) {
        super._beforeTokenTransfer(operator, from, to, ids, amounts, data);
    }
}
```

Run `forge build` to check for any immediate errors. The contract should compile successfully.

> Pro Tip 💡: There might be scenarios where you may want to change values like `keyHash`, `numWords`, or `blockConfirmations`. It is a good idea to expose these variables through a public function guarded by an `onlyOwner` modifier so that you can configure these values if needed.

Let us move on to testing the contract with Forge's testing utilities.

## Testing locally using the mock contract

Chainlink provides us with a `VRFCoordinatorV2Mock` contract for testing purposes. It simulates the behavior of the actual `VRFCoordinatorV2` contract, which allows us to test VRF-powered contracts locally.

Create a file named `vrfTest1.t.sol` inside the' test' directory.

Set up the imports required for testing:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/nftVRF.sol";
import "../lib/chainlink-brownie-contracts/contracts/src/v0.8/mocks/VRFCoordinatorV2Mock.sol";
```

Initialize the test contract like this:

```solidity
contract PatrickInTheGymTest is Test {

}
```

Declare some state variables for the test contract:

```solidity
    //Creating instances of the main contract
    //and the mock contract
    PatrickInTheGym public patrickInTheGym;
    VRFCoordinatorV2Mock public mock;

    //To keep track of the number of NFTs
    //of each tokenID
    mapping(uint256 => uint256) supplytracker;
    
    //This is a shorthand used to represent the full address
    // address(1) == 0x0000000000000000000000000000000000000001
    address alpha = address(1);
```

Some concepts about testing in Foundry:

* If the name of any solidity function in your directory starts with the string "`test`", Forge will treat it as a test function. Therefore, `testVRF()`, is a valid name for a test function, but `VRFtest()` is not.
    
* Forge runs all test functions in a new instance of the EVM by default. This means the state changes due to one test function have no bearing on the results of the next one.
    
* `setup()` is a special function that can be included in your Foundry testing suite. This function is executed by Forge every time before running a new test function.
    
* We will use the `setup()` function to 'set up' the blockchain state we require to test our randomized minting function.
    

Define a new function named `setup()` below the state variables like this:

```solidity
    function setUp() public {
        //Can ignore this. Just sets some base values
        // In real-world scenarios, you won't be deciding the 
        //constructor values of the coordinator contract anyways
        mock = new VRFCoordinatorV2Mock(100000000000000000, 1000000000);

        //Creating a new subscription through account 0x1
        //Prank cheatcode explained below the code snippet
        vm.prank(alpha);
        uint64 subId = mock.createSubscription();

        //funding the subscription with 1000 LINK
        mock.fundSubscription(subId, 1000000000000000000000);

        //Creating a new instance of the main consumer contract
        patrickInTheGym = new PatrickInTheGym(subId, address(mock));

        //Adding the consumer contract to the subscription
        //Only owner of subscription can add consumers
        vm.prank(alpha);
        mock.addConsumer(subId, address(patrickInTheGym));
    }
```

> Note 📝: The Prank cheat code is a convenient way to 'impersonate' a call to the blockchain from a specific address.
> 
> The call right below the Prank cheatcode will be executed with the specified address being set as `msg.sender`.

Now finally, create a function named `testRandomness()` as follows:

```solidity
function testRandomness() public {

        for (uint i = 1; i <= 1000; i++) {

        //Creating a random address using the 
        //variable {i}
        //Useful to call the mint function from a 100
        //different addresses
        address addr = address(bytes20(uint160(i)));
        vm.prank(addr);
        uint requestID = patrickInTheGym.mint();

        //Have to impersonate the VRFCoordinatorV2Mock contract
        //since only the VRFCoordinatorV2Mock contract 
        //can call the fulfillRandomWords function
        vm.prank(address(mock));
        mock.fulfillRandomWords(requestID,address(patrickInTheGym));
        }

        //Calling the total supply function on all tokenIDs
        //to get a final tally, before logging the values.
        supplytracker[1] = patrickInTheGym.totalSupply(1);
        supplytracker[2] = patrickInTheGym.totalSupply(2);
        supplytracker[3] = patrickInTheGym.totalSupply(3);

        console2.log("Supply with tokenID 1 is " , supplytracker[1]);
        console2.log("Supply with tokenID 2 is " , supplytracker[2]);
        console2.log("Supply with tokenID 3 is " , supplytracker[3]);
    }
```

You can run the test file using this command:

```bash
forge test --match-path test/vrfTest1.t.sol -vvvvv
```

> Pro Tip 💡: You can adjust the verbosity levels in your terminal by configuring the '*\-v*' flag. You can read more about this on [Foundry book](https://book.getfoundry.sh/forge/tests?highlight=verbosity#logs-and-traces).

This is what I get back in the terminal if I run the for loop 1000 times:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686752353074/d0efff59-c29f-4662-9546-56356d472e61.png align="center")

* As you can see, the percentages align with what we want.
    
* Please note that a 'test' function passes only if it clears all the required conditions. We did not set any conditions for our test function to fail.
    
* This testing phase was a crude way of checking whether our VRF randomness works.
    

> Sad Note 😔: I wanted to include a full-blown section on Invariant testing.  
> The idea was to leverage [foundry-chainlink-toolkit](https://github.com/smartcontractkit/foundry-chainlink-toolkit) to write a much more wholesome testing suite.
> 
> This project is designed to be used with Forge to spin up a local Chainlink node quickly. However, I couldn't set up the node despite my best efforts.  
> I will write part 2 of this tutorial as soon as possible.

## Deploying to Mumbai

Since I couldn't give you a cool tutorial on invariant testing with a local Chainlink node, let us move on to deploying and verifying our contract.

At the root of your Foundry project, create a `.env` file. Fill the env file with these values:

```bash
RPC_URL=
PRIVATE_KEY=
POLYGONSCAN_API_KEY=
```

* You can get an RPC URL from services like [Alchemy](https://www.alchemy.com/), [Chainstack](https://chainstack.com/), or [Quicknode](https://quicknode.com/). You can also use a public RPC URL if you so wish.
    
* Use a private key with some MATIC tokens on the Mumbai testnet.
    
* Get a [Polygonscan API key](https://polygonscan.com/). A key from the mainnet explorer will work on Mumbai as well.
    

With all the values filled, save the `.env` file. Run this command in the terminal to source these env variables to the terminal:

```bash
source .env
```

We will create a deployment script to deploy our contract to the blockchain.

Create a file named `nftVRF.s.sol` inside the `script` folder. Set up the imports like this:

```solidity
pragma solidity ^0.8.4;

import "forge-std/Script.sol";
import "../src/nftVRF.sol";
```

Create a new contract that inherits `Scripts.sol` that Forge provides us:

```solidity
contract PatrickInTheGymDeploy is Script {
}
```

Now fill the contract like this:

```solidity
    function run() public {

        //Forge can read private key directly from the env file
        uint PrivateKey = vm.envUint("PRIVATE_KEY");
        
        //This cheatcode will broadcast all included transactions
        //on chain
        vm.startBroadcast(PrivateKey);
        
        //Your subscription id will be different
        //but the Coordinator address will remain the same
        //Unless you're not deploying on Mumbai
        PatrickInTheGym patrickInTheGym = new PatrickInTheGym(5125, 0x7a1BaC17Ccc5b313516C5E16fb24f7659aA5ebed);

        vm.stopBroadcast();
    }
```

* Scripts are executed inside the `run()` function by default.
    
* We can deploy our contract to the blockchain by creating a new instance within the `Broadcast()` cheat codes.
    
* Save the file.
    

To execute the script, run this command in the terminal:

```bash
forge script script/nftVRF.s.sol:PatrickInTheGymDeploy \
--rpc-url $RPC_URL \
--broadcast -vvvv
```

Forge will return a contract address in the terminal. Open the address in [Mumbai Explorer](https://mumbai.polygonscan.com/).

To verify this contract, run this command:

> Note 📝: I configured my compiler version to 0.8.17 in the toml file. The rest of the values are default.

```bash
forge verify-contract <YOUR_SMART_CONTRACT_ADDRESS> \
--chain-id 80001 \
--num-of-optimizations 200 \
--watch --compiler-version v0.8.17+commit.8df45f5f \
--constructor-args $(cast abi-encode "constructor(uint64, address)" 5125 0x7a1BaC17Ccc5b313516C5E16fb24f7659aA5ebed)
src/nftVRF.sol:PatrickInTheGym \
--etherscan-api-key $POLYGONSCAN_API_KEY
```

## Getting your own random Patrick

The contract is deployed on the Mumbai testnet and is [verified](https://mumbai.polygonscan.com/address/0xa88d8c1854152882a4f487423d75a361d6a07e5c#writeContract).

Just go to the link to get your random Patrick and call the mint function. As you'll notice, you have no power of choosing the `tokenID`, which was the whole point.  
You can check out the whole collection on [Opensea](https://testnets.opensea.io/collection/patrick-through-vrf-1).  
As of now, nobody has been able to mint the `tokenID` 1.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686768657980/8bca6610-5b1f-4bdd-a553-926356f7d9ed.png align="center")

> A confession on Verification: I have verified contracts on-chain using Forge many times before, but no matter how much I tried, I couldn't verify the contract this time.  
> I have NO IDEA what I am doing wrong.  
> I deployed the exact same contract to Mumbai through REMIX and verified it from there.  
> Please feel free to enlighten me on what I am doing wrong if you can figure it out.