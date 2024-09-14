# Fjord Token Staking - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)

- Medium Risk Findings
  - [[M-01] Incorrect stake reduction in `FjordPoints` on Sablier stream cancellation](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Fjord

### Dates: Aug 20th, 2024 - Aug 27th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-fjord)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 0
- Medium: 1
- Low: 0

# Medium Risk Findings

## <a id='M-01'></a>[M-01] Incorrect stake reduction in `FjordPoints` on Sablier stream cancellation

## Summary

When a Sablier stream is cancelled, the `onStreamCanceled` function in `FjordStaking` calls `_unstakeVested`, which reduces the stake in the `FjordPoints` contract. However, it uses `msg.sender` (the Sablier contract) instead of the actual stream owner, leading to incorrect stake accounting and potential issues in the points system.

## Impact

- The wrong address (Sablier contract) will have its stake reduced in the `FjordPoints` contract, while the actual user's stake remains unchanged.
- However, since the Sablier contract address doesn't exist in the `FjordPoints` contract's `users` mapping, the `onUnstaked` function might be DoS'd, preventing stream cancellations.

## Proof of Concept

Here's a full flow of the Sablier stream cancellation:

[FjordStaking.sol#L840](https://github.com/Cyfrin/2024-08-fjord/blob/main/src/FjordStaking.sol#L840)

```javascript
function onStreamCanceled(
    uint256 streamId,
    address sender,
    uint128 senderAmount,
    uint128 /*recipientAmount*/
) external override onlySablier checkEpochRollover {
    address streamOwner = _streamIDOwners[streamId];
    if (streamOwner == address(0)) revert StreamOwnerNotFound();

    _redeem(streamOwner);

    NFTData memory nftData = _streamIDs[streamOwner][streamId];

    uint256 amount =
        uint256(senderAmount) > nftData.amount ? nftData.amount : uint256(senderAmount);

@>  _unstakeVested(streamOwner, streamId, amount);
    emit SablierCanceled(streamOwner, streamId, sender, amount);
}
```

[FjordStaking.sol#L561](https://github.com/Cyfrin/2024-08-fjord/blob/main/src/FjordStaking.sol#L561)

```javascript
function _unstakeVested(address streamOwner, uint256 _streamID, uint256 amount) internal {
    // existing code...

    //INTERACT
    // 6) Return Sablier NFT
    if (isFullUnstaked) {
        sablier.transferFrom({ from: address(this), to: streamOwner, tokenId: _streamID });
    }

    // 7) Reduce user's stake in points contract
@>  points.onUnstaked(msg.sender, amount); // <---- msg.sender is actually Sablier

    emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
}
```

[FjordPoints.sol#L220](https://github.com/Cyfrin/2024-08-fjord/blob/main/src/FjordPoints.sol#L220)

```javascript
function onUnstaked(address user, uint256 amount)
    external
    onlyStaking
    checkDistribution
    updatePendingPoints(user)
{
@>  UserInfo storage userInfo = users[user]; // <----- might even DoS because Sablier's sender doesn't exist here
    if (amount > userInfo.stakedAmount) {
        revert UnstakingAmountExceedsStakedAmount();
    }
    userInfo.stakedAmount = userInfo.stakedAmount.sub(amount);
    totalStaked = totalStaked.sub(amount);
    emit Unstaked(user, amount);
}
```

## Tools Used

Manual code review

## Recommended Mitigation Steps

Modify the `FjordStaking::_unstakeVested` function to accept the stream owner's address as a parameter, and pass this address to the `points.onUnstaked` call.

```diff
function _unstakeVested(address streamOwner, uint256 _streamID, uint256 amount) internal {
    // existing code...

    //INTERACT
    // 6) Return Sablier NFT
    if (isFullUnstaked) {
        sablier.transferFrom({ from: address(this), to: streamOwner, tokenId: _streamID });
    }

    // 7) Reduce user's stake in points contract
-   points.onUnstaked(msg.sender, amount);
+   points.onUnstaked(streamOwner, amount);

    emit VestedUnstaked(streamOwner, epoch, amount, _streamID);
}
```
