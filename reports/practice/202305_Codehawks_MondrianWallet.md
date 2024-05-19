# First Flight #15: Mondrian Wallet - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- [High Risk Findings](#high-risks-findings)
  - [H-01 Inadequate signature validation in `MondrianWallet::_validateSignature`](#H-01)
- [Medium Risk Findings](medium-risks-findings)
  - [M-01 Missing assignment of a random Mondrian artwork to the newly created `MondrianWallet`](#M-01)
  - [M-02 Uneven distribution of NFT artworks in `MondrianWallet::tokenURI`](#M-02)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #15: Mondrian Wallet

### Dates: May 9th, 2024 - May 16th, 2024

Our team loves account abstraction, and abstract art, so we decided to combine them! Users who create an account abstraction wallet with MondrianWallet will get a cool account abstraction wallet, with a random Mondrian art painting!

[See more contest details here](https://www.codehawks.com/contests/clvxt8idd00014zcc81dv6rde)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 2
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a>[H-01] Inadequate signature validation in `MondrianWallet::_validateSignature`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/main/contracts/MondrianWallet.sol#L119

## Summary

`MondrianWallet::_validateSignature` fails to verify the identity of the message signer.

## Vulnerability Details

`MondrianWallet::_validateSignature` is responsible for validating the signatures of operations passed through the `EntryPoint`. Currently, the function only confirms that a signature is technically valid without verifying if it was signed by an authorized party (e.g., the wallet owner or another trusted signer).

```javascript
// MondrianWallet.sol
function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
    internal
    pure
    returns (uint256 validationData)
{
    bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
@>  ECDSA.recover(hash, userOp.signature);
    return SIG_VALIDATION_SUCCESS;
}
```

## Impact

Failing to verify the signer allows any user who can craft a valid signature to pass signature checks, potentially leading to unauthorized actions being performed under the guise of valid operations via ``MondrianWallet::execute`, which is called in a future part of ERC-4337 transaction flow.

## Tools Used

Manual review

## Recommendations

Add a check within the `MondrianWallet::_validateSignature` function to ensure that the operation is authorized by verifying the signature against the owner of the wallet:

```diff
function _validateSignature(PackedUserOperation calldata userOp, bytes32 userOpHash)
    internal
    pure
    returns (uint256 validationData)
{
    bytes32 hash = MessageHashUtils.toEthSignedMessageHash(userOpHash);
-   ECDSA.recover(hash, userOp.signature);
+   require(ECDSA.recover(hash, userOp.signature) == owner(), "Invalid signature");
    return SIG_VALIDATION_SUCCESS;
}
```

<br>

# Medium Risk Findings

## <a id='M-01'></a>M-01. Missing assignment of a random Mondrian artwork to the newly created `MondrianWallet`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/main/contracts/MondrianWallet.sol#L59-L61

## Summary

`MondrianWallet` currently lacks the functionality to mint NFTs to users who deploy their own instance of it, as per project specifications.

## Vulnerability Details

`MondrianWallet` is intended to provide each user with a unique NFT featuring a Mondrian artwork upon creation of their wallet instance. However, the contract does not include functionality to mint these NFTs automatically.

## Impact

Users expecting to receive a unique and randomly assigned NFT artwork might be disappointed or misled, which could harm the project's reputation and user trust.

## Tools Used

Manual review

## Recommendations

To address these issues effectively, `MondrianWallet` should implement token minting on wallet creation, with a truly randomized `tokenId`, which could be supplied via the Chainlink VRF.

Here's the recommended steps:

- Calculate the address of the new `MondrianWallet` before deploying it
- Create a Chainlink VRF subscription and fund it with LINK tokens
- Add the following code to the `MondrianWallet.sol`:

```diff
+import "@chainlink/contracts/src/v0.8/vrf/interfaces/VRFCoordinatorV2Interface.sol";
+import "@chainlink/contracts/src/v0.8/vrf/VRFConsumerBaseV2.sol";

-contract MondrianWallet is Ownable, ERC721, IAccount {
+contract MondrianWallet is Ownable, ERC721, IAccount, VRFConsumerBaseV2 {
    // ...

    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
    IEntryPoint private immutable i_entryPoint;
+   uint256 private immutable i_requestId;

-   constructor(address entryPoint)
+   constructor(address entryPoint, address vrfCoordinator, bytes32 keyHash)
        Ownable(msg.sender)
        ERC721("MondrianWallet", "MW")
+       VRFConsumerBaseV2(vrfCoordinator)
    {
        i_entryPoint = IEntryPoint(entryPoint);

+       i_requestId = VRFCoordinatorV2Interface(vrfCoordinator).requestRandomWords(
+           keyHash,
+           subscriptionId, // TODO: Inject your subscription ID
+           3, // requestConfirmations
+           40000, // callbackGasLimit
+           1 // numWords
+       );
    }

+   function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
+       require(i_requestId == requestId, "Invalid requestId");

+       uint256 tokenId = randomWords[0] % 4;
+       mint(tokenId);
+   }

    // ...
}
```

<br>

## <a id='M-02'></a>M-02. Uneven distribution of NFT artworks in `MondrianWallet::tokenURI`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-Mondrian-Wallet/blob/main/contracts/MondrianWallet.sol#L165

## Summary

`MondrianWallet::tokenURI` uses a modulo operation to assign NFT artworks based on the token ID, which results in an unequal distribution of the artworks, particularly impacting the availability of `ART_FOUR`.

## Vulnerability Details

`MondrianWallet::tokenURI` is designed to return a URI for a Mondrian art piece associated with a specific NFT token ID. It employs a modulo operation (`tokenId % 10`) to select one of four art URIs. However, due to this method, the distribution of the art pieces becomes uneven. Specifically, `ART_FOUR` will be over-represented compared to the other artworks, because all token IDs resulting in a remainder of 3 to 9 (7 out of 10 possibilities) will be assigned `ART_FOUR`.

## Impact

This uneven distribution might lead to diminished value or interest in more frequently assigned artworks, potentially affecting the perceived rarity and value of the NFTs.

## Tools Used

Manual review

## Recommendations

To ensure an equal distribution of artworks, consider modifying the `modNumber` calculation to cycle through the art pieces evenly:

```diff
function tokenURI(uint256 tokenId) public view override returns (string memory) {
    // Previous code stays the same

+   uint256 modNumber = tokenId % 4;
-   uint256 modNumber = tokenId % 10;

    // Latter code stays the same
}
```

Adjusting the modulo operation from `% 10` to `% 4` ensures each artwork is assigned to exactly 25% of the token IDs.
