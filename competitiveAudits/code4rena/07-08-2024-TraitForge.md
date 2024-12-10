# TraitForge - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Cannot move to the next generation and mint new Nfts due to access control issue in `EntropyGenerator::initializeAlphaIndices()`](#H-01)
    - ### [H-02. Lack of check for `maxGeneration` while incrementing generation allows minting of NFTs beyond the maximum cap](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Lack of functions to pause or unpause in all contracts implementing the Openzeppelin Pausable plugin](#M-01)
    - ### [M-02. Incorrect higher mint price for the first token minted in each generation starting from the second generation](#M-02)
    - ### [M-03. Different sets of parent tokens can evaluate to the same average entropy value which takes away the uniqueness of tokens](#M-03)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: TraitForge

### Dates: Jul 30, 2024 - Aug 7, 2024

[See more contest details here](https://code4rena.com/audits/2024-07-traitforge)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 3
- Low/QA: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Cannot move to the next generation and mint new Nfts due to access control issue in `EntropyGenerator::initializeAlphaIndices()`

## Summary

While minting the 10,001 th Nft (the first Nft in the second generation) using `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` functions, the `TraitForgeNft::_incrementGeneration()` function calls the `EntropyGenerator::initializeAlphaIndices()` function. However, the `EntropyGenerator::initializeAlphaIndices()` function has an `onlyOwner` modifier. Since the `TraitForgeNft` contract isn't the owner of `EntropyGenerator` contract, this will always lead to a revert.

## Vulnerability Details

Consider the code segment from `TraitForgeNft::_incrementGeneration()` function,

```js
    function _incrementGeneration() private {
        require(generationMintCounts[currentGeneration] >= maxTokensPerGen, "Generation limit not yet reached");
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;
        priceIncrement = priceIncrement + priceIncrementByGen;
@>      entropyGenerator.initializeAlphaIndices();
        emit GenerationIncremented(currentGeneration);
    }
```

The `TraitForgeContract::_incrementGeneration()` function tries to re-initialize the alpha indices (with the God entropy value of 999999). Now let's see the `EntropyGenerator::initializeAlphaIndices()` function,

```js
@>  function initializeAlphaIndices() public whenNotPaused onlyOwner {
        uint256 hashValue = uint256(keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp)));

        uint256 slotIndexSelection = (hashValue % 258) + 512;
        uint256 numberIndexSelection = hashValue % 13;

        slotIndexSelectionPoint = slotIndexSelection;
        numberIndexSelectionPoint = numberIndexSelection;
    }
```

**Links**:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L353

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206

The function has an `onlyOwner` modifier. The owner of the contract is the deployer, so it will always revert when called by `TraitForgeContract::_incrementGeneration()` function.

This issue is applicable to `TraitForgeNft::mintWithBudget()` function as well. However, it currently has a bug which doesn't allow minting beyond the first generation.

```js
    function mintWithBudget(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
        uint256 mintPrice = calculateMintPrice();
        
        // more code here

@     while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
            _mintInternal(msg.sender, mintPrice);
            amountMinted++;
            budgetLeft -= mintPrice;
            mintPrice = calculateMintPrice();
        }
        
        // more code here
    }
```

**Link**:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L215

If the number of token ids minted goes beyond `TraitForgeNft.maxTokensPerGen`, the function will not mint further, however, this is a finding on its own. If this was to be resolved, the function would also be susceptible to the same problem of not being able to move to the next generations.

## Impact

No more tokens can be minted starting from generation 2. The total number of Nfts minted will be capped to 10k in the first generation. This breaks the protocol's core functionality -- being able to mint new Nfts and transitioning to the next generations.

## Proof of Concept

To write the PoC, the following change had to be made to the `TraitForgeNft` contract to stay within the block gas limit.

Changed the `TraitForgeNft.maxTokensPerGen` variable from `1e4` to `1e3`, to avoid `OutOfGas` error.

We will be using the following setup to write PoCs in Foundry,

```js
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {console} from "lib/forge-std/src/console.sol";
import {Test} from "lib/forge-std/src/Test.sol";

import {Trait} from "../contracts/Trait/Trait.sol";
import {TraitForgeNft} from "../contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "../contracts/EntropyGenerator/EntropyGenerator.sol";
import {EntityTrading} from "../contracts/EntityTrading/EntityTrading.sol";
import {EntityForging} from "../contracts/EntityForging/EntityForging.sol";
import {DevFund} from "../contracts/DevFund/DevFund.sol";
import {Airdrop} from "../contracts/Airdrop/Airdrop.sol";
import {DAOFund} from "../contracts/DAOFund/DAOFund.sol";
import {NukeFund} from "../contracts/NukeFund/NukeFund.sol";

contract POCs is Test {
    address public owner;
    address public user1;

    address public dev1;
    address public dev2;
    address public dev3;

    Trait public traitToken;
    TraitForgeNft public traitForgeNft;
    EntropyGenerator public entropyGenerator;
    EntityTrading public entityTrading;
    EntityForging public entityForging;
    DevFund public devFund;
    Airdrop public airdrop;
    DAOFund public daoFund;
    NukeFund public nukeFund;

    function setUp() public {
        owner = makeAddr("owner");
        user1 = makeAddr("user1");
        dev1 = makeAddr("dev1");
        dev2 = makeAddr("dev2");
        dev3 = makeAddr("dev3");

        string memory name = "Trait";
        string memory symbol = "TRT";
        uint8 decimals = 18;
        uint256 tokenTotalSupply = 10_000e18;
        address uniswapV3Router = makeAddr("uniswap v3 router");

        vm.startPrank(owner);
        traitToken = new Trait(name, symbol, decimals, tokenTotalSupply);
        traitForgeNft = new TraitForgeNft();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityTrading = new EntityTrading(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));
        devFund = new DevFund();
        airdrop = new Airdrop();
        daoFund = new DAOFund(address(traitToken), uniswapV3Router);
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(devFund), payable(daoFund));

        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setNukeFundContract(payable(nukeFund));
        traitForgeNft.setAirdropContract(address(airdrop));
        entropyGenerator.setAllowedCaller(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(nukeFund));
        entityForging.setNukeFundAddress(payable(nukeFund));
        airdrop.setTraitToken(address(traitToken));
        airdrop.transferOwnership(address(traitForgeNft));

        uint256 weights = 100;
        devFund.addDev(dev1, weights);
        devFund.addDev(dev2, weights);
        devFund.addDev(dev3, weights);

        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();
        vm.stopPrank();
    }

    // PoCs go here
}
```

Now run the following test case,

```js
    function testMovingToNextGenerationAndMintingNftsRevertsDueToAccessControlIssueInEntropyGenerator() public {
        // Provide some ether to user1
        uint256 maxEthToDeal = 1000 ether;
        vm.deal(user1, maxEthToDeal);

        // Changed this value to 1_000 Nfts in the TraitForgeNft contract to stay within the block gas limit
        uint256 nftsToMint = traitForgeNft.maxTokensPerGen();
        // To avoid the whitelist, we will warp by 1 day
        uint256 warpBy = 1 days + 1;
        vm.warp(block.timestamp + warpBy);
        bytes32[] memory proof = new bytes32[](0);
        uint256 count = 0;

        vm.startPrank(user1);
        // Mint 1k Nfts
        // This allows us to move from the first generation to the second generation
        while (count < nftsToMint) {
            // `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` both use the
            // `TraitForgeNft::calculateMintPrice()` function to calculate the price for the next mint
            uint256 ethToSupply = traitForgeNft.calculateMintPrice();
            traitForgeNft.mintToken{value: ethToSupply}(proof);
            ++count;
        }
        vm.stopPrank();

        uint256 priceForNextMint = traitForgeNft.calculateMintPrice();

        vm.startPrank(user1);
        vm.expectRevert("Ownable: caller is not the owner");
        traitForgeNft.mintToken{value: priceForNextMint}(proof);
        vm.stopPrank();
    }
```

The test passes with the following logs,

```shell
Ran 1 test for test/POCs.sol:POCs
[PASS] testMovingToNextGenerationAndMintingNftsRevertsDueToAccessControlIssueInEntropyGenerator() (gas: 280671900)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 119.37ms (108.89ms CPU time)

Ran 1 test suite in 120.32ms (119.37ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review. Foundry for writing the PoC.

## Recommended Mitigation

Allow both the owner and the `TraitForgeNft` contract to initialize the alpha indices. Allowing owner is necessary because the constructor initializes the alpha indices during deployment.

Make the following changes to `EntropyGenerator`,

```diff
    modifier onlyAllowedCaller() {
-       require(msg.sender == allowedCaller, "Caller is not allowed");
+       require(msg.sender == allowedCaller || msg.sender == owner(), "Caller is not allowed");
        _;
    }

    // more code here

-   function initializeAlphaIndices() public whenNotPaused onlyOwner {
+   function initializeAlphaIndices() public whenNotPaused onlyAllowedCaller {
        uint256 hashValue = uint256(keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp)));

        uint256 slotIndexSelection = (hashValue % 258) + 512;
        uint256 numberIndexSelection = hashValue % 13;

        slotIndexSelectionPoint = slotIndexSelection;
        numberIndexSelectionPoint = numberIndexSelection;
    }
