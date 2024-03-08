# First Flight #10: One Shot - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Weak randomness in `RapBattle::_battle` function allows the challenger to predict the winner beforehand](#H-01)
    - ### [H-02. The challenger can win a rap battle without placing the credibility token bet](#H-02)
    - ### [H-03. No check for ownership of rapper tokenId leads to attackers entering rap battles with someone else's tokenId and being able to win defender's credibility token bet](#H-03)

- ## Low Risk Findings
    - ### [L-01. The battlesWon property of Nft metadata of the winning rapper in a rap battle is never updated](#L-01)
    - ### [L-02. The `RapBattle::Battle` event is triggered with wrong data (wrong winner) if the random value is equal to the defenderRapperSkill](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

Stood 9th out of a total of 77 participants.

### Number of findings:

   - High: 3
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Weak randomness in `RapBattle::_battle` function allows the challenger to predict the winner beforehand            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

## Summary

Weak randomness in `RapBattle::_battle` function allows the challenger to predict the winner beforehand. The challenger can simulate a scenario where he/she enters at the right, pre-determined moment to win the rap battle.

## Vulnerability Details

Blockchain is a deterministic entity. Thus, using any of the blockchain specific variables to generate random values produces weak randomness. The random value can be predicted beforehand and the result influenced.

In `RapBattle::_battle` function, the random value to pick a winner is calculated by hashing block.timestamp, block.prevrandao, and msg.sender, creating a predictable value, and thus, a predictable winner. A malicious challenger can calaculate the exact time and block difficulty when he/she could win the rap battle. This defeats the purpose of the protocol -- to provide fair rap battles to rappers.

## Impact

The ability to influence the outcome of the rap battle lies in the hands of the challengers. The defenders enter the rap battle beforehand, so they play no part in this exploit. The defenders are doomed to lose their credibility token bet.

## Tools Used

Foundry, VSCodium.

## Recommendations

Use a cryptographically provable random number generator such as Chainlink VRF.

## <a id='H-02'></a>H-02. The challenger can win a rap battle without placing the credibility token bet            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L73

## Summary

In a rap battle, if the challenger doesn't approve credibility tokens to the RapBattle contract, the battle  reverts if the defender is picked as the winner. However, if the challenger wins, he/she gets the credibility token bet placed by the defender. The challenger may enter the battle over and over again until he/she wins. This is a serious disruption of what the protocol intends to achieve.

## Vulnerability Details

The defender enters the battle by calling `RapBattle::goOnStageOrBattle`, transferring his/her rapper tokenId, and credibility token bet to the RapBattle contract. When a challenger enters next, he/she doesn't need to transfer his/her credibility token bet to the RapBattle contract. This allows the challenger to enter a rap battle without approving credibility tokens to the RapBattle contract. If the defender is picked as the winner, the defender never gets the part of credibility token bet from the challenger, because the transaction reverts. This is because the challenger never approved credibility tokens to the contract, so the following line fails in `RapBattle::_battle` function:

```javascript
73.    credToken.transferFrom(msg.sender, _defender, _credBet)
```

The challenger can keep calling `RapBattle::goOnStageOrBattle` over and over again, until he/she wins and receives the defender's credibility token bet.

## Impact

The challenger is guaranteed a win if he/she enters the battle over and over again without approving credibility token bet to the RapBattle contract. This also means that the challenger can enter rap battle without having any credibility tokens whatsoever. The rappers who enter the battle first are doomed to lose their credibility tokens with this exploit.

## Proof of Concept

1. Defender enters the rap battle by approving his/her rapper tokenId and credibility token bet to the RapBattle contract. The RapBattle contract transfers the tokenId and credibility token bet to itself.
2. The challenger enters the battle with his/her tokenId, and without approving any credibility tokens to the RapBattle contract. It is also possible that the challenger doesn't have any credibility tokens at all.
3. If the defender wins, the transaction reverts because the contract failed to transfer tokens from challenger to the defender.
4. If the challenger wins, he/she gets the credibility token bet placed by the defender.

