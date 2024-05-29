# First Flight #12: Kitty Connect - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- [High Risk Findings](#high-risks-findings)
  - [[H-01] Missing fee token approval for `IRouterClient` in `bridgeNftWithData`](#H-01)
  - [[H-02] Improper token ownership update in `_updateOwnershipInfo` leads to double ownership of the same Kitty](#H-02)
  - [[H-03] Failure to update owner's token ID array in `mintBridgedNFT` leads to ownership tracking inconsistencies and lost Kitties](#H-03)
  - [[H-04] Lack of reclaim mechanism for failed bridging operations in `KittyBridge`](#H-04)

<br>

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #12: Kitty Connect

### Dates: Mar 28th, 2024 - Apr 4th, 2024

Get pumped for the 2nd Community Submitted First Flight on CodeHawks, brought to us by Shikhar Agarwal! This project allows users to buy a cute cat from our branches and mint NFT for buying a cat. The NFT will be used to track the cat info and all related data for a particular cat corresponding to their token ids. Kitty Owner can also Bridge their NFT from one chain to another chain via Chainlink CCIP.

[See more contest details here](https://www.codehawks.com/contests/clu7ddcsa000fcc387vjv6rpt)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 0
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a>[H-01] Missing fee token approval for `IRouterClient` in `bridgeNftWithData`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyBridge.sol#L47-L77

## Summary

The `bridgeNftWithData` function in the `KittyBridge` contract attempts to bridge Kitties across chains using Chainlink's CCIP and requires a fee in LINK tokens to be paid to the `IRouterClient` contract. However, it does not include a step to approve the router contract to spend the required LINK fee on behalf of the `KittyBridge` contract.

## Vulnerability Details

ERC-20 tokens, such as LINK, require an owner to approve a spender to transfer tokens up to a specified allowance. The `bridgeNftWithData` function calculates the necessary fees in LINK tokens for the bridging operation and checks if the contract has sufficient LINK token balance. However, it overlooks the need to set an allowance for the router contract, assuming the contract's balance is sufficient for the fee. This assumption fails when the router contract attempts to run transfer LINK tokens from the `KittyBridge` contract to itself without the necessary approval, causing the transaction to revert.

## Impact

This vulnerability renders the `bridgeNftWithData` function inoperative, as it will always fail when attempting to bridge NFTs due to the inability to transfer LINK tokens as fees.

## Tools Used

Foundry, Manual Inspection

## Proof of Code

<details>
<summary>Code</summary>
Add the following code to the `KittyTest.t.sol` file:

```javascript
function test_bridgeNftWithData_FailsIfGasTokenNotApproved() public {
    vm.prank(kittyConnectOwner);
    kittyBridge.allowlistSender(networkConfig.router, true);

    // ******************** Bridge NFT with Data ********************
    string
        memory _catImage = "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62";
    string memory _catName = "Meowdy";
    string memory _catBreed = "Ragdoll";
    uint256 _catDob = block.timestamp;

    uint256 _tokenId = kittyConnect.getTokenCounter();

    vm.prank(partnerA);
    kittyConnect.mintCatToNewOwner(
        user,
        _catImage,
        _catName,
        _catBreed,
        _catDob
    );

    // Before bridging, confirm that the Bridge does not have enough LINK allowance for fees
    uint256 _kittyBridgeLinkAllowanceToRouter = IERC20(
        kittyBridge.getLinkToken()
    ).allowance(address(kittyBridge), address(networkConfig.router));
    assertEq(_kittyBridgeLinkAllowanceToRouter, 0);

    vm.prank(user);
    vm.expectRevert("ERC20: insufficient allowance");
    kittyConnect.bridgeNftToAnotherChain(
        networkConfig.otherChainSelector,
        address(kittyBridge),
        _tokenId
    );
    // *******************************************************
}
```

</details>

## Recommended Mitigation

Implement a LINK token approval step within the `bridgeNftWithData` function or as part of the initial setup/configuration of the `KittyBridge` contract to grant the router contract an allowance to spend LINK tokens on its behalf. This approval should be for an amount at least equal to the anticipated fee for the operation.

```diff
function bridgeNftWithData(
        uint64 _destinationChainSelector,
        address _receiver,
        bytes memory _data
    )
        external
        onlyAllowlistedDestinationChain(_destinationChainSelector)
        validateReceiver(_receiver)
        returns (bytes32 messageId)
    {
        // Create an EVM2AnyMessage struct in memory with necessary information for sending a cross-chain message
        Client.EVM2AnyMessage memory evm2AnyMessage = _buildCCIPMessage(
            _receiver,
            _data,
            address(s_linkToken)
        );

        // Initialize a router client instance to interact with cross-chain router
        IRouterClient router = IRouterClient(this.getRouter());

        // Get the fee required to send the CCIP message
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > s_linkToken.balanceOf(address(this))) {
            revert KittyBridge__NotEnoughBalance(
                s_linkToken.balanceOf(address(this)),
                fees
            );
        }

+       s_linkToken.approve(address(router), fees);

        // Code below stays the same
    }
```

<br>

## <a id='H-02'></a>[H-02] Improper token ownership update in `_updateOwnershipInfo` leads to double ownership of the same Kitty

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyConnect.sol#L181-L185

## Summary

`_updateOwnershipInfo` properly updates new owner's ownership of the Kitty, but fails to remove old owner's ownership of it.

## Vulnerability Details

Upon transferring a token from one owner to another via `safeTransferFrom`, `_updateOwnershipInfo` is called to update the contract's internal state to reflect this change. However, due to the lack of logic to remove the token ID from the `s_ownerToCatsTokenId` array of the previous owner, the contract's state inaccurately reflects the ownership of the token. This flaw can be exploited by the previous owner to perform actions as if they still possessed the token, including bridging it to another chain, thereby creating a discrepancy in ownership across different platforms and within the ecosystem.

## Impact

This vulnerability undermines the security and integrity of the token ownership model within the `KittyConnect` ecosystem. Exploiting this flaw can lead to multiple parties claiming ownership of the same token, resulting in confusion, loss of trust, and potential financial implications for the involved parties.

## Tools Used

Foundry

### Proof of Code

Add the following code to the `KittyTest.t.sol` file:

```javascript
function test_transferCatToNewOwner_DoesntUpdateOwnership() public {
    // ******************** Mint kitty ********************
    uint256 tokenId = kittyConnect.getTokenCounter();

    vm.prank(partnerA);
    kittyConnect.mintCatToNewOwner(
        user,
        "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62",
        "Meowdy",
        "Ragdoll",
        block.timestamp
    );
    // ****************************************************

    // ******************** Transfer kitty to a new owner ********************
    address newOwner = makeAddr("newOwner");

    vm.prank(user);
    kittyConnect.approve(newOwner, tokenId);
    vm.expectEmit(false, false, false, true, address(kittyConnect));
    emit CatTransferredToNewOwner(user, newOwner, tokenId);

    // Shop partner transfers to the new owner
    vm.prank(partnerA);
    kittyConnect.safeTransferFrom(user, newOwner, tokenId);

    uint256[] memory oldOwnerTokenIds = kittyConnect.getCatsTokenIdOwnedBy(
        user
    );
    uint256[] memory newOwnerTokenIds = kittyConnect.getCatsTokenIdOwnedBy(
        newOwner
    );

    // Confirm that they both own the same Kitty
    assertEq(oldOwnerTokenIds.length, 1);
    assertEq(newOwnerTokenIds.length, 1);

    assertEq(newOwnerTokenIds[0], tokenId);
    assertEq(oldOwnerTokenIds[0], tokenId);
    // **********************************************************************
}
```

## Recommendations

Implement logic within the `_updateOwnershipInfo` function to remove the token ID from the previous owner's `s_ownerToCatsTokenId` array. This update ensures the contract's state accurately reflects the current ownership of tokens and prevents the previous owner from interacting with tokens they no longer own.

```diff
function _updateOwnershipInfo(
    address currCatOwner,
    address newOwner,
    uint256 tokenId
) internal {
    s_catInfo[tokenId].prevOwner.push(currCatOwner);
    s_catInfo[tokenId].idx = s_ownerToCatsTokenId[newOwner].length;
    s_ownerToCatsTokenId[newOwner].push(tokenId);

+   // Remove the token from the previous owner's array
+   uint256 tokenIndex = s_catInfo[tokenId].idx;
+   uint256 lastTokenIndex = s_ownerToCatsTokenId[currCatOwner].length - 1;
+   uint256 lastTokenId = s_ownerToCatsTokenId[currCatOwner][lastTokenIndex];

+   // Swap and pop
+   s_ownerToCatsTokenId[currCatOwner][tokenIndex] = lastTokenId;
+   s_catInfo[lastTokenId].idx = tokenIndex;
+   s_ownerToCatsTokenId[currCatOwner].pop();
}
```

<br>

## <a id='H-03'></a>[H-03] Failure to update owner's token ID array in `mintBridgedNFT` leads to ownership tracking inconsistencies and lost Kitties

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyConnect.sol#L154-L179

## Summary

When bridging, users can lose their Kitties due to a lacking state update when the bridged Kitty gets minted.

## Vulnerability Details

The `mintBridgedNFT` function is designed to mint a new Kitty token upon receiving a CCIP message from another chain. It assignes the next available index (`idx`) for the newly minted Kitty, but it does not update the owner's token ID array (`s_ownerToCatsTokenId[catOwner]`). This oversight can lead to scenarios where two tokens claim the same index position, undermining the integrity of token ownership tracking within the contract.

## Impact

All of the bridged Kitties become unavailable upon arrival, because they will either:

- occupy unavailable indexes (because `index = s_ownerToCatsTokenId[catOwner].length` isn't reachable yet)
- overwrite/be overwritten by newly minted Kitties at that exact index, which would cause losses of all Kitties at the same index when bridging to a new chain via `bridgeNftToAnotherChain`.

## Tools Used

Foundry

### Proof of Code

Add the following code to the `KittyTest.t.sol` file:

```javascript
function test_MintBridgedNFT_UpdatesOwnerTokenArray() public {
        vm.prank(kittyConnectOwner);
        kittyBridge.allowlistSender(networkConfig.router, true);

        // ******************** Bridge kitty ********************
        bytes memory data = abi.encode(
            user,
            "Bridged Kitty #1",
            "Domestic",
            "ipfs://QmbxwGgBGrNdXPm84kqYskmcMT3jrzBN8LzQjixvkz4c62",
            block.timestamp,
            partnerA
        );

        Client.Any2EVMMessage memory message = Client.Any2EVMMessage({
            messageId: bytes32(0),
            sender: abi.encode(networkConfig.router),
            sourceChainSelector: networkConfig.otherChainSelector,
            data: data,
            destTokenAmounts: new Client.EVMTokenAmount[](0)
        });

        vm.prank(networkConfig.router);
        kittyBridge.ccipReceive(message);

        // Post-minting, fetch bridged Kitty's token ID and owner's token ID array
        uint256 bridgedKittyTokenId = kittyConnect.getTokenCounter() - 1;
        uint256[] memory ownerTokenIds = kittyConnect.getCatsTokenIdOwnedBy(
            user
        );

        // Check that the bridged Kitty's token ID is present in the owner's array
        bool isTokenIdPresent = false;
        for (uint256 i = 0; i < ownerTokenIds.length; i++) {
            if (ownerTokenIds[i] == bridgedKittyTokenId) {
                isTokenIdPresent = true;
                break;
            }
        }

        assertFalse(
            isTokenIdPresent,
            "Newly bridged token ID is actually in the owner's token array"
        );
        assertEq(kittyConnect.getCatInfo(bridgedKittyTokenId).idx, 0);
        // *******************************************************

        // ****** Mint new kitty and override bridged kitty ******
        vm.prank(partnerA);
        kittyConnect.mintCatToNewOwner(user, "ipfs_hash", "Bob", "Birman", 0);

        uint256 mintedKittyId = kittyConnect.getTokenCounter() - 1;
        ownerTokenIds = kittyConnect.getCatsTokenIdOwnedBy(
            user
        );

        // Check that the bridged Kitty's token ID is present in the owner's array
        for (uint256 i = 0; i < ownerTokenIds.length; i++) {
            if (ownerTokenIds[i] == mintedKittyId) {
                isTokenIdPresent = true;
                break;
            }
        }
        assertTrue(
            isTokenIdPresent,
            "Newly minted token ID isn't in the owner's token array"
        );
        assertEq(kittyConnect.getCatInfo(mintedKittyId).idx, 0);
        // *******************************************************
    }
```

## Recommendations

Ensure that every token minting operation, including those initiated through the `mintBridgedNFT` function, correctly updates the `s_ownerToCatsTokenId` mapping to reflect the new token ID.

```diff
function mintBridgedNFT(bytes memory data) external onlyKittyBridge {
        // Code above stays the same

        s_catInfo[tokenId] = CatInfo({
            catName: catName,
            breed: breed,
            image: imageIpfsHash,
            dob: dob,
            prevOwner: new address[](0),
            shopPartner: shopPartner,
            idx: s_ownerToCatsTokenId[catOwner].length
        });

+       s_ownerToCatsTokenId[catOwner].push(tokenId);

        emit NFTBridged(block.chainid, tokenId);
        _safeMint(catOwner, tokenId);
    }
```

<br>

## <a id='H-04'></a>[H-04] Lack of reclaim mechanism for failed bridging operations in `KittyBridge`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-03-kitty-connect/blob/main/src/KittyConnect.sol#L138

## Summary

In the event of a bridging failure, there is no mechanism in place for users to reclaim their Kitties, leading to a permanent loss of assets.

## Vulnerability Details

`KittyConnect` leverages a bridging approach that immediately burns the user's token upon initiation of the bridging process. While this design assumes successful completion of the cross-chain transfer, it does not account for potential failures in the bridging operation. Currently, there is no on-chain mechanism to revert the burn or allow users to reclaim their tokens if the operation fails, effectively resulting in a total loss of the token without recourse for recovery.

## Impact

Failed bridging operations would cause permanent loss of users' assets.

## Tools Used

Manual inspection

## Recommendations

Mitigation of this issue would require a thorough architecture redesign. Here are a few recommendations:

- Implement a two-phase burning mechanism, where the token is not burned prematurely and is instead locked in the `KittyBridge` until the bridging operation is confirmed
- Confirm successfully executed bridging operation using an off-chain component such as Chainlink Keepers or a custom relayer, which would inform the `KittyBridge` about the status of the bridging operation:
  - If the bridging successfully went through, finally burn the locked Kitty
  - If the bridging failed, allow users to reclaim their Kitties

## Note

This report has been disapproved by the lead judge, but I still think that it was important to note down. The CCIP part on the source chain would never go through due to [H-01](#H-01), but if that was not an issue and if something went wrong on the destination chain, it would cause a loss for the users on the source chain.

Lead judge's response: `I can't see a reason for this code to fail, please provide it.`

Here's the appeal I made:

```
The highlighted code does not fail per se, but it showcases a dangerous operation that cannot be reverted in case bridging fails. That is why I've written the recommendations, which could be implemented to remove the dangers of burning NFTs immediately upon initiating bridging and not being able to reclaim them if anything goes wrong.

The key point in my finding is that there's no tracking of users' balances in the bridge, which could be leveraged to reclaim Kitties from incomplete bridging operations.

I understand that the following is way out of scope, but I wanted to point out that it's necessary to have a similar mechanism that zkSync's ERC20 bridge has, called claimFailedDeposit, which allows user to prove that their transfer failed in order to get their tokens back. It would require large changes, which I did not include in the finding, which might've been a bad on my end.
```

Here's the CodeArena's snapshot of the [zkSync's ERC20 bridge](https://github.com/code-423n4/2022-10-zksync/blob/main/ethereum/contracts/bridge/L1ERC20Bridge.sol#L171-L187) for their audit.