```

Now let's unit test the changes,

```js
    function testMitigation() public {
        // Provide some ether to user1
        uint256 maxEthToDeal = 1000 ether;
        vm.deal(user1, maxEthToDeal);

        // Changed this value to 1_000 Nfts in the TraitForgeNft contract to stay within the block gas limit
        uint256 nftsToMint = traitForgeNft.maxTokensPerGen();
        // To avoid the whitelist, we will warp by 1 day
        uint256 warpBy = 1 days + 1;
        vm.warp(block.timestamp + warpBy);
        bytes32[] memory proof = new bytes32[](0);
        uint256 count = 0;

        vm.startPrank(user1);
        // Mint 1k Nfts
        // This allows us to move from the first generation to the second generation
        while (count < nftsToMint) {
            // `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` both use the
            // `TraitForgeNft::calculateMintPrice()` function to calculate the price for the next mint
            uint256 ethToSupply = traitForgeNft.calculateMintPrice();
            traitForgeNft.mintToken{value: ethToSupply}(proof);
            ++count;
        }
        vm.stopPrank();

        uint256 priceForNextMint = traitForgeNft.calculateMintPrice();

        // Now let's try minting the first token of the second generation
        // This will succeed
        vm.startPrank(user1);
        traitForgeNft.mintToken{value: priceForNextMint}(proof);
        vm.stopPrank();
    }
```

The test passes with the following logs,

```shell
Ran 1 test for test/POCs.sol:POCs
[PASS] testMitigation() (gas: 280966635)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 121.13ms (112.62ms CPU time)