Add the following function to your test contract in your Foundry test file.

```javascript
    function testChallengerGoesToBattle(uint256 time, bytes32 difficulty) public {
        // setup phase
        address defender = makeAddr("defender");
        address challenger = makeAddr("challenger");

        // both the challenger and the defender get a rapper Nft minted to them, and stake it to gain some credibility tokens
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.stopPrank();

        vm.startPrank(challenger);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 1);
        streets.stake(1);
        vm.stopPrank();

        vm.warp(4 days + 1);

        vm.prank(defender);
        streets.unstake(0);
        vm.prank(challenger);
        streets.unstake(1);

        console.log("defender's starting balance:", cred.balanceOf(defender)); // 4 tokens
        console.log("challenger's starting balance:", cred.balanceOf(challenger)); // 4 tokens

        // defender enters the battle, transferring his/her rapper tokenId and credibility token bet to the RapBattle contract
        vm.startPrank(defender);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(0, 1);
        vm.stopPrank();
        console.log("defender enters battle. Defender's current cred token balance: ", cred.balanceOf(defender)); // 3 tokens, 1 token transferred to rapBattle as bet

        // challenger doesn't approve his/her part of the credibility token bet, and yet, is able to enter the battle!
        vm.startPrank(challenger);
        vm.warp(time);
        vm.prevrandao(difficulty);
        rapBattle.goOnStageOrBattle(1, 1);
        vm.stopPrank();
        console.log(cred.balanceOf(challenger)); // if challenger keeps retrying, and then finally wins, challenger has 5 tokens
        // however, if the challenger doesn't win, the test fails
    }
```

With the randomized time and difficulty in the fuzz test, the challenger was able to successfully win 210 times  (using the seed "0x1"). The fuzz test reverted after 210 runs when the defender actually won, and the contract failed to transfer the credibility token bet from the challenger to the defender.

```shell
[FAIL. Reason: ERC20InsufficientAllowance(0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9, 0, 1); counterexample: calldata=0xe72e7d7a83f1c303821a4f3710a55512966f7dbf3494022843afe030110938b44ae86c023402084467cc14285f3d5a08090605290820549c984b662e4a1c0918250b0851 args=[59680139242138372652014777408658648486100020451655575976028705195302838627330 [5.968e76], 0x3402084467cc14285f3d5a08090605290820549c984b662e4a1c0918250b0851]] testChallengerGoesToBattle(uint256,bytes32) (runs: 210, μ: 598940, ~: 598940)
```

## Tools Used

Foundry, VSCodium.

## Recommendations

When the challenger goes to battle, the `RapBattle::goOnStageOrBattle` function should transfer the credibility token bet from the challenger to the contract. Only then the random winner should be picked.

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
+           credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

Or, the `RapBattle::goOnStageOrBattle` function should check if the challenger has approved credibility token bet to the contract before picking a random winner.

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
+           require(credToken.allowance(msg.sender, address(this)) == _credBet, "Challenger must approve tokens to the contract before going to battle");
            _battle(_tokenId, _credBet);
        }
    }
