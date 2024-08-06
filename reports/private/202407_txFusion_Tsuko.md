# Tsuko - Audit Report

Conducted by: ironside, July 2024

<br>

# Table of Contents

- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Scope](#audit-scope)
- [Executive Summary](#executive-summary)
  - [Findings Count](#findings-count)
  - [Findings Summary](#findings-summary)
- [Findings](#findings)
  - [High Findings](#high-findings)
  - [Medium Findings](#medium-findings)
  - [Low Findings](#low-findings)

<br>

# Protocol Summary

Tsuko is a Paymaster as a Service (PaaS) platform built on ZKsync's native Account Abstraction system. It enables developers to create and manage custom paymasters with specific restrictions, allowing for selective free transactions on the ZKsync network.

Key features of the Tsuko protocol include:

1. **Custom Paymaster Creation**: Users can deploy two types of paymasters:
   - **Sponsored Paymasters**: Cover gas fees for all qualifying transactions.
   - **ERC20 Paymasters**: Allow users to pay gas fees using specific ERC20 tokens.
2. **Flexible Restriction System**: Paymasters can be configured with various restrictions to control which transactions they sponsor:
   - **Contract Restrictions**: Limit sponsorship to specific smart contracts.
   - **User Restrictions**: Allow only whitelisted user addresses.
   - **Function Restrictions**: Sponsor only specific function calls within contracts.
   - **Function Parameter Restrictions**: Sponsor transactions based on specific parameter values.
   - **NFT Ownership Restrictions**: Limit sponsorship to holders of specific NFTs.
3. **Factory Contract**: A central factory contract manages the deployment and configuration of paymasters and restrictions.
4. **Oracle Integration**: Utilizes price feed oracles for accurate feature pricing (paying gas with ERC20 tokens, paying for deployment of new contracts etc).

Tsuko simplifies the process of integrating paymasters in ZKsync dApps, enabling developers to create more user-friendly experiences by removing the need for end-users to hold ETH for gas fees, subject to the specified restrictions.

<br>

# Disclaimer

The team conducting the audit makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

Some parts of the vulnerable code and recommendations to resolve those issues have been redacted, as per client's request.

<br>

# Risk Classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

### Impact

- High - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- Medium - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- Low - can lead to any kind of unexpected behavior with some of the protocol's functionalities that's not so critical.

### Likelihood

- High - attack path is possible with reasonable assumptions that mimic on-chain conditions, and the cost of the attack is relatively low compared to the amount of funds that can be stolen or lost.
- Medium - only a conditionally incentivized attack vector, but still relatively likely.
- Low - has too many or too unlikely assumptions or requires a significant stake by the attacker with little or no incentive.

<br>

# Audit Scope

- Review commit hash: `e1ca9f220a180e872d6cc32b966b9a563e0f18e3`
- Fixes commit hash: TBD

The following smart contracts were in the scope of the audit:

```bash
├── ContractRestriction.sol
├── ERC20Paymaster.sol
├── ERC1155Restriction.sol
├── Factory.sol
├── FunctionParamsRestriction.sol
├── FunctionRestriction.sol
├── NFT721Restriction.sol
├── Paymaster.sol
├── RestrictionFactory.sol
├── SponsoredPaymaster.sol
├── UserRestriction.sol
└── interfaces/
    ├── IRestriction.sol
    └── Oracles.sol
```

<br>

# Executive Summary

Over the course of the security review, Ironside engaged with txFusion to review Tsuko. In this period of time a total of 7 issues were uncovered.

| Protocol Name     | Tsuko                                        |
| ----------------- | -------------------------------------------- |
| **Repository**    | https://github.com/txfusion/txSync-contracts |
| **Date**          | July 2024                                    |
| **Protocol Type** | Paymaster as a Service                       |

## Findings Count

| Severity  | Amount |
| --------- | ------ |
| High      | 1      |
| Medium    | 2      |
| Low       | 4      |
| **Total** | 7      |

## Findings Summary

| ID                                                                                                                                | Title                                                                                                                | Severity | Status       |
| --------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | -------- | ------------ |
| [H-01](#h-01-lack-of-transaction-limitations-in-sponsoredpaymaster-can-lead-to-draining-of-projects-gas-funds)                    | Lack of transaction limitations in `SponsoredPaymaster` can lead to draining of project's gas funds                  | High     | Resolved     |
| [M-01](#m-01-single-point-of-failure-in-oracle-integration-leads-to-potential-dos-of-the-paymaster-or-overundercharging-of-users) | Single point of failure in oracle integration leads to potential DoS of the paymaster or over/undercharging of users | Medium   | Resolved     |
| [M-02](#m-02-hardcoded-decimals-for-zk-chains-base-token-can-lead-to-improper-price-calculations-for-required-gas)                | Hardcoded decimals for ZK chain's base token can lead to improper price calculations for required gas                | Medium   | Resolved     |
| [L-01](#l-01-insufficient-gas-estimation-buffer-may-lead-to-transaction-failures-in-volatile-network-condition)                   | Insufficient gas estimation buffer may lead to transaction failures in volatile network condition                    | Low      | Acknowledged |
| [L-02](#l-02-nft-restrictions-can-be-temporarily-disabled-by-setting-nftcontract-to-address0)                                     | NFT restrictions can be temporarily disabled by setting `nftContract` to `address(0)`                                | Low      | Acknowledged |
| [L-03](#l-03-lack-of-secure-ownership-transfer-mechanism-in-paymaster-and-restriction-contracts)                                  | Lack of secure ownership transfer mechanism in paymaster and restriction contracts                                   | Low      | Acknowledged |
| [L-04](#l-04-lack-of-event-emissions-for-critical-state-changes)                                                                  | Lack of event emissions for critical state changes                                                                   | Low      | Acknowledged |

<br>

# Findings

## High Findings

## [H-01] Lack of transaction limitations in `SponsoredPaymaster` can lead to draining of project's gas funds

### Severity

- **Impact**: High
- **Likelihood**: Medium

### Description

The `SponsoredPaymaster` contract lacks transaction limitations, allowing malicious users to potentially drain all of the project's gas funds by repeatedly sending transactions, for e.g. swapping minimal amounts on DEXes.

The vulnerability stems from the lack of rate limiting in the `SponsoredPaymaster` contract. While the `ERC20Paymaster` requires users to provide their ERC20 tokens in exchange for paying gas in the chain's native currency, `SponsoredPaymaster` doesn't require any input from the users, therefore, they could freely waste its gas.

```javascript
// SponsoredPaymaster.sol
function validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata _transaction)
    external
    payable
    override
    onlyBootloader
    onlyAllowedTransaction(_transaction)
    returns (bytes4 magic, bytes memory context)
    {
        // existing code...
    }
```

### Impact

Without proper rate limiting, a malicious actor could exploit the `SponsoredPaymaster` by continuously sending transactions, causing the paymaster to cover gas fees for each transaction. This could lead to:

- Rapid depletion of the paymaster's ETH balance.
- Denial of service for legitimate users as the paymaster runs out of funds.
- Financial losses for the project maintaining the paymaster.

## Recommended Mitigation Steps

1. Implement a rate limiter contract with customizable parameters, like so:

```javascript
abstract contract RateLimiter is Ownable {
    enum RateLimitType {
        Linear,
        Exponential
    }

    modifier rateLimit(address user_) {
        // 1) If there's no rate limit setting available, skip
        // 2) If there's a rate limit setting available
        // 2.1) Check if the function can be executed based on user's rate limit state
        // 2.2) Execute the function
        // 2.3) Update user's rate limit state

        // >>> Implementation redacted
    }

    function getRateLimitDelay(address user_) public view returns (uint256){
        // Calculate the rate limit delay based on user's number of transactions and global rate limit calculation type

        // >>> Implementation redacted
    }

    // >>> Rest of the contract redacted
}
```

2. Inherit the `RateLimiter` in the root `Paymaster` contract:

```diff
- contract Paymaster is IPaymaster {
+ contract Paymaster is IPaymaster, RateLimiter {
    // ... existing code ...
}
```

3. Use the `rateLimit(address)` modifier in `SponsoredPaymaster::validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata)` call:

```diff
function validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata _transaction)
    external
    payable
    override
    onlyBootloader
    onlyAllowedTransaction(_transaction)
+   rateLimit(address(uint160(_transaction.from)))
    returns (bytes4 magic, bytes memory context)
    {
        // ...
    }
```

<br>

## Medium Findings

## [M-01] Single point of failure in oracle integration leads to potential DoS of the paymaster or over/undercharging of users

### Severity

- **Impact**: High
- **Likelihood**: Low

### Description

`ERC20Paymaster`, `Factory` and `RestrictionFactory` rely on DIA's single oracle for price feeds. If this oracle fails or provides incorrect data, it could lead to a denial of service for the entire paymaster system. The current implementation lacks redundancy and error handling for oracle failures.

### Impact

If the single oracle fails, becomes compromised, or provides stale or incorrect data:

- Transactions may fail due to incorrect price calculations, causing a DoS for users trying to use the paymaster.
- Also, if the oracle responds but with bad/stale prices, the paymaster may incorrectly calculate the required ERC20 tokens for transactions, potentially leading to under or overcharging users.

## Recommended Mitigation Steps

1. Implement a price fetching library to provide redundancy and fallback options for price feeds:

```javascript
library PriceFetchLib {
    enum OracleType {
        Chainlink,
        DIA,
        Custom
    }

    struct AssetConfig {
        OracleConfig primaryOracle;
        OracleConfig[] backupOracles; // in case the primary oracle fails
    }

    struct OracleConfig {
        OracleType oracleType;
        string priceKey; // Used for DIA
        address oracleAddress;
        uint256 stalenessThreshold;
    }

    struct State {
        mapping(address => AssetConfig) assetConfigs;
    }

    function getPrice(address asset_) internal view returns (uint256) {
        // 1) Attempt to fetch from a primary oracle
        // 2) If primary oracle fails, attempt to fetch from backup oracles

        // >>> Implementation redacted
    }

    // >>> Rest of the contract redacted
}
```

2. Configure multiple oracles for each asset, including primary and backup oracles, in all contracts that need them, e.g.:

```diff
contract Factory is Initializable, UUPSUpgradeable, OwnableUpgradeable {
+   using PriceFetchLib for PriceFetchLib.State;

    // >>> Rest of the contract redacted

+   function updateBaseTokenAndConfig(
+       address _asset,
+       PriceFetchLib.OracleConfig memory _primaryOracle,
+       PriceFetchLib.OracleConfig[] memory _backupOracles
+   ) external onlyOwner {
+       asset = _asset;
+       PriceFetchLib.setAssetConfig(_asset, _primaryOracle, _backupOracles);
+   }
}
```

3. Implement proper error handling and logging for oracle failures to aid in monitoring and maintenance.

4. Consider implementing a circuit breaker mechanism that can pause the paymaster in case of persistent oracle issues.

5. Regularly monitor and update oracle configurations to ensure they remain reliable and up-to-date.

<br>

## [M-02] Hardcoded decimals for ZK chain's base token can lead to improper price calculations for required gas

### Severity

- **Impact**: High
- **Likelihood**: Low

### Description

The current implementation assumes a fixed number of decimals for the ZK chain's base token (ETH) to be 18. This assumption can lead to incorrect price calculations, especially if the system is deployed on a chain with a different base token or if the base token's decimals change in the future.

### Impact

If the actual number of decimals for the base token differs from the hardcoded value:

- Functions that handle gas price calculations will return incorrect amounts, which could lead to under or overcharging users for paymaster creation.
- It may cause transactions to fail due to insufficient gas, or waste user funds by overestimating required gas.

## Recommended Mitigation Steps

1. Implement the `SafeERC20Decimals` library to safely retrieve and manage token decimals for tokens who might not implement the `decimals()` function, so we track fallback values for them:

```javascript
library SafeERC20Decimals {
    function safeDecimals(address token_) internal view returns (uint8) {
        (bool success, bytes memory encodedDecimals) =
            address(token_).staticcall(abi.encodeWithSelector(IERC20Metadata.decimals.selector));
        if (success && encodedDecimals.length >= 32) {
            uint256 returnedDecimals = abi.decode(encodedDecimals, (uint256));
            if (returnedDecimals > type(uint8).max) {
                revert SafeERC20Decimals__InvalidDecimals(token_, returnedDecimals);
            }

            return uint8(returnedDecimals);
        }

        uint8 _defaultDecimals = defaultDecimals[token_];
        return _defaultDecimals != 0 ? _defaultDecimals : 18;
    }

    // >>> Rest of the contract redacted
}
```

2. Add a function to update the base token address and its decimals if needed:

```diff
contract Factory is Initializable, UUPSUpgradeable, OwnableUpgradeable {
+   using SafeERC20Decimals for SafeERC20Decimals.State;

    // existing code...

+   function updateDefaultDecimals(address token_, uint8 decimals_) external onlyOwner {
+       SafeERC20Decimals.updateDefaultDecimals(token_, decimals_);
+   }
}
```

3. Fetch base token's decimals each time when performing a calculation which includes the price of the base token:

```diff
function calculatePaymasterCreationRequiredETH() public view returns (uint256) {
    // >>> Implementation redacted
}
```

4. Ensure that the base token and its decimals are correctly set during contract deployment and verified after deployment.

5. Add events to log any changes to the base token or its decimals for better traceability.

6. Implement a sanity check function that verifies the consistency between the stored base token decimals and the actual token contract's decimals, which can be run periodically or before critical operations.

<br>

## Low Findings

## [L-01] Insufficient gas estimation buffer may lead to transaction failures in volatile network condition

### Severity

- **Impact**: Medium
- **Likelihood**: Low

### Description

The current implementation of gas estimation in the `ERC20Paymaster`/`SponsoredPaymaster` contract does not include a buffer to account for potential fluctuations in network conditions. This could lead to transaction failures during periods of high network congestion or rapid gas price changes.

```javascript
function validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata _transaction)
    external
    payable
    override
    onlyBootloader
    onlyAllowedTransaction(_transaction)
    returns (bytes4 magic, bytes memory context)
{
    // existing code...

    if (paymasterInputSelector == IPaymasterFlow.general.selector)
@>      uint256 requiredETH = _transaction.gasLimit * _transaction.maxFeePerGas;

    // existing code...
}
```

### Impact

The lack of a gas estimation buffer could result in:

1. Failed transactions due to insufficient gas, especially during periods of network congestion.
2. Poor user experience as transactions may need to be resubmitted with higher gas limits.
3. Increased operational overhead for users and developers who need to manually adjust gas parameters.

## Recommended Mitigation Steps

1. Implement a configurable gas multiplier to provide a buffer for gas estimation:

```diff
function validateAndPayForPaymasterTransaction(bytes32, bytes32, Transaction calldata transaction)
    external
    payable
    override
    onlyBootloader
    returns (bytes4 magic, bytes memory context)
{
    // ... existing code ...
    if (paymasterInputSelector == IPaymasterFlow.general.selector) {
-       uint256 requiredETH = _transaction.gasLimit * _transaction.maxFeePerGas;
+       // e.g. if 2 decimals, then gasMultiplier = 120 => 120%
+       uint256 requiredBaseToken = _transaction.gasLimit * transaction.maxFeePerGas * gasMultiplier / 100;
    // ... rest of the function ...
}
```

2. Make the gas multiplier configurable by the contract owner to allow for adjustments based on network conditions.
3. Consider implementing an automatic adjustment mechanism that updates the gas multiplier based on recent transaction success rates or network congestion metrics.

## [L-02] NFT restrictions can be temporarily disabled by setting `nftContract` to `address(0)`

### Severity

- **Impact**: Medium
- **Likelihood**: Low

### Description

`NFT721Restriction`/`NFT1155Restriction` contract allows the owner to set the NFT contract address through the constructor and `updateNftContract`/`updateContractAddress` function. However, there is no check to prevent setting this address to the zero address (`address(0)`). If the `nftContract`/`contractAddress` is set to `address(0)`, it will cause the `canPayForTransaction` function to fail, effectively disabling the restriction temporarily.

### Impact

If the `nftContract` is set to `address(0)`, either accidentally or maliciously by the owner:

- The `canPayForTransaction` function will always revert due to a call to the zero address.
- This will effectively disable the `NFT721Restriction`/`NFT1155Restriction`, preventing any transactions from being processed through paymasters using this restriction.

## Recommended Mitigation Steps

Add a zero-address check in the constructors and `updateNftContract`/`updateContractAddress` function:

## [L-03] Lack of secure ownership transfer mechanism in paymaster and restriction contracts

### Severity

- **Impact**: Medium
- **Likelihood**: Low

### Description

The current implementation of ownership transfer in the paymaster and restriction contracts uses the basic `Ownable` pattern from OpenZeppelin. This pattern allows for immediate transfer of ownership, which can be risky if the new owner's address is incorrectly specified or unintended.

### Impact

The lack of a secure ownership transfer mechanism could result in:

1. Accidental transfer of ownership to an incorrect address, potentially locking the contract permanently.
2. No way to recover from an incorrect ownership transfer.
3. Loss of paymaster funds, since the new owner can withdraw all ETH/ERC20 tokens

While this doesn't directly impact the contract's day-to-day functionality, it introduces unnecessary risk during ownership changes.

Similar inheritance can be found in other restriction contracts.

## Recommended Mitigation Steps

Replace `OwnableUpgradeable` with `Ownable2StepUpgradeable` from OpenZeppelin in all relevant contracts.

```diff
- import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
+ import {Ownable2StepUpgradeable} from "@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol";

- contract Factory is Initializable, UUPSUpgradeable, OwnableUpgradeable {
+ contract Factory is Initializable, UUPSUpgradeable, Ownable2StepUpgradeable {
    // ... existing code ...
}
```

`Ownable2StepUpgradeable` introduces a two-step ownership transfer process, where the initial owner initiates a transfer to the new `pendingOwner` by calling `transferOwnership`. `pendingOwner` can only become an actual `owner` by calling `acceptOwnership`, therefore, no accidental ownership transfers can occur.

## [L-04] Lack of event emissions for critical state changes

Several important functions that modify the contract state do not emit events. This includes setter functions and other critical operations. Emitting events for significant state changes is crucial for off-chain monitoring and transparency.

Code locations:

- [NFT721Restriction.sol#L19 - updateNftContract](https://github.com/txfusion/txSync-contracts/blob/e1ca9f220a180e872d6cc32b966b9a563e0f18e3/contracts/NFT721Restriction.sol#L19)
- [ERC1155Restriction.sol#L87 - updateContractAddress](https://github.com/txfusion/txSync-contracts/blob/e1ca9f220a180e872d6cc32b966b9a563e0f18e3/contracts/ERC1155Restriction.sol#L87)
- [Factory.sol#L115 - updatePaymasterCreationPriceInUSD](https://github.com/txfusion/txSync-contracts/blob/e1ca9f220a180e872d6cc32b966b9a563e0f18e3/contracts/Factory.sol#L115)
- [RestrictionFactory.sol#L25 - updateRestrictionCreationPriceInUSD](https://github.com/txfusion/txSync-contracts/blob/e1ca9f220a180e872d6cc32b966b9a563e0f18e3/contracts/RestrictionFactory.sol#L252)