Ran 1 test suite in 122.24ms (121.13ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## <a id='H-02'></a>H-01. Lack of check for `maxGeneration` while incrementing generation allows minting of NFTs beyond the maximum cap

## Summary

The absence of a check for `maxGeneration` in `TraitForgeNft::_incrementGeneration` allows for the potential minting of tokens beyond the intended maximum cap of 10 generations (or as dictated by `setMaxGeneration`). This oversight could lead to unintended consequences, including dilution of token value and Nuke Fund rewards.
 
## Vulnerability Details

The core issue stems from the omission of a condition that would verify whether the current generation count (`currentGeneration`) surpasses the maximum allowable generations (`maxGeneration`). Without this safeguard, the contract fails to enforce the limit on the number of tokens that can be minted, thereby permitting unrestricted minting beyond the designated cap. This instance can be seen below:

```js
    function _incrementGeneration() private {
        // @audit only checks if all tokens have been minted for the current generation
@>      require(generationMintCounts[currentGeneration] >= maxTokensPerGen, "Generation limit not yet reached");
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;
        priceIncrement = priceIncrement + priceIncrementByGen;
        entropyGenerator.initializeAlphaIndices();
        emit GenerationIncremented(currentGeneration);
    }
```
 
* https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L347
 
NOTE:
 
1. This is not an issue for `forge` as there is a check established to handle this situation.

```js
    function forge(address newOwner, uint256 parent1Id, uint256 parent2Id, string memory)
        external
        whenNotPaused
        nonReentrant
        returns (uint256)
    {
        require(msg.sender == address(entityForgingContract), "unauthorized caller");
        uint256 newGeneration = getTokenGeneration(parent1Id) + 1;

        /// Check new generation is not over maxGeneration
@>      require(newGeneration <= maxGeneration, "can't be over max generation");

        // Calculate the new entity's entropy
        (uint256 forgerEntropy, uint256 mergerEntropy) = getEntropiesForTokens(parent1Id, parent2Id);
        // ...
    }
```

2. `TraitForgeNft::mintWithBudget` doesn't have this issue either since `mintWithBudget` can only be used to mint tokens for the first generation because of the conditional below: (This is an entire issue on its own)

```js
    function mintWithBudget(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
        // ...

        // @audit only works for the first generation as _tokenIds exceeds maxTokensPerGen
@>      while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
            _mintInternal(msg.sender, mintPrice);
            amountMinted++;
            budgetLeft -= mintPrice;
            mintPrice = calculateMintPrice();
        }

        // ...
    }
```

Although `mintWithBudegt` would also be affected along with `mintToken` if the bug mentioned above was fixed.

## Impact

This vulnerability presents significant risks to the economics, integrity and stability of the protocol. By facilitating unlimited minting, it could cause:

* **Economic Imbalance**: Excessive minting could devalue existing NFTs by increasing the overall supply, affecting market dynamics and pricing and also cause market saturation.
* **Bad reputation**: - Undermine trust in the project's economic model.
* **Dilution of Nuke Fund**: Holders would be discouraged from nuking their tokens due to the dilution of nuke rewards.

## Proof of Code

To write the PoC, some changes had to be made to the `TraitForgeNft` contract to stay within the block gas limit and avoid a revert due to another issue highlighted below.

The following changes were made to write the PoC for this finding:

1. Changed the `TraitForgeNft::maxTokensPerGen` variable from `10,000` to `10`, to avoid `OutOfGas` error while minting entire generations of tokens.

```diff
contract TraitForgeNft is ITraitForgeNft, ERC721Enumerable, ReentrancyGuard, Ownable, Pausable {
    // Constants for token generation and pricing
-   uint256 public maxTokensPerGen = 10000;
+   uint256 public maxTokensPerGen = 10;
    uint256 public startPrice = 0.005 ether;
```

2. Commented out the following line from `TraitForgeNft::_incrementGeneration()` function because it causes a revert, and is a completely different finding on its own. Here, the `TraitForgeNft` contract tries to re-initialize the alpha indices in the `EntropyGenerator` contract -- the slots which hold the God entropy value of `999999`. However, it will always revert because `TraitForgeNft` isn't the owner of `EntropyGenerator` contract and `EntropyGenerator::initializeAlphaIndices()` function has an `onlyOwner` modifier.

```js
    function _incrementGeneration() private {
        require(generationMintCounts[currentGeneration] >= maxTokensPerGen, "Generation limit not yet reached");
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;
        priceIncrement = priceIncrement + priceIncrementByGen;
@>      // entropyGenerator.initializeAlphaIndices();
        emit GenerationIncremented(currentGeneration);
    }
```

As the POC is written in Foundry, you will have to carry out the steps listed below:

1. Initialize Foundry within the repository by following the steps from [Foundry docs](https://book.getfoundry.sh/config/hardhat?highlight=hardha#adding-foundry-to-a-hardhat-project)
2. Inside the `test` folder, create a file named `TraitForgeNft.t.sol` and add the following lines of code to import the required files:

```js
// SPDX-License-Identifier: MIT
pragma solidity 0.8.20;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {EntityForging} from "contracts/EntityForging/EntityForging.sol";
import {IEntityForging} from "contracts/EntityForging/IEntityForging.sol";
import {TraitForgeNft} from "contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "contracts/EntropyGenerator/EntropyGenerator.sol";
import {Airdrop} from "contracts/Airdrop/Airdrop.sol";
import {DevFund} from "contracts/DevFund/DevFund.sol";
import {NukeFund} from "contracts/NukeFund/NukeFund.sol";
import {EntityTrading} from "contracts/EntityTrading/EntityTrading.sol";
import {IEntityTrading} from "contracts/EntityTrading/IEntityTrading.sol";
```

3. Add the following testing contract with the required state variables and a `setUp()` function which is called before every test execution:

```js
contract EntityForgingTest is Test {
    EntityForging entityForging;
    TraitForgeNft traitForgeNft;
    EntropyGenerator entropyGenerator;
    Airdrop airdrop;
    DevFund devFund;
    NukeFund nukeFund;
    EntityTrading entityTrading;

    address owner = makeAddr("owner");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");

    function setUp() public {
        vm.startPrank(owner);

        // Deploy TraitForgeNft contract
        traitForgeNft = new TraitForgeNft();

        // Deploy Airdrop contract
        airdrop = new Airdrop();
        traitForgeNft.setAirdropContract(address(airdrop));
        airdrop.transferOwnership(address(traitForgeNft));

        // Deploy entropyGenerator contract
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entropyGenerator.writeEntropyBatch1();
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));

        // Deploy EntityForging contract
        entityForging = new EntityForging(address(traitForgeNft));
        traitForgeNft.setEntityForgingContract(address(entityForging));

        // Deploy DevFund and NukeFund contract
        devFund = new DevFund();
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(address(devFund)), payable(owner));
        traitForgeNft.setNukeFundContract(payable(address(nukeFund)));

        // Deploy EntityTrading contract
        entityTrading = new EntityTrading(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        vm.stopPrank();

        vm.deal(player1, 100 ether);
        vm.deal(player2, 10 ether);

        // Used to avoid whitelist proof
        vm.warp(86402);
        vm.roll(86402);
    }

    // Paste POC tests here
}
```

4. Now add the following test to the file:

```js
function test_MintToken_MoreThan_MaxGenerations_Allowed() public {
    // to pass as argument to the mint function
    bytes32[] memory proof;

    // 10 tokens per gen, last 10 tokens will be from the 11th gen
    vm.startPrank(player1);
    for (uint256 i = 1; i <= 110; ++i) {
        traitForgeNft.mintToken{value: 1 ether}(proof);
    }
    vm.stopPrank();

    uint256 genNumberReached = traitForgeNft.currentGeneration();
    console.log("Generation Minted -", genNumberReached);
    // assert that max generations limit has been crossed
    assert(genNumberReached >traitForgeNft.maxGeneration());
}
```

5. Run the test by using command `forge test --mt test_MintToken_MoreThan_MaxGenerations_Allowed -vv`
6. The test passes with the following logs:

```shell
Ran 1 test for test/foundry-tests/TraitForgeNft.t.sol:EntityForgingTest
[PASS] test_MintToken_MoreThan_MaxGenerations_Allowed() (gas: 32102579)
Logs:
  Generation Minted - 11

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 16.41ms (12.42ms CPU time)
```

## Recommended Mitigation

To address this vulnerability, the function must incorporate a check to ensure that the current generation count does not exceed the maximum allowed. This can be done by making the following changes to the `_incrementGeneration` function:

```diff
    function _incrementGeneration() private {
+       /// Check current generation would not cross maxGeneration even after incrementing
+       require(currentGeneration + 1 <= maxGeneration, "Max Generations minted!");
        require(generationMintCounts[currentGeneration] >= maxTokensPerGen, "Generation limit not yet reached");
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;
        priceIncrement = priceIncrement + priceIncrementByGen;
        // entropyGenerator.initializeAlphaIndices();
        emit GenerationIncremented(currentGeneration);
    }
```

Use the following unit test to test against the changes made:

```js
    function test_Mitigation() public {
        // to pass as argument to the mint function
        bytes32[] memory proof;

        // 10 tokens per gen, for 10 generations
        vm.startPrank(player1);
        for (uint256 i = 1; i <= 100; ++i) {
            traitForgeNft.mintToken{value: 1 ether}(proof);
        }

        // minting reverts as expected
        vm.expectRevert("Max Generations minted!");
        traitForgeNft.mintToken{value: 1 ether}(proof);

        vm.stopPrank();
    }
```

Test passes with the following logs:

```shell
Ran 1 test for test/foundry-tests/TraitForgeNft.t.sol:EntityForgingTest
[PASS] test_Mitigation() (gas: 29215276)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.81ms (15.49ms CPU time)
```

## Tools Used

Manual Review and Foundry for POC


# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of functions to pause or unpause in all contracts implementing the Openzeppelin Pausable plugin

## Summary

All contracts implementing the `Pausable` plugin in the TratiForge protocol do not have admin-restricted functions to pause/unpause the contracts.

## Vulnerability Details

The following contracts implement the `Pausable` plugin,

1. DevFund
2. EntityForging
3. EntityTrading
4. EntropyGenerator
5. NukeFund
6. TraitForgeNft

The following functions cannot be paused/unpaused,

`DevFund::claim()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/DevFund/DevFund.sol#L61

`EntityForging::listForForging()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L67

`EntityForging::forgeWithListed()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L102

`EntityForging::cancelListingForForging()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityForging/EntityForging.sol#L177

`EntityTrading::listNFTForSale()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L38

`EntityTrading::buyNFT()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L63

`EntityTrading::cancelListing()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntityTrading/EntityTrading.sol#L94

`EntropyGenerator::initializeAlphaIndices()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206

`NukeFund::nuke()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153

`TraitForgeNft::startAirdrop()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L96

`TraitForgeNft::burn()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L141

`TraitForgeNft::forge()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L153

`TraitForgeNft::mintToken()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L181

`TraitForgeNft::mintWithBudget()` -- https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L202

Though these contracts use the `Pausable` plugin, there exist no admin-restricted functions to pause/unpause in case of an exploit.

## Impact

The pause/unpause mechanism exists to pause the protocol's core functionalities (minting, nuking, burning, listing, cancelling listing, forging, cancelling forging, etc) in case of an exploit or issue so as to protect user's funds. If an unforseen situation arises, or the protocol is exploited, the protocol team won't be able to pause it. Also, this breaks the protocol's intended way of functioning. Various functions throughout the contracts implement the `whenNotPaused` modifier, so it is understood that the protocol team intended to include this functionality.

## Tools Used

Manual review.

## Recommended Mitigation

Add the following 2 functions to pause and unpause all contracts implementing the `Pausable` plugin. All pausable contracts should also use the `Ownable` plugin so that only the owner can pause/unpause the contracts.

```diff
+   function pauseContract() external onlyOwner {
+       _pause();
+   }

+   function unpauseContract() external onlyOwner {
+       _unpause();
+   }
```

## <a id='M-02'></a>M-02. Incorrect higher mint price for the first token minted in each generation starting from the second generation

## Summary

Users will need to pay a higher price to mint the first token for each generation starting from generaton 2, instead of the `0.005 ether` starting price. This is because the `TraitForgeNft.currentGeneration` variable isn't incremented on the `10_000` th mint of each generation. It is incremented on the `10_001` th mint (basically, the first mint of the next generation). So on the first mint of the next generation, the `TraitForgeNft` contract still assumes to be in the last generation and calculates a wrong mint price which is higher than `0.005 ether`. Only then is the `TraitForgeNft.currentGeneration` variable incremented. This is applicable to both `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` functions.

## Vulnerability Details

Let's see two code segments, one from `TraitForgeNft::mintToken()` and the other from `TraitForgeNft::_mintInternal()` function,

```js
    function mintToken(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
@>      uint256 mintPrice = calculateMintPrice();
        require(msg.value >= mintPrice, "Insufficient ETH send for minting.");

@>      _mintInternal(msg.sender, mintPrice);

        // more code here
    }
```

The price for minting is calculated first.

```js
    function _mintInternal(address to, uint256 mintPrice) internal {
        if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
@>          _incrementGeneration();
        }

        _tokenIds++;
        uint256 newItemId = _tokenIds;
        _mint(to, newItemId);
        uint256 entropyValue = entropyGenerator.getNextEntropy();

        tokenCreationTimestamps[newItemId] = block.timestamp;
        tokenEntropy[newItemId] = entropyValue;
        tokenGenerations[newItemId] = currentGeneration;
@>      generationMintCounts[currentGeneration]++;
        initialOwners[newItemId] = to;

        // more code here
    }
```

**Links**:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L190

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L193

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L282.

The current generation is incremented after price calculation for the mint. Thus, while moving from one generation to the next, the contract still thinks it's in the last generation during the price calculation phase, and gives an incorrect high price when it should be `0.005 ether` instead. Then the `TraitForgeNft.currentGeneration` value is incremented and the contract functions as expected from the second mint of the next generation.

This issue is applicable to `TraitForgeNft::mintWithBudget()` function as well. However, it currently has a bug which doesn't allow minting beyond the first generation.

```js
    function mintWithBudget(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
        uint256 mintPrice = calculateMintPrice();
        
        // more code here

@>      while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
            _mintInternal(msg.sender, mintPrice);
            amountMinted++;
            budgetLeft -= mintPrice;
            mintPrice = calculateMintPrice();
        }
        
        // more code here
    }
