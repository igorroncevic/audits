# Tadle - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings

  - [[H-01] Lack of token approval from `CapitalPool` to `TokenManager` in `TokenManager::withdraw` leads to DoS of all withdrawals](#H-01)
  - [[H-02] Incorrect approval in `TokenManager::_transfer` leads to DoS of all withdrawals](#H-02)
  - [[H-03] Incorrect authorization check in `DeliveryPlace::settleAskTaker` allows maker to settle taker's ask order and mistakenly trigger a deposit instead of the taker](#H-03)
  - [[H-04] Lack of balance reduction in `TokenManager::withdraw` allows unlimited withdrawals](#H-04)
  - [[H-05] Unauthorized manipulation of referral rates in `SystemConfig::updateReferrerInfo` allows for increased fee allocation](#H-05)

- Low Risk Findings
  - [[L-01] Unclaimed platform fees accumulate indefinitely in maker info, potentially leading to loss of funds](#L-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Tadle

### Dates: Aug 5th, 2024 - Aug 12th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-tadle)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 5
- Medium: 0
- Low: 1

<br>

# High Risk Findings

## <a id='H-01'></a>[H-01] Lack of token approval from `CapitalPool` to `TokenManager` in `TokenManager::withdraw` leads to DoS of all withdrawals

## Summary

`TokenManager::withdraw` uses `_safe_transfer_from` instead of `_transfer`, which doesn't handle the necessary approvals. This causes all withdrawal attempts from the `CapitalPool` to fail, effectively locking user funds in the protocol.

[TokenManager.sol#L175-L180](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/core/TokenManager.sol#L175-L180)

```javascript
// src/core/TokenManager.sol
function withdraw(address _tokenAddress, TokenBalanceType _tokenBalanceType) external whenNotPaused {
    // existing code...

    if (_tokenAddress == wrappedNativeToken) {
        // existing code...
    } else {
@>      _safe_transfer_from(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount);
    }
}
```

[Rescuable.sol#L104-L117](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/utils/Rescuable.sol#L104-L117)

```javascript
// src/utils/Rescuable.sol
function _safe_transfer_from(address token, address from, address to, uint256 amount) internal {
@> (bool success,) = token.call(abi.encodeWithSelector(TRANSFER_FROM_SELECTOR, from, to, amount));

    if (!success) {
        revert TransferFailed();
    }
}
```

## Impact

This bug renders the withdrawal functionality completely inoperable. Users are unable to retrieve their funds from the protocol, leading to a loss of trust and potential financial losses if users need immediate access to their assets.

## Tools Used

Foundry, Manual code review

## Proof of Concept

Place the following test in `PreMarkets.t.sol`:

```javascript
// import {Rescuable} from "../src/utils/Rescuable.sol";
function test_withdraw_insufficientAllowance() public {
    // ~~~~~~~~~~~~~~~~~~~~ Setup ~~~~~~~~~~~~~~~~~~~~
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 INITIAL_BALANCE = 1000 * 1e18;
    uint256 MAKER_POINTS = 10_000;
    uint256 TAKER_POINTS = 5_000;
    uint256 TOKEN_AMOUNT = 100 * 1e18;
    uint256 COLLATERAL_RATE = 10_000 * 1.2; // 120% collateral rate
    uint256 EACH_TRADE_TAX = 10_000 * 0.03; // 3% trade tax

    // Provide Alice and Bob with initial token balances
    deal(address(mockUSDCToken), alice, INITIAL_BALANCE);
    deal(address(mockUSDCToken), bob, INITIAL_BALANCE);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 1) Create a maker offer ~~~~~~~~~~
    vm.startPrank(alice);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createMaker(
        CreateMakerParams(
            marketPlace,
            address(mockUSDCToken),
            MAKER_POINTS,
            TOKEN_AMOUNT,
            COLLATERAL_RATE,
            EACH_TRADE_TAX,
            OfferType.Ask,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 2) Create a taker order ~~~~~~~~~~
    vm.startPrank(bob);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createTaker(offerAddr, TAKER_POINTS);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 3) Attempt withdrawal (should fail due to insufficient allowance) ~~~~~~~~~~
    vm.startPrank(alice);
    uint256 aliceBalanceBeforeWithdrawal = mockUSDCToken.balanceOf(alice);
    uint256 aliceWithdrawableBalance =
        tokenManager.userTokenBalanceMap(alice, address(mockUSDCToken), TokenBalanceType.SalesRevenue);

    console.log("> Alice's actual balance BEFORE withdrawal attempt: %18e", aliceBalanceBeforeWithdrawal);
    console.log("> Alice's withdrawable balance: %18e\n", aliceWithdrawableBalance);

    vm.expectRevert(Rescuable.TransferFailed.selector);
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);
    uint256 aliceBalanceAfterWithdrawal = mockUSDCToken.balanceOf(alice);
    console.log(">> Alice's actual balance AFTER withdrawal attempt: %18e\n", aliceBalanceAfterWithdrawal);

    assertEq(
        aliceBalanceBeforeWithdrawal,
        aliceBalanceAfterWithdrawal,
        "Alice's balance should not change due to failed withdrawal"
    );
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 4) Check capital pool allowance ~~~~~~~~~~
    uint256 capitalPoolAllowance = mockUSDCToken.allowance(address(capitalPool), address(tokenManager));
    console.log(">>> Capital Pool allowance for TokenManager: %18e", capitalPoolAllowance);
    assertEq(capitalPoolAllowance, 0, "Capital Pool should not have given allowance to TokenManager");
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
}
```

## Recommendations

Replace `_safe_transfer_from` with `_transfer` in the `TokenManager::withdraw` function, so that necessary token approvals are properly handled:

```diff
function withdraw(address _tokenAddress, TokenBalanceType _tokenBalanceType) external whenNotPaused {
    // existing code...

    if (_tokenAddress == wrappedNativeToken) {
        // existing code...
    } else {
-       _safe_transfer_from(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount);
+       _transfer(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount, capitalPoolAddr);
    }
}
```

NOTE: This still won't fully fix the allowance issue, due to the next vulnerability: "Incorrect approval in `TokenManager::_transfer` leads to DoS of all withdrawals", which is covered in a different submission.

## <a id='H-02'></a>[H-02] Incorrect approval in `TokenManager::_transfer` leads to DoS of all withdrawals

## Summary

`TokenManager::_transfer` incorrectly approves the `TokenManager`'s address instead of the token address when setting up allowances for the capital pool. This error causes all withdrawal attempts to fail, effectively locking user funds in the protocol.

[TokenManager.sol#L247](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/core/TokenManager.sol#L247)

```javascript
// src/core/TokenManager.sol
function _transfer(address _token, address _from, address _to, uint256 _amount, address _capitalPoolAddr) internal {
    uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
    uint256 toBalanceBef = IERC20(_token).balanceOf(_to);

    if (_from == _capitalPoolAddr && IERC20(_token).allowance(_from, address(this)) == 0x0) {
@>      ICapitalPool(_capitalPoolAddr).approve(address(this));
    }

    // rest of the function...
}
```

[CapitalPool.sol#L28-L34](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/core/CapitalPool.sol#L28-L34)

```javascript
// src/core/CapitalPool.sol
contract CapitalPool is CapitalPoolStorage, Rescuable, ICapitalPool {
    bytes4 private constant APPROVE_SELECTOR = bytes4(keccak256(bytes("approve(address,uint256)")));

    constructor() Rescuable() {}

    function approve(address tokenAddr) external {
        address tokenManager = tadleFactory.relatedContracts(RelatedContractLibraries.TOKEN_MANAGER);
@>      (bool success,) = tokenAddr.call(abi.encodeWithSelector(APPROVE_SELECTOR, tokenManager, type(uint256).max));

        if (!success) {
            revert ApproveFailed();
        }
    }
}
```

## Impact

This bug renders the withdrawal functionality completely inoperable. Users are unable to retrieve their funds from the protocol.

## Tools Used

Foundry, Manual code review

## Proof of Concept

Place the following test in `PreMarkets.t.sol`:

```javascript
// import {ICapitalPool} from "../src/interfaces/ICapitalPool.sol";
function test_transfer_invalidApprovalForCapitalPool() public {
    // ~~~~~~~~~~~~~~~~~~~~ Setup ~~~~~~~~~~~~~~~~~~~~
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 INITIAL_BALANCE = 1000 * 1e18;
    uint256 MAKER_POINTS = 1000;
    uint256 TAKER_POINTS = 500;
    uint256 TOKEN_AMOUNT = 100 * 1e18;
    uint256 COLLATERAL_RATE = 10_000 * 1.2; // 120% collateral rate
    uint256 EACH_TRADE_TAX = 10_000 * 0.03; // 3% trade tax

    deal(address(mockUSDCToken), alice, INITIAL_BALANCE);
    deal(address(mockUSDCToken), bob, INITIAL_BALANCE);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 1) Create a maker offer ~~~~~~~~~~
    vm.startPrank(alice);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createMaker(
        CreateMakerParams(
            marketPlace,
            address(mockUSDCToken),
            MAKER_POINTS,
            TOKEN_AMOUNT,
            COLLATERAL_RATE,
            EACH_TRADE_TAX,
            OfferType.Ask,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 2) Create a taker order  ~~~~~~~~~~
    vm.startPrank(bob);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createTaker(offerAddr, TAKER_POINTS);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~ 3) Withdrawal attempt - should revert due to bad 'approve' parameter ~~~
    vm.startPrank(alice);
    vm.expectRevert(ICapitalPool.ApproveFailed.selector);
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);
    //  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
}
```

## Recommended Mitigation Steps

Correct the approval in the `_transfer` function:

```diff
function _transfer(address _token, address _from, address _to, uint256 _amount, address _capitalPoolAddr) internal {
    uint256 fromBalanceBef = IERC20(_token).balanceOf(_from);
    uint256 toBalanceBef = IERC20(_token).balanceOf(_to);

    if (_from == _capitalPoolAddr && IERC20(_token).allowance(_from, address(this)) == 0x0) {
-       ICapitalPool(_capitalPoolAddr).approve(address(this));
+       ICapitalPool(_capitalPoolAddr).approve(_token);
    }

    // rest of the function...
}
```

## <a id='H-03'></a>[H-03] Incorrect authorization check in `DeliveryPlace::settleAskTaker` allows maker to settle taker's ask order and mistakenly trigger a deposit instead of the taker

## Summary

`DeliveryPlace::settleAskTaker` has an incorrect authorization check that allows the maker to settle the taker's ask instead of the taker themselves. It also incorrectly requires makers to deposit tokens to settle takers' orders.

[DeliveryPlace.sol#L361](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/core/DeliveryPlace.sol#L361)

```javascript
// src/core/DeliveryPlace.sol
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    // previous code...

    if (status == MarketPlaceStatus.AskSettling) {
@>      if (_msgSender() != offerInfo.authority) {
            revert Errors.Unauthorized();
        }
    }else {
        // existing code...
    }


    uint256 settledPointTokenAmount = marketPlaceInfo.tokenPerPoint * _settledPoints;
    ITokenManager tokenManager = tadleFactory.getTokenManager();

    if (settledPointTokenAmount > 0) {
@>      tokenManager.tillIn(_msgSender(), marketPlaceInfo.tokenAddress, settledPointTokenAmount, true);

        // the rest of the function...
    }
}
```

## Impact

This vulnerability can lead to unintended financial losses for makers as they are forced to deposit their own tokens to settle orders that should be settled by takers.

## Proof of Concept

Place the following test in `PreMarkets.t.sol`:

```javascript
function test_makerDepositsInsteadOfTaker_issue() public {
    // ~~~~~~~~~~~~~~~~~~~~ Setup ~~~~~~~~~~~~~~~~~~~~
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 INITIAL_BALANCE = 1000 * 1e18;
    uint256 MAKER_POINTS = 10_000;
    uint256 TAKER_POINTS = 5_000;

    // Provide Alice and Bob with initial token balances
    deal(address(mockUSDCToken), alice, INITIAL_BALANCE);
    deal(address(mockUSDCToken), bob, INITIAL_BALANCE);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 1) Create a maker offer ~~~~~~~~~~
    vm.startPrank(alice);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createMaker(
        CreateMakerParams(
            marketPlace,
            address(mockUSDCToken),
            MAKER_POINTS,
            0.01 * 1e18,
            10_000 * 1.2, // 120% collateral rate
            10_000 * 0.05, // 5% each trade tax
            OfferType.Bid,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 2) Create a taker order ~~~~~~~~~~
    vm.startPrank(bob);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createTaker(offerAddr, TAKER_POINTS);
    address stockAddr = GenerateAddress.generateStockAddress(1);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 3) Update marketplace to AskSettling status ~~~~~~~~~~
    vm.startPrank(user1);
    systemConfig.updateMarket("Backpack", address(mockUSDCToken), 0.01 * 1e18, block.timestamp - 1, 3600);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // NOTE: with current implementation, if Bob tries to trigger the function, he'll run into `revert Errors.Unauthorized()`
    // therefore, we're running the test in order to allow it to pass through the authorization and demonstrate the currently available flow
    // by calling the function as Alice, who's a maker in this case

    // ~~~~~~~~~~ 4) Issue: Alice attempts to settle the ask, which should be done by Bob ~~~~~~~~~~
    vm.startPrank(alice);
    uint256 aliceBalanceBeforeSettlement = mockUSDCToken.balanceOf(alice);
    uint256 bobBalanceBeforeSettlement = mockUSDCToken.balanceOf(bob);

    deliveryPlace.settleAskTaker(stockAddr, TAKER_POINTS); // This should not change Alice's balance if correct

    uint256 aliceBalanceAfterSettlement = mockUSDCToken.balanceOf(alice);
    uint256 bobBalanceAfterSettlement = mockUSDCToken.balanceOf(bob);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    console.log("> Alice's balance BEFORE settleAskTaker: %18e", aliceBalanceBeforeSettlement);
    console.log("> Bob's balance BEFORE settleAskTaker: %18e\n", bobBalanceBeforeSettlement);
    console.log(">> Alice's balance AFTER settleAskTaker: %18e", aliceBalanceAfterSettlement);
    console.log(">> Bob's balance AFTER settleAskTaker: %18e\n", bobBalanceAfterSettlement);

    // Check balances to confirm if Alice's balance decreased, indicating she deposited tokens
    assertEq(
        aliceBalanceBeforeSettlement,
        aliceBalanceAfterSettlement,
        "Alice's balance should not change, indicating incorrect token flow."
    );
}
```

The aforementioned test should yield the following output:

```bash
Ran 1 test for test/PreMarkets.t.sol:PreMarketsTest
[FAIL. Reason: assertion failed] test_makerDepositsInsteadOfTaker_issue() (gas: 1293294)
Logs:
  > Alice's balance BEFORE settleAskTaker: 999.99
  > Bob's balance BEFORE settleAskTaker: 999.993725

  >> Alice's balance AFTER settleAskTaker: 949.99
  >> Bob's balance AFTER settleAskTaker: 999.993725

  Error: Alice's balance should not change, indicating incorrect token flow.
  Error: a == b not satisfied [uint]
    Expected: 949990000000000000000
      Actual: 999990000000000000000
```

## Recommended Mitigation Steps

Correct the authorization check to ensure that only the taker can call `settleAskTaker`:

```diff
function settleAskTaker(address _stock, uint256 _settledPoints) external {
    // previous code...

    if (status == MarketPlaceStatus.AskSettling) {
-       if (_msgSender() != offerInfo.authority) {
+       if (_msgSender() != stockInfo.authority) {
            revert Errors.Unauthorized();
        }
    }else{
        // the rest of the function...
    }
}
```

Now we can update the previous test to allow Bob to settle his order:

```javascript
function test_makerDepositsInsteadOfTaker_fixed() public {
    // ~~~~~~~~~~~~~~~~~~~~ Setup ~~~~~~~~~~~~~~~~~~~~
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 INITIAL_BALANCE = 1000 * 1e18;
    uint256 MAKER_POINTS = 10_000;
    uint256 TAKER_POINTS = 5_000;

    // Provide Alice and Bob with initial token balances
    deal(address(mockUSDCToken), alice, INITIAL_BALANCE);
    deal(address(mockUSDCToken), bob, INITIAL_BALANCE);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 1) Create a maker offer ~~~~~~~~~~
    vm.startPrank(alice);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createMaker(
        CreateMakerParams(
            marketPlace,
            address(mockUSDCToken),
            MAKER_POINTS,
            0.01 * 1e18,
            10_000 * 1.2, // 120% collateral rate
            10_000 * 0.05, // 5% each trade tax
            OfferType.Bid,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 2) Create a taker order ~~~~~~~~~~
    vm.startPrank(bob);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createTaker(offerAddr, TAKER_POINTS);
    address stockAddr = GenerateAddress.generateStockAddress(1);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 3) Update marketplace to AskSettling status ~~~~~~~~~~
    vm.startPrank(user1);
    systemConfig.updateMarket("Backpack", address(mockUSDCToken), 0.01 * 1e18, block.timestamp - 1, 3600);
    vm.stopPrank();
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 4) Fix: Now Bob attempts to settle the ask, not Alice ~~~~~~~~~~      <<<<
    vm.startPrank(bob);
    uint256 aliceBalanceBeforeSettlement = mockUSDCToken.balanceOf(alice);
    uint256 bobBalanceBeforeSettlement = mockUSDCToken.balanceOf(bob);

    deliveryPlace.settleAskTaker(stockAddr, TAKER_POINTS); // This should not change Alice's balance if correct

    uint256 aliceBalanceAfterSettlement = mockUSDCToken.balanceOf(alice);
    uint256 bobBalanceAfterSettlement = mockUSDCToken.balanceOf(bob);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    console.log("> Alice's balance BEFORE settleAskTaker: %18e", aliceBalanceBeforeSettlement);
    console.log("> Bob's balance BEFORE settleAskTaker: %18e\n", bobBalanceBeforeSettlement);
    console.log(">> Alice's balance AFTER settleAskTaker: %18e", aliceBalanceAfterSettlement);
    console.log(">> Bob's balance AFTER settleAskTaker: %18e\n", bobBalanceAfterSettlement);

    // Check balances to confirm if Alice's balance decreased, indicating she deposited tokens
    assertEq(
        aliceBalanceBeforeSettlement,
        aliceBalanceAfterSettlement,
        "Alice's balance should not change, indicating incorrect token flow."
    );
}
```

The updated test should yield the following output:

```bash
[PASS] test_makerDepositsInsteadOfTaker_fixed() (gas: 1278601)
Logs:
  > Alice's balance BEFORE settleAskTaker: 999.99
  > Bob's balance BEFORE settleAskTaker: 999.993725

  >> Alice's balance AFTER settleAskTaker: 999.99
  >> Bob's balance AFTER settleAskTaker: 949.993725
```

## <a id='H-04'></a>[H-04] Lack of balance reduction in `TokenManager::withdraw` allows unlimited withdrawals

## Summary

`TokenManager::withdraw` allows users to withdraw their balance, but it fails to reduce the user's balance after a successful withdrawal. This oversight enables users with any balance to repeatedly withdraw the same amount, potentially draining the entire `CapitalPool`.

## Impact

This vulnerability could lead to a complete loss of funds in the `CapitalPool`. Any user with a positive balance can drain the `CapitalPool` by withdrawing their balance multiple times, far exceeding their actual holdings. This could result in:

1. Depletion of the `CapitalPool`'s funds
2. Inability of other users to withdraw their legitimate balances

## Proof of Concept

The vulnerability exists in the `withdraw` function of the TokenManager contract, which retrieves the user's balance, transfers the funds, but never reduces the balance. A malicious user could exploit this by repeatedly calling the `withdraw` function.

NOTE: this PoC requires adding fixes from my other findings titled "Lack of token approval from `CapitalPool` to `TokenManager` in `TokenManager::withdraw` leads to DoS of all withdrawals" and "Incorrect approval in `TokenManager::_transfer` leads to DoS of all withdrawals". I will list the following lines here:

```diff
function withdraw(address _tokenAddress, TokenBalanceType _tokenBalanceType) external whenNotPaused {
    // previous code...

    if (_tokenAddress == wrappedNativeToken) {
        // ...
    } else {
-       _safe_transfer_from(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount); // <<< incorrect, lacks approvals
+       _transfer(_tokenAddress, capitalPoolAddr, _msgSender(), claimAbleAmount, capitalPoolAddr); // <<< correct, with appropriate approvals
    }
}
```

and `_transfer` function:

```diff
if (_from == _capitalPoolAddr && IERC20(_token).allowance(_from, address(this)) == 0x0) {
-   ICapitalPool(_capitalPoolAddr).approve(address(this)); // <<< incorrect
+   ICapitalPool(_capitalPoolAddr).approve(_token); // <<< correct
}
```

These bugs have been reported already as part of other issues, but in order to make this PoC work, they had to be fixed temporarily.

<br>

```javascript
function test_withdraw_vulnerability() public {
    // ~~~~~~~~~~~~~~~~~~~~ Setup ~~~~~~~~~~~~~~~~~~~~
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    uint256 MAKER_POINTS = 1000;
    uint256 TAKER_POINTS = 500;
    uint256 TOKEN_AMOUNT = 100 * 1e18;
    uint256 TOKENS_PER_POINT = 1e18; // 1 for 1
    uint256 INITIAL_BALANCE = 100_000 * 1e18;

    deal(address(mockUSDCToken), alice, INITIAL_BALANCE);
    deal(address(mockUSDCToken), bob, INITIAL_BALANCE);

    console.log(
        "- Alice's userTokenBalance before coming into the protocol: %18e",
        tokenManager.userTokenBalanceMap(alice, address(mockUSDCToken), TokenBalanceType.SalesRevenue)
    );
    console.log("- Alice's actual balance before coming into the protocol: %18e", mockUSDCToken.balanceOf(alice));
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 1) Create a maker offer ~~~~~~~~~~
    vm.startPrank(alice);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createMaker(
        CreateMakerParams(
            marketPlace,
            address(mockUSDCToken),
            MAKER_POINTS,
            TOKEN_AMOUNT,
            10_000 * 1.2, // 120% collateral rate
            10_000 * 0.03, // 3% trade tax
            OfferType.Ask,
            OfferSettleType.Turbo
        )
    );
    address offerAddr = GenerateAddress.generateOfferAddress(0);
    (,,, OfferStatus status,,,,,,,,,,) = preMarktes.offerInfoMap(offerAddr);
    require(status == OfferStatus.Virgin, "Maker offer not created successfully");
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 2) Create a taker order  ~~~~~~~~~~
    vm.startPrank(bob);
    mockUSDCToken.approve(address(tokenManager), type(uint256).max);
    preMarktes.createTaker(offerAddr, TAKER_POINTS);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 3) Settle the offer ~~~~~~~~~~
    vm.startPrank(user1); // user1 == admin
    systemConfig.updateMarket("Backpack", address(mockUSDCToken), TOKENS_PER_POINT, block.timestamp - 1, 3600);

    vm.startPrank(alice);
    deliveryPlace.settleAskMaker(offerAddr, TAKER_POINTS);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 4) Check initial balances ~~~~~~~~~~
    uint256 initialUserTokenBalance =
        tokenManager.userTokenBalanceMap(alice, address(mockUSDCToken), TokenBalanceType.SalesRevenue);
    console.log("> Initial Alice's userTokenBalance value: %18e", initialUserTokenBalance);

    uint256 initialActualTokenBalance = mockUSDCToken.balanceOf(alice);
    console.log("> Initial Alice's actual token balance: %18e", initialActualTokenBalance);
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 5) First withdrawal ~~~~~~~~~~
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);

    // Check balance after first withdrawal (should be 0)
    console.log(
        ">> Alice's userTokenBalance value after first withdrawal: %18e",
        tokenManager.userTokenBalanceMap(alice, address(mockUSDCToken), TokenBalanceType.SalesRevenue)
    );
    console.log(">> Actual token balance after first withdrawal: %18e", mockUSDCToken.balanceOf(alice));
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // ~~~~~~~~~~ 6) Perform second withdrawal (should fail or return 0, but it doesn't) ~~~~~~~~~~
    tokenManager.withdraw(address(mockUSDCToken), TokenBalanceType.SalesRevenue);

    // Check balance after second withdrawal (should still be 0, but it's not)
    uint256 userTokenBalanceAfterSecondWithdrawal =
        tokenManager.userTokenBalanceMap(alice, address(mockUSDCToken), TokenBalanceType.SalesRevenue);
    console.log(
        ">>> Alice's userTokenBalance value after second withdrawal: %18e", userTokenBalanceAfterSecondWithdrawal
    );
    console.log(">>> Actual token balance after second withdrawal: %18e", mockUSDCToken.balanceOf(alice));
    // ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

    // Assert that the balance is still the same as the initial balance, demonstrating the vulnerability
    assertEq(userTokenBalanceAfterSecondWithdrawal, initialUserTokenBalance);

    vm.stopPrank();
}
```

This test will yield the following logs:

```bash
- Alice's userTokenBalance before coming into the protocol: 0
- Alice's actual balance before coming into the protocol: 100000

> Initial Alice's userTokenBalance value: 170
> Initial Alice's actual token balance: 99380

>> Alice's userTokenBalance value after first withdrawal: 170
>> Actual token balance after first withdrawal: 99550

>>> Alice's userTokenBalance value after second withdrawal: 170
>>> Actual token balance after second withdrawal: 99720
```

which signifies that Alice is able to repeteadly withdraw tokens from the protocol, even though she should've withdrawn all of her tokens with her withdrawal.

## Tools Used

Foundry, Manual review

## Recommended Mitigation Steps

Update the `withdraw` function to reduce the user's balance after a successful withdrawal.

```diff
function withdraw(address _tokenAddress, TokenBalanceType _tokenBalanceType) external whenNotPaused {
    uint256 claimAbleAmount = userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType];

    if (claimAbleAmount == 0) {
        return;
    }

+   delete userTokenBalanceMap[_msgSender()][_tokenAddress][_tokenBalanceType];

    // the rest of the function...
}
```

## <a id='H-05'></a>[H-05] Unauthorized manipulation of referral rates in `SystemConfig::updateReferrerInfo` allows for increased fee allocation

## Summary

`SystemConfig::updateReferrerInfo` allows any address (except the referrer) to update referral rates. While there are safeguards to ensure the referrer's rate doesn't fall below the base rate, this still allows anyone else to potentially manipulate of fee distribution.

[SystemConfig.sol#L46-L48](https://github.com/Cyfrin/2024-08-tadle/blob/main/src/core/SystemConfig.sol#L46-L48)

```javascript
// src/core/SystemConfig.sol
function updateReferrerInfo(address _referrer, uint256 _referrerRate, uint256 _authorityRate) external {
@>  if (_msgSender() == _referrer) {
        revert InvalidReferrer(_referrer);
    }

    // rest of the function...
}
```

## Impact

An attacker could set up a separate account to update their own referral rates, potentially allocating up to 70% of the referral fees to themselves as the "authority". This could lead to unfair distribution of referral fees.

## Proof of Concept

Let's introduce Alice (attacker) and Bob (maker/offer owner) to the attack scenario.

Before Bob settles an offer, Alice could her referral fee to 100% and get all of the fees for herself whenever `Premarkets::_updateReferralBonus` executes.

## Tools Used

Manual review

## Recommended Mitigation Steps

While there's an argument that this will be resolved by enforcing access control on the client, there's no clear way to resolve the issue on the smart contract itself. Here are some ideas though:

- Enforce a maximum referrer rate (e.g., 70% of total rate) to ensure some fees always go to the authority or protocol.
- Implement access control on the `updateReferrerInfo` function, allowing only authorized addresses (e.g., admin or governance) to update referral rates.
- Consider implementing a timelock or a cooldown period for rate changes to prevent rapid manipulations and so that the offer maker could react to fee updates and inform authorized bodies of the update.

<br>

# Low Risk Findings

## <a id='L-01'></a>[L-01] Unclaimed platform fees accumulate indefinitely in maker info, potentially leading to loss of funds

## Summary

`PreMarkets::createTaker` accumulates platform fees for each maker's offer, but there is no implemented mechanism to claim or distribute these fees. This could lead to fees being locked in the contract indefinitely.

## Impact

Platform fees are continuously added to `makerInfo.platformFee` with each taker order, but there is no function to withdraw or distribute these fees. This could result in loss of revenue for the platform or intended fee recipients.

## Proof of Concept

[PreMarkets.sol#L263](https://github.com/igorroncevic/2024-08-tadle/blob/72c93f73a26ec7472868cb509e8b454286810223/src/core/PreMarkets.sol#L263)

```javascript
// src/core/PreMarkets.sol
function createTaker(address _offer, uint256 _points) external payable {
    // existing code...

    uint256 remainingPlatformFee = _updateReferralBonus(
        platformFee,
        depositAmount,
        stockAddr,
        makerInfo,
        referralInfo,
        tokenManager
    );

@>  makerInfo.platformFee = makerInfo.platformFee + remainingPlatformFee;

    // rest of the function...
}
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Implement a function to claim accumulated platform fees, accessible only by authorized addresses (e.g., protocol admin or fee recipient).

- If the platform is supposed to keep the platform fees, then the suggestion would be to track them in a separate variable and allow admins to withdraw them.
- If the maker is supposed to keep the platform fees, then something like this would be the suggested course of action:

```diff
function createTaker(address _offer, uint256 _points) external payable {
    // existing code...

    uint256 remainingPlatformFee = _updateReferralBonus(
        platformFee,
        depositAmount,
        stockAddr,
        makerInfo,
        referralInfo,
        tokenManager
    );

-   makerInfo.platformFee = makerInfo.platformFee + remainingPlatformFee;
+   tokenManager.addTokenBalance(
+       TokenBalanceType.PlatformFee, makerInfo.authority, makerInfo.tokenAddress, remainingPlatformFee
+   );

    // rest of the function...
}
```

```diff
enum TokenBalanceType {
    TaxIncome,
    ReferralBonus,
    SalesRevenue,
    RemainingCash,
    MakerRefund,
-   PointToken
+   PointToken,
+   PlatformFee
}
```
