# Titles Protocol - Findings Report

## Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [[H-01] Incorrect routing of collection fees in `FeeManager::_splitProtocolFee`](#H-01)
- Medium Risk Findings
  - [[M-01] `TitlesGraph::_setAcknowledged` does not properly update the `acknowledged` flag](#M-01)

<br>

## <a id='contest-summary'></a>Contest Summary

### Sponsor: Titles Protocol

### Dates: Apr 22nd, 2024 - Apr 26th, 2024

TITLES builds creative tools powered by artist-owned AI models. The underlying TITLES protocol enables the publishing of referential NFTs, including managing attribution and splitting payments with the creators of the attributed works.

[See more contest details here](https://audits.sherlock.xyz/contests/326)

## <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a>[H-01] Incorrect routing of collection fees in `FeeManager::_splitProtocolFee`

### Summary

`FeeManager::_splitProtocolFee` incorrectly routes the collection referrer's share of the fees to the mint referrer instead of the intended collection referrer.

### Vulnerability Detail

The issue arises due to the misuse of the `referrer_` parameter in the `FeeManager::_splitProtocolFee` function. This parameter is incorrectly used for routing both the mint referrer and collection referrer shares. Specifically, the function fails to differentiate between the individual who referred the minting of a specific token (`referrer_`) and the entity responsible for referring the entire collection (contained in the `referrers` mapping), leading to potential misallocation of fees.

### Impact

This causes improper distribution of fees between involved parties, causing a disbalance in protocol's parties' funding.

### Code Snippet

[FeeManager.sol#L438](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/fees/FeeManager.sol#L438)

### Tool used

Manual Review

### Recommendation

To rectify this vulnerability, the routing logic within `_splitProtocolFee` should properly route collection referrer's fee share as follows:

```diff
_route(
    Fee({asset: asset_, amount: collectionReferrerShare}),
-   Target({target: referrer_, chainId: block.chainid}),
+   Target({target: referrers[edition_], chainId: block.chainid}),
    payer_
);
```

---

# Medium Risk Findings

## <a id='M-01'></a>[M-01] `TitlesGraph::_setAcknowledged` does not properly update the `acknowledged` flag

### Summary

`TitlesGraph::_setAcknowledged` does not properly update the `acknowledged` flag of the Edge struct

### Vulnerability Detail

When acknowledging an `Edge` via `TitlesGraph::_setAcknowledged`, the `acknowledged` flag is not properly updated, due to the fact that the function is updating the return value of type `Edge memory`, changes to which is not persisted in storage.

### Impact

Failed update would violate the sanctity of the data in the `TitlesGraph`, leading to potential future issues if the `acknowledged` flag was to be leveraged to validate `Edge` traversal.

### Code Snippet

[TitlesGraph.sol#L202](https://github.com/sherlock-audit/2024-04-titles/blob/main/wallflower-contract-v2/src/graph/TitlesGraph.sol#L202)

### Tool used

Manual Review, Foundry

<details>
<summary>Proof of Code</summary>
Add the following code to the `TitlesGraph.t.sol` file:

```javascript
function test_acknowledgeEdge_doesntUpdateStorage() public {
    // ********** Setup **********
    Node memory from = Node({
        nodeType: NodeType.COLLECTION_ERC1155,
        entity: Target({target: address(1), chainId: block.chainid}),
        creator: Target({target: address(2), chainId: block.chainid}),
        data: ""
    });

    Node memory to = Node({
        nodeType: NodeType.TOKEN_ERC1155,
        entity: Target({target: address(3), chainId: block.chainid}),
        creator: Target({target: address(4), chainId: block.chainid}),
        data: abi.encode(42)
    });

    // Only the `from` node's entity can create the edge.
    vm.prank(from.entity.target);
    titlesGraph.createEdge(from, to, "");

    vm.expectEmit(true, true, true, true);
    emit IEdgeManager.EdgeAcknowledged(
        Edge({from: from, to: to, acknowledged: true, data: ""}), to.creator.target, ""
    );
    // ***************************

    // ********** Acknowledge the edge **********
    // Only the `to` node's creator (or the entity itself) can acknowledge it
    bytes32 _edgeId = keccak256(abi.encode(from, to));

    vm.prank(to.creator.target);
    titlesGraph.acknowledgeEdge(_edgeId, "");
    // ******************************************

    // *** Confirm that the update didn't go through ***
    (,, bool acknowledged,) = titlesGraph.edges(_edgeId);
    assertFalse(acknowledged);
}
```

</details>

### Recommendation

Change the type of the return value to Edge storage, so that it would properly update the storage variable.

```diff
function _setAcknowledged(bytes32 edgeId_, bytes calldata data_, bool acknowledged_)
        internal
-       returns (Edge memory edge)
+       returns (Edge storage edge)
{
    // ...
}
```