```

**Link**:

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L215

If the number of token ids minted goes beyond `TraitForgeNft.maxTokensPerGen`, the function will not mint further, however, this is a finding on its own. If this was to be resolved, the function would also be susceptible to the same problem while minting the first token in each generation starting from generation 2.

## Impact

Users will need to pay an incorrect/high amount for minting the first token for each generation starting from generation 2 instead of the `0.005 ether` starting price. This essentially causes a fund loss for users (paying higher instead of 0.005 ether). Also, the amount to pay for the first token of each generation continues to become larger and larger as the generations progress (due to the `TraitForgeNft.priceIncrement` variable increasing by `TraitForgeNft.priceIncrementByGen` each generation). However, the fund loss is only limited to 9 mints upto the 10th generation.

## Proof of Concept

To write the PoC, some changes had to be made to the `TraitForgeNft` contract to stay within the block gas limit and avoid a revert due to another issue highlighted below.

The following changes were made to write the PoC for this finding.

Changed the `TraitForgeNft.maxTokensPerGen` variable from `1e4` to `1e3`, to avoid `OutOfGas` error.

Commented out the following line from `TraitForgeNft::_incrementGeneration()` function because it causes a revert, and is a completely different finding on its own. Here, the `TraitForgeNft` contract tries to re-initialize the alpha indices in the `EntropyGenerator` contract -- the slots which hold the God entropy value of `999999`. However, it will always revert because `TraitForgeNft` isn't the owner of `EntropyGenerator` contract and `EntropyGenerator::initializeAlphaIndices()` function has an `onlyOwner` modifier.

```js
    function _incrementGeneration() private {
        require(generationMintCounts[currentGeneration] >= maxTokensPerGen, "Generation limit not yet reached");
        currentGeneration++;
        generationMintCounts[currentGeneration] = 0;
        priceIncrement = priceIncrement + priceIncrementByGen;
@>      entropyGenerator.initializeAlphaIndices();
        emit GenerationIncremented(currentGeneration);
    }
