# First Flight #14: AirDropper - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Eligible users can claim their airdrop amounts over and over again, draining the contract](#H-01)
    - ### [H-02. Account abstraction will lead to some users not being able to claim their airdrop amounts](#H-02)
    - ### [H-03. Incorrect zkSync Era USDC address leads to tokens being stuck in the airdrop contract](#H-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #14

### Dates: Apr 25th, 2024 - May 2nd, 2024

[See more contest details here](https://www.codehawks.com/contests/clvb821kr0001jzdbi6ggixb0)

# <a id='results-summary'></a>Results Summary

Stood 5th out of a total of 54 participants.

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Eligible users can claim their airdrop amounts over and over again, draining the contract            

## Description

A user eligible for the airdrop can verify themselves as being part of the merkle tree and claim their airdrop amount. However, there is no mechanism enabled to track the users who have already claimed their airdrop, and the merkle tree is still composed of the same user. This allows users to drain the `MerkleAirdrop` contract by calling the `MerkleAirdrop::claim()` function over and over again.

## Impact

**Severity: High**<br/>**Likelihood: High**

A malicious user can call the `MerkleAirdrop::claim()` function over and over again until the contract is drained of all its funds. This also means that other users won't be able to claim their airdrop amounts.

## Proof of Code

Add the following test to `./test/MerkleAirdrop.t.sol`,

```javascript
    function testClaimAirdropOverAndOverAgain() public {
        vm.deal(collectorOne, airdrop.getFee() * 4);

        for (uint8 i = 0; i < 4; i++) {
            vm.prank(collectorOne);
            airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
        }

        assertEq(token.balanceOf(collectorOne), 100e6);
    }
```

The test passes, and the malicious user has drained the contract of all its funds.

## Tools Used

Manual review and Foundry.

## Recommended Mitigation

Use a mapping to store the addresses that have claimed their airdrop amounts. Check and update this mapping each time a user tries to claim their airdrop amount.

```diff
contract MerkleAirdrop is Ownable {
    using SafeERC20 for IERC20;

    error MerkleAirdrop__InvalidFeeAmount();
    error MerkleAirdrop__InvalidProof();
    error MerkleAirdrop__TransferFailed();
+   error MerkleAirdrop__AlreadyClaimed();

    uint256 private constant FEE = 1e9;
    IERC20 private immutable i_airdropToken;
    bytes32 private immutable i_merkleRoot;
+   mapping(address user => bool claimed) private s_hasClaimed;

    ...

    function claim(address account, uint256 amount, bytes32[] calldata merkleProof) external payable {
+       if (s_hasClaimed[account]) revert MerkleAirdrop__AlreadyClaimed();
        if (msg.value != FEE) {
            revert MerkleAirdrop__InvalidFeeAmount();
        }
        bytes32 leaf = keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
        if (!MerkleProof.verify(merkleProof, i_merkleRoot, leaf)) {
            revert MerkleAirdrop__InvalidProof();
        }
+       s_hasClaimed[account] = true;
        emit Claimed(account, amount);
        i_airdropToken.safeTransfer(account, amount);
    }
```

Now, let's unit test the changes,

```javascript
    function testCannotClaimAirdropMoreThanOnceAnymore() public {
        vm.deal(collectorOne, airdrop.getFee() * 2);

        vm.prank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);

        vm.prank(collectorOne);
        airdrop.claim{ value: airdrop.getFee() }(collectorOne, amountToCollect, proof);
    }
```

The test correctly fails, with the following logs,

```shell
Failing tests:
Encountered 1 failing test in test/MerkleAirdropTest.t.sol:MerkleAirdropTest
[FAIL. Reason: MerkleAirdrop__AlreadyClaimed()] testCannotClaimAirdropMoreThanOnceAnymore() (gas: 96751)
```
## <a id='H-02'></a>H-02. Account abstraction will lead to some users not being able to claim their airdrop amounts            

## Description

Users using wallets with account abstraction will have different addresses on different chains. The merkle root is generated from the account addresses from the Ethereum L1, as stated in the documentation. This will render some users unable to claim their airdrop amounts on the zkSync Era Mainnet.

## Impact

**Severity: High**<br/>**Likelihood: Medium**

Some users will not be able to claim their airdrop amounts from the `MerkleAirdrop` contract. Additionally, the unclaimed tokens will remain stuck in the contract since there is no function available for the owner to withdraw the remaining balance.

## Tools Used

Manual review.

## Recommended Mitigation

Generate the merkle root from the correct zkSync Era Mainnet addresses instead of the Ethereum addresses.
## <a id='H-03'></a>H-03. Incorrect zkSync Era USDC address leads to tokens being stuck in the airdrop contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L8

https://github.com/Cyfrin/2024-04-airdropper/blob/781cf225664a4ad11e9654aaa39cc528016bf214/script/Deploy.s.sol#L18

## Description

The deployment script declares the zkSync Era Mainnet USDC token address as `0x1D17CbCf0D6d143135be902365d2e5E2a16538d4` at line 8. However, the actual token address is `0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4`. The wrong address uses a `b` instead of an `a` as the 21st character. The `MerkleAirdrop` receives the wrong token address, which causes all token transfers to fail when users try to claim their airdrop amounts, essentially locking the funds in the contract.

## Impact

**Severity: High**<br/>**Likelihood: High**

The `MerkleAirdrop` contract receives an invalid zkSync Era USDC address. However, the correct address is used to transfer USDC to the contract at line 18 in the deployment script. This will cause the USDC tokens to be stuck in the contract, as all transfers will fail when users will try to claim their airdrop amounts. The fact that the token pointed to by the `MerkleAirdrop` contract is immutable and cannot be changed by the owner of the contract aggravates the issue.

## Tools Used

Manual review.

## Recommended Mitigation

Use the correct zkSync Era Mainnet USDC address, and use the same address variable throughout the deployment script. If the same variable (with incorrect address) had been used throughout, the deployment script would have failed at line 18 (token tranfer to the `MerkleAirdrop` contract), saving the protocol team from loss of funds.

In the deployment script, make the following changes:

```diff
-   address public s_zkSyncUSDC = 0x1D17CbCf0D6d143135be902365d2e5E2a16538d4;
+   address public s_zkSyncUSDC = 0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4;
    bytes32 public s_merkleRoot = 0xf69aaa25bd4dd10deb2ccd8235266f7cc815f6e9d539e9f4d47cae16e0c36a05;
    // 4 users, 25 USDC each
    uint256 public s_amountToAirdrop = 4 * (25 * 1e6);

    // Deploy the airdropper
    function run() public {
        vm.startBroadcast();
        MerkleAirdrop airdrop = deployMerkleDropper(s_merkleRoot, IERC20(s_zkSyncUSDC));
        // Send USDC -> Merkle Air Dropper
-      IERC20(0x1d17CBcF0D6D143135aE902365D2E5e2A16538D4).transfer(address(airdrop), s_amountToAirdrop);
+      IERC20(s_zkSyncUSDC).transfer(address(airdrop), s_amountToAirdrop);
        vm.stopBroadcast();
    }
```
