# Munchables - Findings Report

## Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [[H-01] Incorrect Schnibble calculation in `_farmPlots` reduces total rewards instead of applying bonus](#H-01)
  - [[H-02] Munchables remain unproductive after `transferToUnoccupiedPlot` due to an uncleared 'dirty' flag, leading to loss of Schnibble generation`](#H-02)
  - [[H-03] Munchable's `plotId` not updated during `transferToUnoccupiedPlot`](#H-03)

<br>

## <a id='contest-summary'></a>Contest Summary

### Sponsor: Munchables

### Dates: Jul 23rd, 2024 - Jul 29th, 2024

A web3 point farming game in which Keepers nurture creatures to help them evolve, deploying strategies to earn them rewards in competition with other players.

[See more contest details on Code4rena.](https://code4rena.com/audits/2024-07-munchables)

## <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 3
- Medium: 0
- Low: 0

<br>

# High Risk Findings

# <a id='H-01'></a>[H-01] Incorrect Schnibble calculation in `_farmPlots` reduces total rewards instead of applying bonus

## Summary

`_farmPlots` incorrectly calculates the total schnibbles earned, reducing the base amount instead of properly applying a bonus. This results in users receiving significantly fewer schnibbles than intended.

## Impact

This bug severely impacts the reward mechanism of the protocol:

- Users receive far fewer schnibbles than they should, potentially by orders of magnitude.
- The intended bonus system is effectively nullified, as bonuses reduce rewards instead of increasing them.
- This affects both renters (who receive fewer schnibbles) and landlords (who receive less tax).

## Proof of Concept

- [LandManager.sol#L283-L286](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L283-L286)

The bug is in the `_farmPlots` function:

```javascript
schnibblesTotal = uint256(
  (int256(schnibblesTotal) + int256(schnibblesTotal) * finalBonus) / 100
);
```

This calculation divides the entire sum by 100, effectively reducing the original `schnibblesTotal` to 1% of its intended value, plus a small bonus.

For example, if `schnibblesTotal` is 1000 and `finalBonus` is 10:

- Current, wrong calculation: `(1000 + (1000 * 10)) / 100 = 11`
- New, corrected calculation: `1000 + (1000 * 10 / 100) = 1100`

The current implementation reduces rewards by approximately 99% in this scenario.

## Tools Used

Manual review

## Recommended Mitigation Steps

Modify the calculation to apply the bonus correctly:

```diff
- schnibblesTotal = uint256(
-   (int256(schnibblesTotal) + int256(schnibblesTotal) * finalBonus) / 100,
- );
+ schnibblesTotal = schnibblesTotal + uint256((int256(schnibblesTotal) * finalBonus) / 100);
```

---

# <a id='H-02'></a>[H-02] Munchables remain unproductive after `transferToUnoccupiedPlot` due to an uncleared 'dirty' flag, leading to loss of Schnibble generation

## Summary

When a Munchable is marked as `dirty` (typically due to becoming invalid when a landlord reduces their locked amount), it stops generating Schnibbles.

`transferToUnoccupiedPlot` fails to clear the `dirty` flag when moving a Munchable to a valid plot. This results in the Munchable remaining unproductive even after being moved to a valid location, causing a loss of Schnibble generation for the user.

## Impact

- Users lose potential Schnibble generation indefinitely, even after moving their Munchable to a valid plot.
- Users may need to unstake and re-stake their Munchables to resolve the issue, incurring unnecessary gas costs.

## Proof of Concept

- [LandManager.sol#L213-L222](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L213-L222)
- [LandManager.sol#L251](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L251)

1. `transferToUnoccupiedPlot` moves Munchables but doesn't clear the `dirty` flag
2. Next time the user calls any of the functions that trigger `_farmPlots`, `dirty` Munchables are skipped:

```javascript
for (uint8 i = 0; i < staked.length; i++) {
    _toiler = toilerState[tokenId];
    if (_toiler.dirty) continue; // Skip if the toiler state is marked as dirty.

    // ... rest of the function
}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Update the `transferToUnoccupiedPlot` to clear the `dirty` flag:

```diff
function transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)
    external
    override
    forceFarmPlots(msg.sender)
    notPaused
{
    // ... existing checks ...

    toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate;
+   toilerState[tokenId].dirty = false;  // Clear the dirty flag
    plotOccupied[_toiler.landlord][oldPlotId] = Plot({occupied: false, tokenId: 0});
    plotOccupied[_toiler.landlord][plotId] = Plot({occupied: true, tokenId: tokenId});

    // ... rest of the function ...
}
```

---

# <a id='H-03'></a>[H-03] Munchable's `plotId` not updated during `transferToUnoccupiedPlot`

## Summary

`transferToUnoccupiedPlot` fails to update the `plotId` in the `toilerState` mapping when moving a Munchable to a new plot. This can lead to the Munchable being considered `dirty` on an invalid plot in future operations, potentially stopping Schnibble generation.

## Impact

When a Munchable is transferred to a new plot, its `plotId` in the `toilerState` mapping is not updated. This discrepancy can cause several issues:

1. The Munchable may be considered to be `dirty` and on an invalid plot in future `_farmPlots` calls, even though it was moved to a valid plot.
2. This can lead to unexpected loss of Schnibble generation if the landlord reduces their locked amount, making the old (non-updated) plot ID invalid.
3. It creates inconsistency between the `plotOccupied` mapping and the `toilerState` mapping.

## Proof of Concept

- [LandManager.sol#L213-L222](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L213-L222)
- [LandManager.sol#L258-L261](https://github.com/code-423n4/2024-07-munchables/blob/main/src/managers/LandManager.sol#L258-L261)

1. In the `transferToUnoccupiedPlot` function, the `plotId` in `toilerState` is not updated:

```javascript
function transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)
    external
    override
    forceFarmPlots(msg.sender)
    notPaused
{
    // ... existing checks ...

    toilerState[tokenId].latestTaxRate = plotMetadata[toiler.landlord].currentTaxRate;
    plotOccupied[toiler.landlord][oldPlotId] = Plot({occupied: false, tokenId: 0});
    plotOccupied[toiler.landlord][plotId] = Plot({occupied: true, tokenId: tokenId});

    // ... rest of the function ...
}
```

2. In the `_farmPlots` function, the old (incorrect) `plotId` is used to check for validity:

```javascript
if (getNumPlots(landlord) < _toiler.plotId) {
  timestamp = plotMetadata[landlord].lastUpdated;
  toilerState[tokenId].dirty = true;
}
```

This check could incorrectly mark a Munchable as dirty, even if it was moved to a valid plot, because the `plotId` wasn't updated during the transfer.

## Tools Used

Manual review

## Recommended Mitigation Steps

Update the `transferToUnoccupiedPlot` function to properly update the `plotId` in the `toilerState` mapping:

```diff
function transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)
    external
    override
    forceFarmPlots(msg.sender)
    notPaused
{
    // ... existing code ...

+   toilerState[tokenId].plotId = plotId; // Update the plot ID in toilerState
    plotOccupied[_toiler.landlord][oldPlotId] = Plot({occupied: false, tokenId: 0});
    plotOccupied[_toiler.landlord][plotId] = Plot({occupied: true, tokenId: tokenId});

    // ... rest of the function ...
}
```
