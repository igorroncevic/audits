# Report - Winnables Raffles

## Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [[H-01] Failure to update `_lockedETH` in `WinnablesTicketManager::refundPlayers` function leads to potential fund locking](#H-01)
- Medium Risk Findings
  - [[M-01] Malicious admin will not be able to have their admin role removed, compromising protocol as a whole](#M-01)

<br>

## <a id='contest-summary'></a>Contest Summary

### Sponsor: Winnables Raffles

### Dates: Aug 16th, 2024 - Aug 20th, 2024

Winnables is a cutting-edge decentralized raffle platform (transparent and fair) that offers exciting prizes, including NFTs and crypto, all on the Ethereum network.

[See more contest details here](https://audits.sherlock.xyz/contests/516)

## <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a> [H-01] Failure to update `_lockedETH` in `WinnablesTicketManager::refundPlayers` function leads to potential fund locking

## Summary

`WinnablesTicketManager::refundPlayers` refunds players for canceled raffles but fails to update the `_lockedETH` variable. This oversight can lead to a discrepancy between the actual contract balance and the tracked locked ETH, potentially causing issues with future withdrawals.

## Impact

`WinnablesTicketManager::withdrawETH` relies on the `_lockedETH` variable to determine how much ETH can be withdrawn. If `_lockedETH` is not decreased during refunds, it will remain artificially high, preventing the withdrawal of legitimately available funds.

## Proof of Concept

[WinnablesTicketManager.sol#L223-L224](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/WinnablesTicketManager.sol#L223-L224)

This can be exploited or cause issues in the following scenario:

- A raffle is canceled, and players are eligible for refunds.
- `WinnablesTicketManager::refundPlayers` is called multiple times, refunding ETH to players.
- The contract's actual ETH balance decreases, but `_lockedETH` remains unchanged.
- When trying to withdraw ETH using the `WinnablesTicketManager::withdrawETH` function, the available balance calculation `address(this).balance - _lockedETH` may result in an underflow or prevent legitimate withdrawals.

## Tools Used

Manual review

## Recommended Mitigation Steps

Update the `WinnablesTicketManager::refundPlayers` function to decrease `_lockedETH` when refunding players. Additionally, consider adding a safety check to ensure `_lockedETH` doesn't underflow:

```diff
function refundPlayers(uint256 raffleId, address[] calldata players) external {
    // ... existing code ...

    for (uint256 i = 0; i < players.length;) {
        // ... existing code ...

        uint256 amountToSend = (participation & type(uint128).max);
+       require(_lockedETH >= amountToSend, "Insufficient locked ETH");
        _sendETH(amountToSend, player);
+       _lockedETH -= amountToSend;

        // ... rest of the function ...
    }
}
```

---

# Medium Risk Findings

## <a id='M-01'></a>[M-01] Malicious admin will not be able to have their admin role removed, compromising protocol as a whole

## Summary

`Roles::_setRole` always sets the role bit to 1, regardless of the `status` parameter. This means that once an admin role (role 0) is granted, it cannot be revoked, creating a significant security risk if an admin account is compromised.

## Impact

If an admin account is compromised, there's no way to revoke their privileges, potentially leading to:

1. Unauthorized actions being performed indefinitely.
2. Inability to remove malicious actors from the admin role.

This could result in severe consequences such as unauthorized fund transfers, manipulation of raffle parameters, or other malicious actions that could compromise the integrity and security of the entire protocol.

## Proof of Concept

[Roles.sol#L31](https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/Roles.sol#L31)

```javascript
// Roles.sol
function _setRole(address user, uint8 role, bool status) internal virtual {
    uint256 roles = uint256(_addressRoles[user]);
@>  _addressRoles[user] = bytes32(roles | (1 << role));
    emit RoleUpdated(user, role, status);
}
```

This function always sets the role bit to 1 using the bitwise OR operation, regardless of the `status` parameter. As a result, once a role is granted, it cannot be revoked.

To demonstrate:

1. Admin grants role 0 to address A: `_setRole(A, 0, true)`
2. Later, trying to revoke role 0 from A: `_setRole(A, 0, false)`
3. The role is not actually revoked due to the implementation.

## Tools Used

Manual review

## Recommended Mitigation Steps

Modify the `Roles::_setRole` function to properly handle both granting and revoking roles based on the `status` parameter:

```diff
function setRole(address user, uint8 role, bool status) internal virtual {
    uint256 roles = uint256(_addressRoles[user]);

+   if (status) {
+       roles |= (1 << role);
+   } else {
+       roles &= ~(1 << role);
+   }
+
+   _addressRoles[user] = bytes32(roles);
-   _addressRoles[user] = bytes32(roles | (1 << role));
    emit RoleUpdated(user, role, status);
}
```

This modification ensures that roles can be both granted and revoked, aligning with the function's intended behavior and improving the contract's security and flexibility.

<br>