```

## <a id='H-03'></a>H-03. No check for ownership of rapper tokenId leads to attackers entering rap battles with someone else's tokenId and being able to win defender's credibility token bet            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L38

## Summary

An attacker who does not have a rapper tokenId minted to him/her can enter a rap battle as a challenger using someone else's tokenId, or a random, non-existent tokenId as well. If he/she wins, he/she gets the defender's credibility token bet. However, if the defender wins, the transaction reverts because the attacker never approved any credibility tokens to the contract (the attacker doesn't have any credibility tokens to begin with).

## Vulnerability details

The `RapBattle::goOnStageOrBattle` function doesn't check if the rapper tokenId is valid, nor does it check the ownership of the tokenId. This can lead to the following exploits:

1. Exisiting rappers (users of the protocol), can enter as challengers with someone else's rapper tokenId with maxed out stats to maximize their chance of winning (rapper skill is a factor in determining the random winner).
2. An attacker who doesn't assume a role in the protocol (doesn't have a rapper Nft minted to him/her), can enter a rap battle and win credibility tokens. This is made possible because the `RapBattle::goOnStageOrBattle` function doesn't require the challenger to transfer his/her rapper tokenId, nor his/her credibility token bet to the RapBattle contract. Thus, if the defender wins, the transaction reverts as the `RapBattle::_battle` function fails to transfer credibility token bet from challenger to the defender (the challenger never approved the credibility token bet to the contract).
3. Additionally, it is also possible to enter a rap battle as a challenger using a non-existent rapper tokenId. However, attackers will prefer the second option (the option above).

## Proof of Concept

1. Defender enters the rap battle by approving his/her rapper tokenId and credibility token bet to the RapBattle contract. The RapBattle contract transfers the tokenId and credibility token bet to itself.
2. The challenger (attacker) without his/her own rapper tokenId enters the battle with someone else's tokenId (with maxed out stats). Also, the challenger doesn't approve any credibility tokens to the RapBattle contract.
3. If the defender wins, the transaction reverts because the contract failed to transfer tokens from challenger to the defender.
4. If the challenger wins, he/she gets the credibility token bet placed by the defender.

Add the following function to the test contract in your Foundry test file:

```javascript
    function testAttackerEntersWithSomeoneElsesTokenId(uint256 time, bytes32 difficulty) public {
        // setup phase
        address defender = makeAddr("defender");
        address attacker = makeAddr("attacker");
        address pro = makeAddr("pro");

        // defender gets his rapper tokenId minted and stakes it for a day
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.warp(1 days + 1);
        streets.unstake(0);
        vm.stopPrank();

        // pro gets his rapper tokenId minted, stakes it for as long as possible to get max stats and credibility tokens
        vm.startPrank(pro);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 1);
        streets.stake(1);
        vm.warp(5 days + 1);
        streets.unstake(1);
        vm.stopPrank();

        // defender enters battle
        vm.startPrank(defender);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(0, 1);
        vm.stopPrank();

        // attacker enters with pro's rapper tokenId
        vm.startPrank(attacker);
        // fuzz with randomized time and block difficulty
        vm.warp(time);
        vm.prevrandao(difficulty);
        rapBattle.goOnStageOrBattle(1, 1); // didn't approve any credibility tokens to the contract
        assertEq(cred.balanceOf(attacker), 1); // attacker wins defender's credibility token bet
    }
```

The fuzz test ran with randomized time and block difficulty for 272 times with seed "0x1", indicating that the attacker was able to successfully win those battles. The transaction reverted when the defender won.

```shell
Failing tests:
Encountered 1 failing test in test/auditTests.t.sol:RapBattleTest
[FAIL. Reason: ERC20InsufficientAllowance(0x5991A2dF15A8F6A256D3Ec51E99254Cd3fb576A9, 0, 1); counterexample: calldata=0x0b56544000009fc67339fad76937597b36f50df9353a59f65aead188f5d40fda28d2b9b7975c3f3c511c1a03e2a02134ad34521063ac199d0b2430b873191f1e534e2984 args=[1102727873344863962616923325797002069849338298097995261905762206996412855 [1.102e72], 0x975c3f3c511c1a03e2a02134ad34521063ac199d0b2430b873191f1e534e2984]] testAttackerEntersWithSomeoneElsesTokenId(uint256,bytes32) (runs: 272, μ: 616430, ~: 616430)
```

## Impact

Attackers who do not assume any role in the protocol (they don't have a rapper tokenId minted to them) can enter rap battles and win defender's credibility token bet. Also, anyone can use anyone's rapper tokenId for battle (if they enter as challenegrs). This is a serious disruption of what the protocol intends to achieve. Only users with rapper tokenIds should be able to go and battle.

## Tools Used

Foundry, VSCodium.

## Recommendations

In `RapBattle::goOnStageOrBattle` function, check if the rapper tokenId is owned by the msg.sender.

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
+       require(oneShotNft.ownerOf(_tokenId) == msg.sender, "Not owner of rapper tokenId");
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```


