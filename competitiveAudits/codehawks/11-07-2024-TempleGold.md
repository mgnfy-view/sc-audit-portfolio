# TempleGold - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Smart contracts (and account abstraction wallets) can neither perform cross-chain TGLD transfers nor participate in cross-chain auctions](#H-01)
- ## Low Risk Findings
    - ### [L-01. Auction tokens cannot be recovered for the first ever spice auction](#L-01)
    - ### [L-02. TempleGold tokens cannot be recovered when a `DaiGoldAuction` ends with 0 bids](#L-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: TempleDAO

### Dates: Jul 4th, 2024 - Jul 11th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-templegold)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 2

# High Risk Findings

## <a id='H-01'></a>H-01. Smart contracts (and account abstraction wallets) can neither perform cross-chain TGLD transfers nor participate in cross-chain auctions

## Summary

Users often like to simplify their tasks, whether it be automating their interactions with a protocol via smart contracts or making use of account abstraction (AA) wallets (essentially just smart contracts). The same smart contracts deployed on different chains along with AA enabled wallets may produce different addresses. In such cases, the user would not be able to perform cross-chain TGLD transfers or participate in cross-chain auctions.

## Vulnerability Details

According to the docs,

> ### Temple Gold
>
> Temple Gold (TGLD) is a non-tradable non-transferrable cross-chain ERC20 token.
> -> A TGLD holder can only transfer cross-chain to their own account address <-
> TGLD can be transferred to whitelisted addresses. These are TempleGoldStaking, DaiGoldAuction, SpiceAuction and team gnosis multisig address
> TGLD uses layer zero for cross-chain functionality.

Smart Contracts deployed on different chains can yield different addresses, for example the [zkSync docs](https://docs.zksync.io/build/resources/faq#can-someone-claim-the-address-i-have-for-my-contract-in-other-evm-networks-in-zksync-era) state,

> The contract address derivation formula is different from the regular EVM approach. Even if a contract is deployed from the same account address with the same nonce, the ZKsync Era contract address will not be the same as it is in another EVM network.

This prevents Smart Contract Accounts and AA wallets from performing cross-chain transfers of TGLD tokens because of the following implementation of the `TempleGold::send()` function,

```js
function send(
    SendParam calldata _sendParam,
    MessagingFee calldata _fee,
    address _refundAddress
) external payable virtual override(IOFT, OFTCore) returns (MessagingReceipt memory msgReceipt, OFTReceipt memory oftReceipt) {
    // code

    /// cast bytes32 to address
    address _to = _sendParam.to.bytes32ToAddress();
    /// @dev user can cross-chain transfer to self
@>  if (msg.sender != _to) { revert ITempleGold.NonTransferrable(msg.sender, _to); }

    // code
}
```

## Impact

Imagine Alice, with an AA wallet or a smart contract module:

1. Bids DAI in `DaiGoldAuction` and receives TGLD on the source Arbitrum chain.
2. Decides to participate in a `SpiceAuction` on Polygon.
3. Tries to `send` the tokens over but is met with `NonTransferrable` error.
4. Unable to participate in the auction.

As a result, a large populus of users like Alice, would not be able to participate in the Temple ecosystem.

## Recommendations

Encourage users to use personal EOAs instead of contract accounts/AA wallets as updating the token contract logic to allow transfers to other addresses would break the core invariant.

## Tools Used

Manual Review.

# Low Risk Findings

## <a id='L-01'></a>L-01. Auction tokens cannot be recovered for the first ever spice auction

## Summary

The `SpiceAuction::recoverToken()` function is used to recover the auction tokens (either temple gold or a spice token) for an auction whose config is set, but hasn't started yet. However, for auction id 1 (and epoch id 0, since auction hasn't started yet), tokens can never be recovered if the dao wanted to do so.

> **NOTE**
> Auction id isn't a state variable. It is used to better represent the latest auction number. Auction config is set for the `_currentEpochId` + 1 index, and the `_currentEpochId` variable lags behind the auction id by 1 until the auction starts. Then the `_currentEpochId` is incremented, and epoch info is set.