```

We will be using the following setup to write PoCs in Foundry,

```js
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {console} from "lib/forge-std/src/console.sol";
import {Test} from "lib/forge-std/src/Test.sol";

import {Trait} from "../contracts/Trait/Trait.sol";
import {TraitForgeNft} from "../contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "../contracts/EntropyGenerator/EntropyGenerator.sol";
import {EntityTrading} from "../contracts/EntityTrading/EntityTrading.sol";
import {EntityForging} from "../contracts/EntityForging/EntityForging.sol";
import {DevFund} from "../contracts/DevFund/DevFund.sol";
import {Airdrop} from "../contracts/Airdrop/Airdrop.sol";
import {DAOFund} from "../contracts/DAOFund/DAOFund.sol";
import {NukeFund} from "../contracts/NukeFund/NukeFund.sol";

contract POCs is Test {
    address public owner;
    address public user1;

    address public dev1;
    address public dev2;
    address public dev3;

    Trait public traitToken;
    TraitForgeNft public traitForgeNft;
    EntropyGenerator public entropyGenerator;
    EntityTrading public entityTrading;
    EntityForging public entityForging;
    DevFund public devFund;
    Airdrop public airdrop;
    DAOFund public daoFund;
    NukeFund public nukeFund;

    function setUp() public {
        owner = makeAddr("owner");
        user1 = makeAddr("user1");
        dev1 = makeAddr("dev1");
        dev2 = makeAddr("dev2");
        dev3 = makeAddr("dev3");

        string memory name = "Trait";
        string memory symbol = "TRT";
        uint8 decimals = 18;
        uint256 tokenTotalSupply = 10_000e18;
        address uniswapV3Router = makeAddr("uniswap v3 router");

        vm.startPrank(owner);
        traitToken = new Trait(name, symbol, decimals, tokenTotalSupply);
        traitForgeNft = new TraitForgeNft();
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entityTrading = new EntityTrading(address(traitForgeNft));
        entityForging = new EntityForging(address(traitForgeNft));
        devFund = new DevFund();
        airdrop = new Airdrop();
        daoFund = new DAOFund(address(traitToken), uniswapV3Router);
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(devFund), payable(daoFund));

        traitForgeNft.setEntropyGenerator(address(entropyGenerator));
        traitForgeNft.setEntityForgingContract(address(entityForging));
        traitForgeNft.setNukeFundContract(payable(nukeFund));
        traitForgeNft.setAirdropContract(address(airdrop));
        entropyGenerator.setAllowedCaller(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(nukeFund));
        entityForging.setNukeFundAddress(payable(nukeFund));
        airdrop.setTraitToken(address(traitToken));
        airdrop.transferOwnership(address(traitForgeNft));

        uint256 weights = 100;
        devFund.addDev(dev1, weights);
        devFund.addDev(dev2, weights);
        devFund.addDev(dev3, weights);

        entropyGenerator.writeEntropyBatch1();
        entropyGenerator.writeEntropyBatch2();
        entropyGenerator.writeEntropyBatch3();
        vm.stopPrank();
    }

    // PoCs go here
}
```

Now run the following test case,

```js
    function testIncorrectPriceCalculatedForFirstMintOfEachGenerationStartingfromGeneration2() public {
        // Provide some ether to user1
        uint256 maxEthToDeal = 1000 ether;
        vm.deal(user1, maxEthToDeal);

        uint256 startPriceForEachGen = traitForgeNft.startPrice();
        // Changed this value to 1_000 Nfts in the TraitForgeNft contract to stay within the block gas limit
        uint256 nftsToMint = traitForgeNft.maxTokensPerGen();
        // To avoid the whitelist, we will warp by 1 day
        uint256 warpBy = 1 days + 1;
        vm.warp(block.timestamp + warpBy);
        bytes32[] memory proof = new bytes32[](0);
        uint256 count = 0;

        vm.startPrank(user1);
        // Mint 1k Nfts
        // This allows us to move from the first generation to the second generation
        while (count < nftsToMint) {
            // `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` both use the
            // `TraitForgeNft::calculateMintPrice()` function to calculate the price for the next mint
            uint256 ethToSupply = traitForgeNft.calculateMintPrice();
            traitForgeNft.mintToken{value: ethToSupply}(proof);
            ++count;
        }
        vm.stopPrank();

        uint256 expectedPriceForNextMint = startPriceForEachGen;
        uint256 userEthBalanceBefore = user1.balance;
        uint256 userEthBalanceAfter;
        // This is the mint price for the first mint in the second generation
        // It is expected to be 0.005 ether
        uint256 actualEthPriceToSend = traitForgeNft.calculateMintPrice();

        vm.startPrank(user1);
        traitForgeNft.mintToken{value: actualEthPriceToSend}(proof);
        vm.stopPrank();

        userEthBalanceAfter = user1.balance;
        // This is going to be 5000000000000000 (5e15/0.005 ether)
        console.log("Expected mint price for the first token in generation 2: ", expectedPriceForNextMint);
        // This is going to be 29500000000000000 (29.5e15/ 0.0295 ether)
        console.log("Actual mint price for the first token in generation 2: ", actualEthPriceToSend);
        // However, remember that this is for 1000 tokens per generation and the second generation first token mint
        // For 10k tokens per generation and higher generations (with higher price increment), the value will be larger
        // Calculating some:
        // Mint price for first token of second generation: 250000000000000000 (0.25 ether)
        // Mint price for first token of third generation: 300000000000000000 (0.3 ether)
        // And so on

        assert(actualEthPriceToSend > expectedPriceForNextMint);
        assert((userEthBalanceBefore - userEthBalanceAfter) > expectedPriceForNextMint);
    }