# Low Risk Findings

## <a id='L-01'></a>L-01. The battlesWon property of Nft metadata of the winning rapper in a rap battle is never updated            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L54

## Summary

The battlesWon property of Nft metadata of the winning rapper in a rap battle is never updated.

## Vulnerability Details

After winning a rap battle, it is expected that the rapper NFT's battlesWon metadata property is incremented by 1. However, this detail was missed in the protocol code.

## Proof of Concept

Put the following function in the test contract of your Foundry test file:

```javascript
    function testBattlesWonPropertyIsNeverIncremented() public {
        // setup phase
        address defender = makeAddr("defender");
        address challenger = makeAddr("challenger");

        // both the challenger and the defender get a rapper Nft minted to them, and stake it to gain some credibility tokens 
        vm.startPrank(defender);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.stopPrank();

        vm.startPrank(challenger);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 1);
        streets.stake(1);
        vm.stopPrank();

        vm.warp(4 days + 1);

        vm.prank(defender);
        streets.unstake(0);
        vm.prank(challenger);
        streets.unstake(1);

        // defender enters the rap battle
        vm.startPrank(defender);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(0, 1);
        vm.stopPrank();

        // challenger enters the rap battle
        vm.startPrank(challenger);
        oneShot.approve(address(rapBattle), 1);
        cred.approve(address(rapBattle), 1);
        rapBattle.goOnStageOrBattle(1, 1);
        vm.stopPrank();

        uint256 defenderBattleswon = oneShot.getRapperStats(0).battlesWon;
        uint256 challengerBattleswon = oneShot.getRapperStats(1).battlesWon;
        console.log(defenderBattleswon, challengerBattleswon);
        assertEq(defenderBattleswon, 0); // not updated
        assertEq(challengerBattleswon, 0); // not updated
    }
```

The test passes:

```shell
[PASS] testBattlesWonPropertyIsNeverIncremented() (gas: 638762)
Logs:
  0 0
```

## Impact

Rapper tokenIds with higher number of battles won should be more collectible and reputable. It also helps users keep track of the number of battles they have won. Invalid/stale Nft metadata is not desirable as well.

## Tools Used

Foundry, VSCodium.

## Recommendations

In OneShot contract, first allow the RapBattle contract to update the Nft metadata. Then, in `RapBattle::_battle` function, update the battlesWon property of the winner by calling `OneShot::updateRapperStats` function.

## <a id='L-02'></a>L-02. The `RapBattle::Battle` event is triggered with wrong data (wrong winner) if the random value is equal to the defenderRapperSkill            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L67

## Summary

The `RapBattle::Battle` event is triggered with wrong data (wrong winner) if the random value is equal to the defenderRapperSkill. The event is emitted with it's third field indicating the winner to be the challenger, however, the contract picks the defender as the winner.

## Vulnerability Details

The `RapBattle::_battle` function emits the `RapBattle::Battle` event with it's third field calculated using the less than relational operator (<).

```javascript
67.    random < defenderRapperSkill ? _defender : msg.sender
```

However, in the next few lines, we see the usage of less than or equal to relational operator (<=) to pick the winner:

```javascript
70.    if (random <= defenderRapperSkill)
```

Thus, if the random value evaluates to be equal to defenderRapperSkill, the data in the event emitted and the actual winner picked will be different. The winner indicated by the event will be the challenger, but the winner picked by the contract will be the defender.

## Impact

Events capture important information, allowing external observers to react to them and obtain relevant data. Dapps use events to display and update data in their UIs. An event emitted with wrong data may confuse or mislead people. 

## Tools Used

Foundry, VSCodium.

## Recommendations

Make the following changes in the `RapBattle::_battle` function:

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;
        oneShot.approve(address(rapBattle), 1);
        cred.approve(address(rapBattle), 1);

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
-       emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
+       emit Battle(msg.sender, _tokenId, random <= defenderRapperSkill ? _defender : msg.sender);        

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```