## Vulnerability Details

Consider the following code segment from `SpiceAuction::recoverToken()` function,

```js
    function recoverToken(address token, address to, uint256 amount) external override onlyDAOExecutor {
        // ...

        uint256 epochId = _currentEpochId;
@>      EpochInfo storage info = epochs[epochId];
        /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
@>      if (info.startTime == 0) revert InvalidConfigOperation();
        if (!info.hasEnded() && auctionConfigs[epochId + 1].duration == 0) revert RemoveAuctionConfig();

        // ...
    }
```

For the first spice auction which hasn't started yet (auction id 1 and epoch id 0), the epoch info hasn't been set (it is set once the `SpiceAuction::startAuction()` function is called). Thus, [L#249](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L249) accesses a non-existent epoch info (all fields in the struct hold value 0) and the function reverts at [L#251](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L251).

Additionally, auction tokens cannot be recovered during the cooldown period. This will revert at [L#252](https://github.com/Cyfrin/2024-07-templegold/blob/57a3e597e9199f9e9e0c26aab2123332eb19cc28/protocol/contracts/templegold/SpiceAuction.sol#L252).

## Impact

The dao executor won't be able to recover tokens for the first ever spice auction. This is essentially a DOS type of vulnerability.

## Proof Of Concept

Add the following test to `SpiceAuctionTest` contract in `./protocol/test/forge/templegold/SpiceAuction.t.sol`,

```js
    function testRecoverTokenFailsForFirstAuction() public {
        ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
        vm.startPrank(daoExecutor);
        spice.setAuctionConfig(config);
        vm.stopPrank();

        address auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
        dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

        vm.prank(daoExecutor);
        vm.expectRevert(ISpiceAuction.InvalidConfigOperation.selector);
        spice.recoverToken(auctionToken, daoExecutor, 100 ether);
    }
```

The test passes, with the following logs,

```shell
Ran 1 test for test/SpiceAuction.t.sol:SpiceAuctionTest
[PASS] testRecoverTokenFailsForFirstAuction() (gas: 396290)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.06s (2.92ms CPU time)

Ran 1 test suite in 2.08s (2.06s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Tools Used

Manual review and Foundry for writing POC and tests.

## Recommended Mitigation

Make the following changes to `SpiceAuction::recoverToken()` functions,

```diff
    function recoverToken(address token, address to, uint256 amount) external override onlyDAOExecutor {
        // ...

        uint256 epochId = _currentEpochId;
        EpochInfo storage info = epochs[epochId];
        /// @dev use `removeAuctionConfig` for case where `auctionStart` is called and cooldown is still pending
-       if (info.startTime == 0) { revert InvalidConfigOperation(); }
-       if (!info.hasEnded() && auctionConfigs[epochId+1].duration == 0) { revert RemoveAuctionConfig(); }
+       if (epochId != 0) {
+           if (info.startTime == 0) revert InvalidConfigOperation();
+           if (!info.hasEnded() && auctionConfigs[epochId + 1].duration == 0) revert RemoveAuctionConfig();
+       }

        // ...
    }
```

Now let's unit test the changes,

```js
    function testRecoverTokenForFirstAuction() public {
        ISpiceAuction.SpiceAuctionConfig memory config = _getAuctionConfig();
        vm.startPrank(daoExecutor);
        spice.setAuctionConfig(config);
        vm.stopPrank();

        address auctionToken = config.isTempleGoldAuctionToken ? address(templeGold) : spice.spiceToken();
        dealAdditional(IERC20(auctionToken), address(spice), 100 ether);

        vm.prank(daoExecutor);
        spice.recoverToken(auctionToken, daoExecutor, 100 ether);

        assertEq(IERC20(auctionToken).balanceOf(daoExecutor), 100 ether);
    }
```

The test passes with the following logs,

```shell
Ran 1 test for test/SpiceAuction.t.sol:SpiceAuctionTest
[PASS] testRecoverTokenForFirstAuction() (gas: 424640)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.59s (3.46ms CPU time)

Ran 1 test suite in 1.61s (1.59s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## <a id='L-02'></a>L-02. TempleGold tokens cannot be recovered when a `DaiGoldAuction` ends with 0 bids

## Summary

When an auction ends with 0 bids, the `totalAuctionTokenAmount` value of TGLD tokens remain locked forever as `DaiGoldAuction::recoverToken()` function can only be used when the auction is still in its cooldown period.

## Vulnerability Details

`DaiGoldAuction::recoverToken` provides a mechanism to recover tokens that have been distributed and allotted to a specific auction only **BEFORE** the auction starts, that is during the cooldown period as evident by the code below,

```js
function recoverToken(
    address token,
    address to,
    uint256 amount
) external override onlyElevatedAccess {
    // code

    // auction started but cooldown pending
    uint256 epochId = _currentEpochId;
    EpochInfo storage info = epochs[epochId];
    if (info.startTime == 0) { revert InvalidOperation(); }
@>  if (info.isActive()) { revert AuctionActive(); }
@>  if (info.hasEnded()) { revert AuctionEnded(); }

    // code
}
```

When users bid in an auction, they are entitled to their claims, even if there's just one bid, that bidder deserves the entire pot. Users are also allowed to claim tokens from a previous auction as they please, hence we need not worry about tokens remaining stuck in the contract.

However this is not the case when an auction ends with no bids, as in such a situation we would want to recover those tokens as there are no claims. This is prevented by the fact that tokens cannot be recovered after an auction ends.

## Impact

`totalAuctionTokenAmount` value worth of tokens allotted for the auction would remain permanently stuck inside the contract. If this scenario repeats multiple times, a significant amount of tokens would be cut out from circulation.

## Proof of Code

1. Add the following test to the existing `DaiGoldAuction.t.sol` file,

```js
function test_CannotRecover_AuctionTokens_WhenZeroBids() public{
    // set the config and startAuction
    IDaiGoldAuction.AuctionConfig memory config = _getAuctionConfig();
    vm.startPrank(executor);
    daiGoldAuction.setAuctionConfig(config);
    skip(1 days); // give enough time for TGLD emissions
    daiGoldAuction.startAuction();

    /*
        THE AUCTION ENDS WITHOUT ANY BIDS
    */

    uint256 currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info1 = daiGoldAuction.getEpochInfo(currentEpoch);
    vm.warp(info1.endTime);
    uint256 recoveryAmount1 = info1.totalAuctionTokenAmount;

    // unable to recover tokens
    vm.expectRevert(abi.encodeWithSelector(IAuctionBase.AuctionEnded.selector));
    daiGoldAuction.recoverToken(address(templeGold), alice, recoveryAmount1);

    // tokens not included in the next auction either
    assertEq(daiGoldAuction.nextAuctionGoldAmount(), 0);

    // next auction starts and is in cooldown period
    vm.warp(info1.endTime + config.auctionsTimeDiff);
    daiGoldAuction.startAuction();
    currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info2 = daiGoldAuction.getEpochInfo(currentEpoch);
    uint256 recoveryAmount2 = info2.totalAuctionTokenAmount;

    // let's try to recover tokens from previous auction along with current auction
    uint256 totalRecoveryAmount = recoveryAmount1 + recoveryAmount2;
    vm.expectRevert(abi.encodeWithSelector(CommonEventsAndErrors.InvalidAmount.selector, address(templeGold), totalRecoveryAmount));
    daiGoldAuction.recoverToken(address(templeGold), alice, totalRecoveryAmount);

    // assuming that distribution hasn't happened in the meanwhile
    assertEq(templeGold.balanceOf(address(daiGoldAuction)), totalRecoveryAmount);
}
```

1. Also update the vesting period params in `DaiGoldAuction.t.sol::_configureTempleGold` as follows,

```diff
function _configureTempleGold() private {
    ITempleGold.DistributionParams memory params;
    params.escrow = 60 ether;
    params.gnosis = 10 ether;
    params.staking = 30 ether;
    templeGold.setDistributionParams(params);
    ITempleGold.VestingFactor memory factor;
-   factor.numerator = 2 ether;
+   factor.numerator = 1;
-   factor.denominator = 1000 ether;
+   factor.denominator = 162 weeks; // 3 years
    templeGold.setVestingFactor(factor);
    templeGold.setStaking(address(goldStaking));
    // whitelist
    templeGold.authorizeContract(address(daiGoldAuction), true);
    templeGold.authorizeContract(address(goldStaking), true);
    templeGold.authorizeContract(teamGnosis, true);
}
```

1. Run `forge test --mt test_CannotRecover_AuctionTokens_WhenZeroBids -vv`

The test passes successfully with the following logs,

```shell
Ran 1 test for test/forge/templegold/DaiGoldAuction.t.sol:DaiGoldAuctionTest
[PASS] test_CannotRecover_AuctionTokens_WhenZeroBids() (gas: 405414)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 475.59ms (2.81ms CPU time)

Ran 1 test suite in 483.13ms (475.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommendations

Preferable mitigation would be to add another function to `DaiGoldAuction` for specifically recovering tokens when an auction ends with 0 bids as shown below:

```diff
+ /**
+  * @notice Recover auction tokens for epoch with zero bids
+  * @param epochId Epoch Id
+  * @param to Recipient
+  */
+ function recoverAuctionTokenForZeroBidAuction(uint256 epochId, address to) external onlyElevatedAccess {
+     if (to == address(0)) { revert CommonEventsAndErrors.InvalidAddress(); }
+     // has to be valid epoch
+     if (epochId > _currentEpochId) { revert InvalidEpoch(); }
+     // epoch has to be ended
+     EpochInfo storage info = epochs[epochId];
+     if (!info.hasEnded()) { revert AuctionActive(); }
+     // bid token amount for epoch has to be 0
+     if (info.totalBidTokenAmount > 0) { revert InvalidOperation(); }

+     uint256 amount = info.totalAuctionTokenAmount;
+     emit CommonEventsAndErrors.TokenRecovered(to, address(templeGold), amount);
+     templeGold.safeTransfer(to, amount);
+ }
```

Above change can be verified against the following uint test,

```js
function test_MitigationSuccessful() public {
    // set the config and startAuction
    IDaiGoldAuction.AuctionConfig memory config = _getAuctionConfig();
    vm.startPrank(executor);
    daiGoldAuction.setAuctionConfig(config);
    skip(1 days); // give enough time for TGLD emissions
    daiGoldAuction.startAuction();

    /*
        THE AUCTION ENDS WITHOUT ANY BIDS
    */

    uint256 currentEpoch = daiGoldAuction.currentEpoch();
    IAuctionBase.EpochInfo memory info = daiGoldAuction.getEpochInfo(currentEpoch);
    vm.warp(info.endTime);
    uint256 recoveryAmount = info.totalAuctionTokenAmount;

    vm.expectEmit(address(daiGoldAuction));
    emit TokenRecovered(alice, address(templeGold), recoveryAmount);
    daiGoldAuction.recoverAuctionTokenForZeroBidAuction(currentEpoch, alice);

    // assuming alice has no prior token balance
    assertEq(templeGold.balanceOf(alice), recoveryAmount);
}
```

The test passes successfully with the following logs,

```shell
Ran 1 test for test/forge/templegold/DaiGoldAuction.t.sol:DaiGoldAuctionTest
[PASS] test_MitigationSuccessful() (gas: 329745)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 455.58ms (3.28ms CPU time)

Ran 1 test suite in 458.30ms (455.58ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

Another way would be to update `recoverToken` to also account for 0 bid auctions but it is better to have another function for the sake of Separation of Concerns.

## Tools Used

Manual Review and Foundry for POC.