```

The test passes with the following logs,

```shell
Ran 1 test for test/POCs.sol:POCs
[PASS] testWrongPriceCalculatedForFirstMintOfEachGenStartingfromGen2() (gas: 280943740)
Logs:
  Expected mint price for the first token in generation 2:  5000000000000000
  Actual mint price for the first token in generation 2:  29500000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 94.33ms (90.61ms CPU time)

Ran 1 test suite in 102.50ms (94.33ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review. Foundry for writing the PoC.

## Recommended Mitigation

Increment the generation before calculating the price.

Make the following changes in `TraitForgeNft::mintToken()` function,

```diff
    function mintToken(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
+       if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
+           _incrementGeneration();
+       }
        uint256 mintPrice = calculateMintPrice();
        require(msg.value >= mintPrice, "Insufficient ETH send for minting.");

        // more code here
    }
```

Remember, the finding is also valid for `TraitForgeNft::mintWithBudget()` function, so changes need to be made there as well.

```diff
    function mintWithBudget(bytes32[] calldata proof)
        public
        payable
        whenNotPaused
        nonReentrant
        onlyWhitelisted(proof, keccak256(abi.encodePacked(msg.sender)))
    {
-       uint256 mintPrice = calculateMintPrice();
+       uint256 mintPrice;
        uint256 amountMinted = 0;
        uint256 budgetLeft = msg.value;

        while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
+           if (generationMintCounts[currentGeneration] >= maxTokensPerGen) {
+              _incrementGeneration();
+           }
+           mintPrice = calculateMintPrice();
            _mintInternal(msg.sender, mintPrice);
            amountMinted++;
            budgetLeft -= mintPrice;
-           mintPrice = calculateMintPrice();
        }
        if (budgetLeft > 0) {
            (bool refundSuccess,) = msg.sender.call{value: budgetLeft}("");
            require(refundSuccess, "Refund failed.");
        }
    }
```

Let's unit test the `TraitForgeNft::mintToken()` function (in Foundry),

```js
    function testMitigation() public {
        // Provide some ether to user1
        uint256 maxEthToDeal = 1000 ether;
        vm.deal(user1, maxEthToDeal);

        uint256 startPriceForEachGen = traitForgeNft.startPrice();
        // Changed this value to 1_000 Nfts in the TraitForgeNft contract to stay within the block gas limit
        uint256 nftsToMint = traitForgeNft.maxTokensPerGen();
        // To avoid the whitelist, we will warp by 1 day
        uint256 warpBy = 1 days + 1;
        vm.warp(block.timestamp + warpBy);
        bytes32[] memory proof = new bytes32[](0);
        uint256 count = 0;

        vm.startPrank(user1);
        // Mint 1k Nfts
        // This allows us to move from the first generation to the second generation
        while (count < nftsToMint) {
            // `TraitForgeNft::mintToken()` and `TraitForgeNft::mintWithBudget()` both use the
            // `TraitForgeNft::calculateMintPrice()` function to calculate the price for the next mint
            uint256 ethToSupply = traitForgeNft.calculateMintPrice();
            traitForgeNft.mintToken{value: ethToSupply}(proof);
            ++count;
        }
        vm.stopPrank();

        uint256 expectedPriceForNextMint = startPriceForEachGen;
        uint256 userEthBalanceBefore = user1.balance;
        uint256 userEthBalanceAfter;

        vm.startPrank(user1);
        traitForgeNft.mintToken{value: expectedPriceForNextMint}(proof);
        vm.stopPrank();

        userEthBalanceAfter = user1.balance;
        // Both the expected and actual mint price will be 0.005 ether
        console.log("Expected mint price for the first token in generation 2: ", expectedPriceForNextMint);
        console.log(
            "Actual mint price for the first token in generation 2: ", (userEthBalanceBefore - userEthBalanceAfter)
        );

        assertEq((userEthBalanceBefore - userEthBalanceAfter), expectedPriceForNextMint);
    }
```

The unit test passes with the following logs,

```shell
Ran 1 test for test/POCs.sol:POCs
[PASS] testMitigation() (gas: 281335207)
Logs:
  Expected mint price for the first token in generation 2:  5000000000000000
  Actual mint price for the first token in generation 2:  5000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 131.10ms (122.20ms CPU time)

Ran 1 test suite in 141.59ms (131.10ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## <a id='M-03'></a>M-03. Different sets of parent tokens can evaluate to the same average entropy value which takes away the uniqueness of tokens

## Summary

Multiple different sets of forger and merger tokens with the same average entropy value can forge together to form a new entity with the same entropy value as that of another entity. This is also possible when the same set of parent tokens forge multiple times, all their child tokens will have the same entropy and hence the same characteristics which doesn't allow them to be unique.

## Vulnerability Details

TraitForge NFTs are supposed to have unique traits dictated by their entropy values as stated in the [docs](https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/README.md?plain=1#L25):

```
Each digit or total entropy, gives the NFT its unqique traits, that are determined but completely dispursed/random.
```

NFTs belonging to the same generation are supposed to have distinct entropies and characteristics, this is prevented by the fact that child entropy values are derived by averaging the entropy values of the parent tokens in the `TraitForgeNft::forge()` function.

```js
    function forge(address newOwner, uint256 parent1Id, uint256 parent2Id, string memory)
        external
        whenNotPaused
        nonReentrant
        returns (uint256)
    {
        require(msg.sender == address(entityForgingContract), "unauthorized caller");
        uint256 newGeneration = getTokenGeneration(parent1Id) + 1;

        /// Check new generation is not over maxGeneration
        require(newGeneration <= maxGeneration, "can't be over max generation");

        // Calculate the new entity's entropy
        (uint256 forgerEntropy, uint256 mergerEntropy) = getEntropiesForTokens(parent1Id, parent2Id);
@>      uint256 newEntropy = (forgerEntropy + mergerEntropy) / 2;

        // Mint the new entity
        uint256 newTokenId = _mintNewEntity(newOwner, newEntropy, newGeneration);

        return newTokenId;
    }
```

* https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L173

The formula used above is essentially `z = (x+y)/2`, with many `x` and `y` values which result in the same `z` value.

## Impact

Trait Forge NFTs starting from the second generation (as first generation tokens cannot be forged into existence) may see a lot of tokens with the same entropy values. This possibility is higher for tokens with high forge factors and those which will result in better "genetics" for the newer generations. NFTs would no longer be as unique as the first generation and would have a lot of clones.

## Proof of Concept

Let's understand the issue with the help of an example:

Case 1.

1. Two sets of parent tokens exist with entropy values as (not considering they satisfy the forger/merger criterion)-
   
   * Parent1 = 123456
   * Parent2 = 789123
   * Parent3 = 213546
   * Parent4 = 699033
2. Parent1 and Parent2 forge together and Parent3 and Parent4 forge together resulting in child tokens with the same entropy values-
   
   * ChildEntropy1 = ChildEntropy2 = 4,56,289

Case 2. The same set of parent tokens forge more than once, resulting in the same child entropy values.

**Proof of Code**: As the POC is written in Foundry, you will have to carry out the steps listed below:

1. Initialize Foundry within the repository by following the steps from [Foundry docs](https://book.getfoundry.sh/config/hardhat?highlight=hardha#adding-foundry-to-a-hardhat-project)
2. Inside the `test` folder, create a file named `EntropyForging.t.sol` and add the following lines of code to import the required files:

```js
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import {EntityForging} from "contracts/EntityForging/EntityForging.sol";
import {IEntityForging} from "contracts/EntityForging/IEntityForging.sol";
import {TraitForgeNft} from "contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "contracts/EntropyGenerator/EntropyGenerator.sol";
import {Airdrop} from "contracts/Airdrop/Airdrop.sol";
import {DevFund} from "contracts/DevFund/DevFund.sol";
import {NukeFund} from "contracts/NukeFund/NukeFund.sol";
import {EntityTrading} from "contracts/EntityTrading/EntityTrading.sol";
import {IEntityTrading} from "contracts/EntityTrading/IEntityTrading.sol";
```

3. Add the following testing contract with the required state variables and a `setUp()` function which is called before every test execution:

```js
contract EntityForgingTest is Test {
    EntityForging entityForging;
    TraitForgeNft traitForgeNft;
    EntropyGenerator entropyGenerator;
    Airdrop airdrop;
    DevFund devFund;
    NukeFund nukeFund;
    EntityTrading entityTrading;

    address owner = makeAddr("owner");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");

    function setUp() public {
        vm.startPrank(owner);

        // Deploy TraitForgeNft contract
        traitForgeNft = new TraitForgeNft();

        // Deploy Airdrop contract
        airdrop = new Airdrop();
        traitForgeNft.setAirdropContract(address(airdrop));
        airdrop.transferOwnership(address(traitForgeNft));

        // Deploy entropyGenerator contract
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entropyGenerator.writeEntropyBatch1();
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));

        // Deploy EntityForging contract
        entityForging = new EntityForging(address(traitForgeNft));
        traitForgeNft.setEntityForgingContract(address(entityForging));

        // Deploy DevFund and NukeFund contract
        devFund = new DevFund();
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(address(devFund)), payable(owner));
        traitForgeNft.setNukeFundContract(payable(address(nukeFund)));

        // Deploy EntityTrading contract
        entityTrading = new EntityTrading(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        vm.stopPrank();

        vm.deal(player1, 100 ether);
        vm.deal(player2, 10 ether);

        // Used to avoid whitelist proof
        vm.warp(86402);
        vm.roll(86402);
    }

    // Paste POC tests here
}
```

4. Now add the following test to the file:

```js
    function testDuplicateEntropies() public {
        // to pass as argument to the mint function
        bytes32[] memory proof;

        // mint 5 times to reach the first forger token
        vm.startPrank(player1);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);

        // player1 lists tokenId = 5 for forging
        entityForging.listForForging(5, 1 ether);
        vm.stopPrank();

        // player2 mints tokenId = 6
        vm.startPrank(player2);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        // forges with player1's listing, child tokenId = 7
        entityForging.forgeWithListed{value: 1 ether}(5, 6);
        vm.stopPrank();

        // the two parent tokens forge again, child tokenId = 8
        vm.prank(player1);
        entityForging.listForForging(5, 1 ether);
        vm.prank(player2);
        entityForging.forgeWithListed{value: 1 ether}(5, 6);

        uint256 childEntropy1 = traitForgeNft.getTokenEntropy(7);
        uint256 childEntropy2 = traitForgeNft.getTokenEntropy(8);

        console.log("Entropy for Child Token 1 -", childEntropy1);
        console.log("Entropy for Child Token 2 -", childEntropy2);
        assertEq(childEntropy1, childEntropy2);
    }
