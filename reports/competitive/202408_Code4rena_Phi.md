# Phi - Findings Report

## Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [[H-01] Uncapped `mintFee` allows frontrunning attacks on large buy orders](#H-01)
- Medium Risk Findings
  - [[M-01] Refund sent to contract instead of user in `PhiFactory::_processClaim`](#M-01)

<br>

## <a id='contest-summary'></a>Contest Summary

### Sponsor: Phi Protocol

### Dates: Jul 23rd, 2024 - Jul 29th, 2024

Phi Protocol is an open credentialing protocol to help users form, visualize, showcase their onchain identity. It
incentivizes individuals to index blockchain transaction data as onchain credential blocks, curate them, host the
verification process, and mint onchain credential contents.

[See more contest details on Code4rena.](https://code4rena.com/audits/2024-08-phi)

## <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a> [H-01] Uncapped `mintFee` allows frontrunning attacks on large buy orders

## Summary

`PhiFactory::updateArtSettings` allows the art creator to update the `mintFee` without any restrictions or cooldown
period. This creates a vulnerability where the creator can frontrun large buy orders by suddenly increasing the mint
fee, potentially stealing from buyers.

## Impact

A malicious art creator could monitor the mempool for incoming large buy orders. Just before such an order is processed,
they could call `updateArtSettings` to dramatically increase the `mintFee`. This would result in the buyer paying much
more than they intended, with the excess fees going to the creator.

## Proof of Concept

[PhiFactory.sol#L248](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L248)

```javascript
function updateArtSettings(
    uint256 artId_,
    string memory url_,
    address receiver_,
    uint256 maxSupply_,
    uint256 mintFee_,
    uint256 startTime_,
    uint256 endTime_,
    bool soulBounded_,
    IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
)
    external
    onlyArtCreator(artId_)
{
    if (receiver_ == address(0)) {
        revert InvalidAddressZero();
    }

    if (endTime_ < startTime_) {
        revert InvalidTimeRange();
    }
    if (endTime_ < block.timestamp) {
        revert EndTimeInPast();
    }

    PhiArt storage art = arts[artId_];

    if (art.numberMinted > maxSupply_) {
        revert ExceedMaxSupply();
    }

    art.receiver = receiver_;
    art.maxSupply = maxSupply_;
@>  art.mintFee = mintFee_;

    // existing code...
}
```

An attacker could exploit this by:

1. Creating art with a low initial `mintFee`
2. Monitoring the mempool for large buy orders
3. Quickly calling `updateArtSettings` to increase the `mintFee` just before the buy order is processed
4. The buyer's transaction would then execute with the new, higher fee

## Tools Used

Manual review

## Recommended Mitigation Steps

Implement a time delay for mint fee changes to take effect. This gives users time to react to fee updates and prevents
sudden, unexpected fee increases.

Here's a suggested implementation:

```diff
contract PhiFactory is IPhiFactory, Ownable2StepUpgradeable, PausableUpgradeable, UUPSUpgradeable {
+    uint256 public constant FEE_UPDATE_DELAY = 1 days;
+    mapping(uint256 => PendingFeeUpdate) public pendingFeeUpdates;
+
+    struct PendingFeeUpdate {
+        uint256 newFee;
+        uint256 effectiveTime;
+    }
+

    // existing code...

    function updateArtSettings(
        uint256 artId_,
        string memory url_,
        address receiver_,
        uint256 maxSupply_,
        uint256 mintFee_,
        uint256 startTime_,
        uint256 endTime_,
        bool soulBounded_,
        IPhiNFT1155Ownable.RoyaltyConfiguration memory configuration
    )
        external
        onlyArtCreator(artId_)
    {
        // ... existing checks ...

        PhiArt storage art = arts[artId_];

        // ... other updates ...

-       art.mintFee = mintFee_;
+       pendingFeeUpdates[artId_] = PendingFeeUpdate({
+           newFee: mintFee_,
+           effectiveTime: block.timestamp + FEE_UPDATE_DELAY
+       });
+       emit PendingFeeUpdateCreated(artId_, mintFee_, block.timestamp + FEE_UPDATE_DELAY);

        // ... rest of the function ...
    }

+   function applyPendingFeeUpdate(uint256 artId_) external {
+       PendingFeeUpdate memory update = pendingFeeUpdates[artId_];
+       require(update.effectiveTime > 0 && block.timestamp >= update.effectiveTime, "Fee update not ready");
+
+       arts[artId_].mintFee = update.newFee;
+       delete pendingFeeUpdates[artId_];
+       emit FeeUpdateApplied(artId_, update.newFee);
+   }
}
```

---

## <a id='M-01'></a> [M-01] Refund sent to contract instead of user in `PhiFactory::_processClaim`

## Summary

In the `_processClaim` function of PhiFactory, excess ETH is refunded to `_msgSender()`, which is the contract itself
when called internally, instead of the original transaction sender. This results in users losing any overpaid funds
during the claim process.

## Impact

Users who overpay when claiming art (i.e., send more ETH than required for the mint fee) will have their excess funds
trapped in the contract instead of being refunded. This leads to a direct loss of funds for users and may discourage
participation in the protocol, but limited to the amount of overpayment per transaction.

## Proof of Concept

Here's the call stack for this issue:

1. External call to `this.merkleClaim(...)` (same issue stands for `this.signatureClaim(...)`), which changes the
   `msg.sender` ([PhiFactory.sol#L283](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L283)):

```javascript
// PhiFactory.sol
function claim(bytes calldata encodeData_) external payable {
    bytes memory encodedata_ = LibZip.cdDecompress(encodeData_);
    (uint256 artId) = abi.decode(encodedata_, (uint256));
    PhiArt storage art = arts[artId];
    uint256 tokenId = IPhiNFT1155Ownable(art.artAddress).getTokenIdFromFactoryArtId(artId);

    if (art.verificationType.eq("MERKLE")) {
        // existing code...

@>      this.merkleClaim{ value: mintFee }(proof, claimData, mintArgs, leafPart_);
    } else if (art.verificationType.eq("SIGNATURE")) {
        // existing code...

@>      this.signatureClaim{ value: mintFee }(signature_, claimData, mintArgs);
    } else {
        revert InvalidVerificationType();
    }
}
```

2. Internal call to `_processClaim`
   ([PhiFactory.sol#L378-L380](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L378-L380)):

```javascript
// PhiFactory.sol
function merkleClaim(
    bytes32[] calldata proof_,
    bytes calldata encodeData_,
    MintArgs calldata mintArgs_,
    bytes32 leafPart_
)
    external
    payable
    whenNotPaused
{
    // existing code...

@>  _processClaim(
        artId_, minter_, ref_, art.credCreator, mintArgs_.quantity, leafPart_, mintArgs_.imageURI, msg.value
    );
}
```

3. Handling of refunds in `_processClaim` takes current `_msgSender()` into account, which is the contract itself
   ([PhiFactory.sol#L740](https://github.com/code-423n4/2024-08-phi/blob/main/src/PhiFactory.sol#L740)):

```javascript
// PhiFactory.sol
function _processClaim(
    uint256 artId_,
    address minter_,
    address ref_,
    address verifier_,
    uint256 quantity_,
    bytes32 data_,
    string memory imageURI_,
    uint256 etherValue_
)
    private
{
    PhiArt storage art = arts[artId_];

    // Handle refund
    uint256 mintFee = getArtMintFee(artId_, quantity_);
    if ((etherValue_ - mintFee) > 0) {
@>      _msgSender().safeTransferETH(etherValue_ - mintFee);
    }

    // existing code...
}
```

Additionally, this can be checked with the following contract via Remix:

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Context.sol";

contract MsgSenderExternal is Context {
    function getMsgSenderNormal() external view returns (address, address, address) {
        return (msg.sender, tx.origin, _msgSender());
    }

    function getExternalSender() external view returns(address, address, address) {
        return this.getMsgSenderNormal();
    }
}
```

- Calling `getMsgSenderNormal()` will return three identical addresses
- Calling `getExternalSender()` will return identical `msg.sender` and `_msgSender()`, but `tx.origin` (the EOA that
  initiated the call) will be different

This further proves that refund will not go to the EOA calling the `claim` function, but to contract calling
`this.merkleClaim(...)`, because a new external call changes the original `msg.sender`.

## Tools Used

Manual review

## Recommended Mitigation Steps

To fix this issue, pass the original caller's address to `_processClaim` and use it for the refund. Here's a suggested
fix:

```diff
diff --git a/src/PhiFactory.sol b/src/PhiFactory.sol
index a1b2c3d..e4f5g6h 100644
--- a/src/PhiFactory.sol
+++ b/src/PhiFactory.sol
@@ -378,7 +378,7 @@ contract PhiFactory is IPhiFactory, Ownable2StepUpgradeable, PausableUpgradeable
     {
         _validateAndUpdateClaimState(artId_, minter_, mintArgs_.quantity);
         _processClaim(
-            artId_, minter_, ref_, art.credCreator, mintArgs_.quantity, leafPart_, mintArgs_.imageURI, msg.value
+            artId_, minter_, ref_, art.credCreator, mintArgs_.quantity, leafPart_, mintArgs_.imageURI, msg.value, _msgSender()
         );

         emit ArtClaimedData(artId_, "MERKLE", minter_, ref_, art.credCreator, arts[artId_].artAddress, mintArgs_.quantity);
@@ -340,7 +340,7 @@ contract PhiFactory is IPhiFactory, Ownable2StepUpgradeable, PausableUpgradeable
         }

         _validateAndUpdateClaimState(artId_, minter_, mintArgs_.quantity);
-        _processClaim(artId_, minter_, ref_, verifier_, mintArgs_.quantity, data_, mintArgs_.imageURI, msg.value);
+        _processClaim(artId_, minter_, ref_, verifier_, mintArgs_.quantity, data_, mintArgs_.imageURI, msg.value, _msgSender());

         emit ArtClaimedData(artId_, "SIGNATURE", minter_, ref_, verifier_, arts[artId_].artAddress, mintArgs_.quantity);
     }
@@ -818,7 +818,8 @@ contract PhiFactory is IPhiFactory, Ownable2StepUpgradeable, PausableUpgradeable
         uint256 quantity_,
         bytes32 data_,
         string memory imageURI_,
-        uint256 etherValue_
+        uint256 etherValue_,
+        address refundRecipient_
     )
         private
     {
@@ -827,7 +828,7 @@ contract PhiFactory is IPhiFactory, Ownable2StepUpgradeable, PausableUpgradeable
         // Handle refund
         uint256 mintFee = getArtMintFee(artId_, quantity_);
         if ((etherValue_ - mintFee) > 0) {
-            _msgSender().safeTransferETH(etherValue_ - mintFee);
+            refundRecipient_.safeTransferETH(etherValue_ - mintFee);
         }

         // ... rest of the function ...
```
