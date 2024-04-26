# First Flight #13: Baba Marta - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [[H-01] Incorrect tracking of accumulated rewards in `MartenitsaMarketplace::collectReward`](#H-01)
  - [[H-02] Missing access control on `MartenitsaToken::updateCountMartenitsaTokensOwner` allows manipulation of `MartenitsaMarketplace::collectReward`](#H-02)
- Medium Risk Findings
  - [[M-01] `MartenitsaEvent::stopEvent` does not reset participants, which prevents users who joined once from joining future events](#M-01)
  - [[M-02] `MartenitsaVoting::voteForMartenitsa` allows users' Martenitsa tokens to be voted upon](#M-02)

<br>

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #13

### Dates: Apr 11th, 2024 - Apr 18th, 2024

Every year on 1st March people in Bulgaria celebrate a centuries-old tradition called the day of Baba Marta ("Baba" means Grandma and "Mart" means March), related to sending off the winter and welcoming the approaching spring. The "Baba Marta" protocol allows you to buy `MartenitsaToken` and to give it away to friends!

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 2
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a>[H-01] Incorrect tracking of accumulated rewards in `MartenitsaMarketplace::collectReward`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaMarketplace.sol#L106

## Summary

Incorrect tracking of accumulated rewards allows for manipulation of reward distribution and users getting way more HealthTokens than they are eligible to.

```javascript
function collectReward() external {
    // ...
    uint256 amountRewards = (count / requiredMartenitsaTokens) - _collectedRewards[msg.sender];
    if (amountRewards > 0) {
@>      _collectedRewards[msg.sender] = amountRewards;
        healthToken.distributeHealthToken(msg.sender, amountRewards);
    }
}
```

## Vulnerability Details

`MartenitsaMarketplace::collectReward` incorrectly resets the count of collected rewards (`_collectedRewards`) using the `=` operator instead of properly accumulating them with the `+=` operator. This results in inaccurate tracking of the cumulative number of `HealthToken` a user has claimed, leading to potential over-issuance of rewards when users manipulate their token holdings.

## Impact

This vulnerability allows a user to claim more `HealthToken` than entitled by adjusting their `MartenitsaToken` count before each reward claim. The incorrect reset (`=`) of `_collectedRewards` means that if a user's total eligible rewards decrease or remain constant, the count of previously collected rewards can effectively be "forgotten," allowing the user to exploit the reward system.

## Tools Used

Foundry

## Proof of code

<details>
<summary>Code</summary>
Add the following code to the `MartenitsaMarketplace.t.sol` file:

```javascript
function _getMartenitsas(uint256 amount, uint256 tokenIdToStart) internal returns (uint256) {
    uint256 tokenId = tokenIdToStart;

    vm.startPrank(chasy);
    for(uint256 i = 0; i < amount; i++){
        martenitsaToken.createMartenitsa(string(abi.encodePacked("bracelet-", i)));
        marketplace.listMartenitsaForSale(tokenId, 1 wei);
        martenitsaToken.approve(address(marketplace), tokenId);
        marketplace.makePresent(bob, tokenId);
        tokenId++;
    }
    vm.stopPrank();

    return tokenId;
}

function test__CollectReward__ImproperTrackingOfPreviousRewards() public {
    // ********** Setup **********
    assertEq(healthToken.balanceOf(bob), 0);
    assertEq(martenitsaToken.balanceOf(bob), 0);

    uint256[] memory martenitsasCountToAdd = new uint256[](3);
    martenitsasCountToAdd[0] = 6; // should yield in 2 HealthToken  (total=2)
    martenitsasCountToAdd[1] = 3; // should yield in 1 HealthToken  (total=3)
    martenitsasCountToAdd[2] = 9; // should yield in 3 HealthToken  (total=6)

    uint256 tokenId = 0;
    uint256 totalCount = 0;

    for(uint256 i = 0; i < martenitsasCountToAdd.length; i++){
        uint256 currCountToAdd = martenitsasCountToAdd[i];
        totalCount += currCountToAdd;

        tokenId = _getMartenitsas(currCountToAdd, tokenId);
        assertEq(martenitsaToken.balanceOf(bob), totalCount);

        vm.prank(bob);
        marketplace.collectReward();
    }

    // actual: 8 vs expected: 6
    assertEq(healthToken.balanceOf(bob), (totalCount / 3) * 10 ** 18);

    /**
     * EXPLANATION:
     *
     * In case of `_collectedRewards[msg.sender] = amountRewards;` (vulnerable code)
     * 1. Bob has 6 MartenitsaToken
     * 1.a. Bob collects rewards
     * 1.b. `_collectedRewards[msg.sender] = 0`, amountRewards = 2 - 0 = 2
     * 1.c. --> _collectedRewards[msg.sender] = 2
     *
     * 2. Bob gets 3 more MartenitsaToken (total=9)
     * 2.a. Bob collects rewards
     * 2.b. `_collectedRewards[msg.sender] = 2`, amountRewards = 3 - 2 = 1
     * 2.c. --> _collectedRewards[msg.sender] = 1 (*issue* - should be `+= 1`, so it would be 3)
     *
     * EXPLOIT
     * 3. Bob gets 9 more MartenitsaToken (total=18)
     * 3.a. Bob collects rewards
     * 3.b. `_collectedRewards[msg.sender] = 1`, amountRewards = 6 - 1 = 5
     * 3.c. --> _collectedRewards[msg.sender] = 5
     *
     * --------------> healthToken.balanceOf(bob) = 8 (incorrect)

     * In case of `_collectedRewards[msg.sender] += amountRewards;` (fixed code)
     * 1. Bob has 6 MartenitsaToken
     * 1.a. Bob collects rewards
     * 1.b. `_collectedRewards[msg.sender] = 0`, amountRewards = 2 - 0 = 2
     * 1.c. --> _collectedRewards[msg.sender] += 2
     * 1.d. --> _collectedRewards[msg.sender] == 2
     *
     * 2. Bob gets 3 more MartenitsaToken (total=9)
     * 2.a. Bob collects rewards
     * 2.b. `_collectedRewards[msg.sender] = 2`, amountRewards = 3 - 2 = 1
     * 2.c. --> _collectedRewards[msg.sender] += 1
     * 2.d. --> _collectedRewards[msg.sender] == 3
     *
     * 3. Bob gets 9 more MartenitsaToken (total=18)
     * 3.a. Bob collects rewards
     * 3.b. `_collectedRewards[msg.sender] = 3`, amountRewards = 6 - 3 = 3
     * 3.c. --> _collectedRewards[msg.sender] += 3
     * 3.d. --> _collectedRewards[msg.sender] == 6
     *
     * --------------> healthToken.balanceOf(bob) = 6 (correct)
    */
}
```

</details>

## Recommendations

Change the assignment operation in the `MartenitsaMarketplace::collectReward` function to correctly accumulate the total rewards a user has claimed:

```diff
function collectReward() external {
    // ...
    uint256 amountRewards = (count / requiredMartenitsaTokens) - _collectedRewards[msg.sender];
    if (amountRewards > 0) {
-       _collectedRewards[msg.sender] = amountRewards;
+       _collectedRewards[msg.sender] += amountRewards;
        healthToken.distributeHealthToken(msg.sender, amountRewards);
    }
}
```

<br>

## <a id='H-02'></a>[H-02] Missing access control on `MartenitsaToken::updateCountMartenitsaTokensOwner` allows manipulation of `MartenitsaMarketplace::collectReward`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaToken.sol#L57-L70

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaMarketplace.sol#L103

## Summary

Due to lacking access control on `MartenitsaToken::updateCountMartenitsaTokensOwner`, reward distribution mechanism of `MartenitsaMarketplace::collectReward` can be heavily manipulated.

```javascript
// MartenitsaMarketplace.sol
function collectReward() external {
    // ...
@>  uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(msg.sender);
    // ...
}
```

## Vulnerability Details

`MartenitsaToken::updateCountMartenitsaTokensOwner` lacks appropriate access control, allowing any user to freely modify the count of Martenitsa tokens they own. This oversight permits the exploitation of the `MartenitsaMarketplace::collectReward` function, enabling users to claim Health tokens without legitimately owning the requisite number of Martenitsa tokens.

## Impact

This vulnerability allows any user to inflate their `countMartenitsaTokensOwner` arbitrarily. The inflated count directly influences the reward mechanism, leading to unauthorized claim of Health tokens. Such actions can deplete the resources meant for genuine participants and disrupt the integrity and financial stability of the platform.

## Tools Used

Foundry

## Proof of Code

<details>
<summary>Code</summary>
Add the following code to the `MartenitsaMarketplace.t.sol` file:

```javascript
function test__CollectRewards__WithoutAnyMartenitsas() public {
    // ********** Setup **********
    assertEq(healthToken.balanceOf(bob), 0); // Bob has 0 HT
    assertEq(martenitsaToken.balanceOf(bob), 0); // Bob has 0 Martenitsas

    // ********** Bob increases their Martenitsa count due to lacking access control **********
    vm.startPrank(bob);
    martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
    martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");
    martenitsaToken.updateCountMartenitsaTokensOwner(bob, "add");

    // ********** Bob claims their HTs without having any Martenitsas **********
    marketplace.collectReward();
    assertEq(healthToken.balanceOf(bob), 10 ** 18); // Bob gets 1 HT
    vm.stopPrank();
}
```

</details>

## Recommendations

Remove arbitrary tracking of balances via `countMartenitsaTokensOwner` and start using actual `MartenitsaToken` balances to calculate rewards:

```diff
// MartenitsaToken.sol
-    mapping(address => uint256) public countMartenitsaTokensOwner;

-    function updateCountMartenitsaTokensOwner(address owner, string memory operation) external {
-
-        if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("add"))) {
-            countMartenitsaTokensOwner[owner] += 1;
-        } else if (keccak256(abi.encodePacked(operation)) == keccak256(abi.encodePacked("sub"))) {
-            countMartenitsaTokensOwner[owner] -= 1;
-        } else {
-            revert("Wrong operation");
-        }
-    }
```

```diff
// MartenitsaMarketplace.sol
function collectReward() external {
    // ...
-   uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(msg.sender);
+   uint256 count = martenitsaToken.balanceOf(msg.sender);
    // ...
}
```

<br>

# Medium Risk Findings

## <a id='M-01'></a>[M-01] `MartenitsaEvent::stopEvent` does not reset participants, which prevents users who joined once from joining future events

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaEvent.sol#L57-L65

## Summary

Once the user joins a Martenitsa event, they are unable to join future events due to bad participants management after the event ends.

## Vulnerability Details

The `MartenitsaEvent` smart contract implements an event system where users can join and temporarily become producers, so that they can create and sell `MartenitsaToken`s.

However, once the user joins an event, they are permanently recorded in the `_participants` mapping and never removed from it, preventing them from joinining future events.

## Impact

This issue leads to unintended denial of participation for users who are eligible to join Martenitsa events by having enough `HealthToken`s.

## Tools Used

Foundry, manual review

## Proof of Code

<details>
<summary>Code</summary>
Add the following code to the `MartenitsaEvent.t.sol` file:

```javascript
modifier getMartenitsas(uint256 amount) {
    uint256 tokenId = 0;

    vm.startPrank(chasy);
    for(uint256 i = 0; i < amount; i++){
        martenitsaToken.createMartenitsa(string(abi.encodePacked("bracelet-", i)));
        marketplace.listMartenitsaForSale(tokenId, 1 wei);
        martenitsaToken.approve(address(marketplace), tokenId);
        marketplace.makePresent(bob, tokenId);
        tokenId++;
    }
    vm.stopPrank();

    _;
}

function test__JoinEvent__FailsAfterInitialJoin() public getMartenitsas(6) {
    // ********** Setup - Bob gets their 2 HTs **********
    vm.startPrank(bob);
    marketplace.collectReward();
    assertEq(healthToken.balanceOf(bob), 2 * 10**18); // Confirm that Bob has enough HT to join the event
    healthToken.approve(address(martenitsaEvent), type(uint256).max);
    vm.stopPrank();

    // ********** Start event **********
    martenitsaEvent.startEvent(1 days);

    // ********** Bob joins the event *********
    vm.startPrank(bob);
    martenitsaEvent.joinEvent();
    assertEq(healthToken.balanceOf(bob), 1 * 10**18); // Bob sent their 1 HT as a requirement to join
    vm.stopPrank();

    // ********** Stop the event and start it again **********
    vm.warp(block.timestamp + 1 days + 1);
    martenitsaEvent.stopEvent();
    martenitsaEvent.startEvent(1 days);

    // ********** Attempt to join the event again **********
    vm.startPrank(bob);
    vm.expectRevert("You have already joined the event");
    martenitsaEvent.joinEvent();
    vm.stopPrank();

    assertEq(martenitsaEvent.participants(0), bob); // Confirm that Bob is still a participant
    assertEq(martenitsaEvent.isProducer(bob), false); // but is not a producer
}
```

</details>

## Recommendations

Modify the `MartenitsaEvent::stopEvent` function to reset both the participants array and the participants mapping, allowing them to rejoin future events:

```diff
function stopEvent() external onlyOwner {
    require(block.timestamp >= eventEndTime, "Event is not ended");

    for (uint256 i = 0; i < participants.length; i++) {
        isProducer[participants[i]] = false;
+       delete _participants[participants[i]];
    }

+   delete participants;
}
```

<br>

## <a id='M-02'></a>[M-02] `MartenitsaVoting::voteForMartenitsa` allows users' Martenitsa tokens to be voted upon

### Relevant GitHub Links

https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaVoting.sol#L39-L52

## Summary

Producers can transfer their Martenitsa tokens to users, which can still be voted upon during the voting event, which shouldn't be the case.

## Vulnerability Details

`MartenitsaVoting::voteForMartenitsa` incorrectly enforces the rule that only tokens owned by producers can be voted upon. This restriction is intended to ensure that only MartenitsaTokens produced by active, verified producers are considered in voting to maintain quality and relevancy.

However, the current implementation does not effectively enforce this rule when tokens change hands from producers to non-producers, leading to a scenario where non-producers can vote on tokens that should no longer be eligible under the specified criteria.

## Impact

This flaw allows non-producer-owned tokens (which have been transferred from producers) to still be voted upon, contrary to the intended functionality. This can undermine the integrity of the voting process, allowing less relevant tokens to influence the results.

## Tools Used

Foundry, manual review

## Proof of code

<details>
<summary>Code</summary>
Add the following code to the `MartenitsaVoting.t.sol` file:

```javascript
function test__VoteForMartenitsa__NonProducersMartenitsa() public listMartenitsa {
    // ********** Setup **********
    uint256 tokenId = 0;
    vm.startPrank(chasy);
    martenitsaToken.approve(address(marketplace), tokenId);
    marketplace.makePresent(bob, tokenId); // Chasy as the producer gifts listed token to Bob
    vm.stopPrank();

    // ********** Bob as the user votes for their own Martenitsa **********
    vm.prank(bob);
    voting.voteForMartenitsa(tokenId);
}
```

</details>

## Recommendations

Implement an additional check in the `MartenitsaVoting::voteForMartenitsa` function to verify that the owner of the specified `tokenId` is a producer.

```diff
function voteForMartenitsa(uint256 tokenId) external {
+   require(_martenitsaToken.isProducer(_martenitsaToken.ownerOf(tokenId)), "You can only vote for producers' tokens");
    // ...
}
```