```

5. Run the test by using command `forge test --mt testDuplicateEntropies -vv`
6. The test passes with the following logs:

```shell
Ran 1 test for test/foundry-tests/EntityForging.t.sol:EntityForgingTest
[PASS] testDuplicateEntropies() (gas: 2761829)
Logs:
  Entropy for Child Token 1 - 814035
  Entropy for Child Token 2 - 814035

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.15ms (5.35ms CPU time)
```

## Recommended Mitigation

Add a mapping that tracks whether a token with given entropy from a specific generation has already been minted and revert when an attempt is made to mint another token with the same entropy value.

Update the `TraitForgeNft` contract as shown below:

```diff
contract TraitForgeNft is ITraitForgeNft, ERC721Enumerable, ReentrancyGuard, Ownable, Pausable {
    // ...

    mapping(uint256 => uint256) public tokenGenerations;
    mapping(uint256 => address) public initialOwners;
+   // tracked from second generation, first gen is always unique
+   mapping(uint256 generation => mapping(uint256 entropy => uint256 tokenId)) public entropyToTokenIdPerGen;

    // ...

    function _mintInternal(address to, uint256 mintPrice) internal {
        // ...
        uint256 entropyValue = entropyGenerator.getNextEntropy();

+       // to avoid minting new tokens with entropy of existing forged token
+       if (entropyToTokenIdPerGen[currentGeneration][entropyValue] != 0) {
+           entropyValue = entropyGenerator.getNextEntropy();
+       }

        tokenCreationTimestamps[newItemId] = block.timestamp;
        tokenEntropy[newItemId] = entropyValue;
+       // track minted tokens with specific entropy value
+       entropyToTokenIdPerGen[currentGeneration][entropyValue] = newItemId;

        // ...
    }

    function _mintNewEntity(address newOwner, uint256 entropy, uint256 gen) private returns (uint256) {
        require(generationMintCounts[gen] < maxTokensPerGen, "Exceeds maxTokensPerGen");
+       // revert if entity with same entropy exists
+       require(entropyToTokenIdPerGen[gen][entropy] == 0, "Entity with the same entropy already exists!");

        _tokenIds++;
        uint256 newTokenId = _tokenIds;
        _mint(newOwner, newTokenId);

        tokenCreationTimestamps[newTokenId] = block.timestamp;
        tokenEntropy[newTokenId] = entropy;

+       // track minted tokens with specific entropy value
+       entropyToTokenIdPerGen[gen][entropy] = newTokenId;

        // ...
    }
}
```

Use the following unit test to test against the changes made above:

```js
    function test_Mitigation() public {
        // to pass as argument to the mint function
        bytes32[] memory proof;

        // mint 5 times to reach the first forger token
        vm.startPrank(player1);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        traitForgeNft.mintToken{value: 1 ether}(proof);

        // player1 lists tokenId = 5 for forging
        entityForging.listForForging(5, 1 ether);
        vm.stopPrank();

        // player2 mints tokenId = 6
        vm.startPrank(player2);
        traitForgeNft.mintToken{value: 1 ether}(proof);
        // forges with player1's listing, child tokenId = 7
        entityForging.forgeWithListed{value: 1 ether}(5, 6);
        vm.stopPrank();

        // the two parent tokens attempt to forge again
        vm.prank(player1);
        entityForging.listForForging(5, 1 ether);
        // forging reverts as expected
        vm.expectRevert("Entity with the same entropy already exists!");
        vm.prank(player2);
        entityForging.forgeWithListed{value: 1 ether}(5, 6);
    }
```

The test passes with the following logs:

```shell
Ran 1 test for test/foundry-tests/EntityForging.t.sol:EntityForgingTest
[PASS] test_Mitigation() (gas: 2743897)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.39ms (4.91ms CPU time)
```

## Tools Used

Manual Review and Foundry for POC.