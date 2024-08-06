# Visor - Audit Report

Conducted by: ironside, August 2024

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

Visor is a blockchain automation tool designed specifically for the ZKsync ecosystem and its native Account Abstraction protocol. It revolutionizes the way users interact with blockchain technology by offering seamless transaction automation while maintaining complete control over their funds.

### Key Features and Benefits

1. **Self-Custody Automation**: Visor allows users to automate blockchain transactions without surrendering control of their private keys. This solves a critical problem in the crypto space where automation often requires trusting third-party custodians.

2. **Flexible Task Automation**: Users can set up a wide range of automated tasks, from simple recurring transactions to complex, condition-based operations. This feature saves time and reduces the risk of manual errors in repetitive blockchain activities.

3. **Scheduled Transactions**: Plan and execute transactions at precise intervals (hourly, daily, monthly), enabling better financial planning and automated portfolio management.

4. **Intelligent Fund Management**: Visor's automatic top-up feature ensures that transactions never fail due to insufficient funds, solving a common pain point in blockchain operations.

### Core Components

Visor's architecture consists of several key components, each designed to enhance user experience and functionality:

1. **Smart Account**: The central hub of user operations, managing plugins and access control.

2. **Plugin System**: A modular approach allowing for extensible functionality. Key plugins include:

   - **Recurring Plugin**: Enables setting up and managing recurring tasks.
   - **Observable Plugin**: Allows for condition-based task execution, such as actions triggered by specific token balances.

3. **Wallet Access Control**: Provides fine-grained permissions, allowing users to delegate specific actions to different addresses without compromising overall account security.

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

- Review commit hash: `4b35213025349f235ffd6720c98f9252884d737b`
- Fixes commit hash: TBD

The following smart contracts were in the scope of the audit:

```bash
├── Dependencies.sol
├── SmartAccount.sol
├── SmartAccountFactory.sol
├── TokenReceiver.sol
├── extensions
│   ├── AbstractBatchTransactions.sol
│   ├── BatchTransactions.sol
│   └── WalletAccess.sol
├── helpers
│   └── BootloaderChecker.sol
├── interfaces
│   ├── IBatchTransactions.sol
│   ├── IObservablePlugin.sol
│   ├── IPlugin.sol
│   ├── IPluginFactory.sol
│   ├── IPluginManager.sol
│   ├── IRecurringPlugin.sol
│   ├── ISmartAccountFactory.sol
│   ├── ISmartAccountInitialize.sol
│   └── IWalletAccess.sol
├── libraries
│   ├── Address.sol
│   ├── AssemblyCall.sol
│   ├── Errors.sol
│   └── SignatureDecoder.sol
└── plugins
    ├── ObservablePlugin.sol
    ├── PluginFactory.sol
    ├── PluginManager.sol
    └── RecurringPlugin.sol
```

<br>

# Executive Summary

Over the course of the security review, Ironside engaged with txFusion to review Visor. In this period of time a total of 6 issues were uncovered.

| Protocol Name     | Visor                                       |
| ----------------- | ------------------------------------------- |
| **Repository**    | https://github.com/txfusion/visor-contracts |
| **Date**          | August 2024                                 |
| **Protocol Type** | Modular Smart Accounts                      |

## Findings Count

| Severity  | Amount |
| --------- | ------ |
| High      | 1      |
| Medium    | 1      |
| Low       | 4      |
| **Total** | 6      |

## Findings Summary

| ID                                                                                                                         | Title                                                                                                              | Severity | Status       |
| -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | -------- | ------------ |
| [H-01](#h-01-unauthorized-smart-account-deployment-allows-malicious-permission-setting-for-an-unsuspecting-user)           | Unauthorized smart account deployment allows malicious permission setting for an unsuspecting user                 | High     | Resolved     |
| [M-01](#m-01-unrestricted-pluginfactory-address-in-pluginmanagerdeployplugin-allows-potential-malicious-plugin-deployment) | Unrestricted `PluginFactory` address in `PluginManager::deployPlugin` allows potential malicious plugin deployment | Medium   | Resolved     |
| [L-01](#l-01-missing-zero-check-for-maxnumberofexecutions-in-recurringplugin-could-lead-to-task-execution-failure)         | Missing zero check for `maxNumberOfExecutions` in `RecurringPlugin` could lead to task execution failure           | Low      | Acknowledged |
| [L-02](#l-02-lack-of-enable-functionality-for-recurring-tasks-in-recurringplugin)                                          | Lack of `enable` functionality for recurring tasks in `RecurringPlugin`                                            | Low      | Acknowledged |
| [L-03](#l-03-lack-of-zero-address-check-for-owner-in-observableplugin-initialization)                                      | Lack of zero address check for `owner` in `ObservablePlugin` initialization                                        | Low      | Resolved     |
| [L-04](#l-04-potential-zero-value-transfers-in-observableplugin-due-to-inclusive-threshold-check)                          | Potential zero value transfers in `ObservablePlugin` due to inclusive threshold check                              | Low      | Resolved     |

<br>

# Findings

## High Findings

## [H-01] Unauthorized smart account deployment allows malicious permission setting for an unsuspecting user

### Summary

`SmartAccountFactory::deployAccount` allows anyone to deploy a smart account for any address and set initial permissions, potentially granting themselves full access to the account without the owner's knowledge or consent.

### Impact

- Once the smart account is deployed for a certain user, they cannot redeploy it again.
- This would prompt the unsuspecting user into thinking that they've already went through the process of deploying an account for themselves.
- Finally, this allows the malicious actor to drain the user once they've funded their smart account.

### Proof of Concept

1. Bob deploys a smart account for random arbitrary addresses.
2. Bob sets himself as a signer with full permissions.
3. Bob waits for the legitimate owners to fund their accounts.
4. Bob drains the account using their pre-set permissions.

```javascript
function deployAccount(bytes32 salt, address owner, IWalletAccess.AccessRequest calldata signerPermission)
    external
    returns (address accountAddress)
{
@>  if (deployedAccount[owner] != address(0)) revert AccountAlreadyExists();

    accountAddress = _deployProxy(salt);
@>  _initializeAccount(accountAddress, owner, signerPermission); // <<< arbitrary permissions set by anyone

    // the rest of the function...
}
```

### Recommended Mitigation Steps

To mitigate this issue, implement the following changes in the SmartAccount contract's initialize function:

1. Reset permissions if the caller is not the owner:

```diff
function deployAccount(bytes32 salt, address owner, IWalletAccess.AccessRequest calldata signerPermission)
    external
    returns (address accountAddress)
{
    if (deployedAccount[owner] != address(0)) revert AccountAlreadyExists();

    accountAddress = deployProxy(salt);

    IWalletAccess.AccessRequest memory _safePermission;
+   // Set permissions only if the caller is the owner
+   if (msg.sender == owner) {
+       _safePermission = signerPermission;
+   }

-   _initializeAccount(accountAddress, owner, signerPermission);
+   _initializeAccount(accountAddress, owner, _safePermission);

    // the rest of the function...
}
```

This change ensures that if someone other than the owner deploys the account, they cannot set malicious permissions. The legitimate owner will need to set up permissions after claiming their account.

<br>

## Medium Findings

## [M-01] Unrestricted `PluginFactory` address in `PluginManager::deployPlugin` allows potential malicious plugin deployment

### Summary

`PluginManager::deployPlugin` allows the use of any address as a plugin factory, potentially enabling the deployment of malicious plugins if a user provides an incorrect or compromised factory address.

### Impact

If a user inadvertently provides a malicious plugin factory address, it could lead to:

1. Deployment of malicious plugins with full access to the smart account.
2. Potential loss of funds or unauthorized actions performed on behalf of the user.
3. Compromise of the entire smart account's security model.

### Proof of Concept

An attacker could exploit this by:

1. Creating a malicious plugin factory contract.
2. Tricking a user into using this malicious factory address when calling `deployPlugin`.
3. Deploying a malicious plugin that could perform unauthorized actions.

```javascript
function deployPlugin(address _factoryAddress, bytes calldata _data) external onlyThis {
@>  address pluginAddress = IPluginFactory(_factoryAddress).deployPlugin(_data);

    _plugins.add(pluginAddress);

    emit PluginDeployed(pluginAddress);
}
```

### Recommended Mitigation Steps

1. Implement a `PluginFactoryRegistry` contract to whitelist approved plugin factories
2. Since the `SmartAccount` (and consequently `PluginManager`) isn't upgradeable, we can make the `SmartAccountFactory` a central entity, which we can leverage to query the newly created `PluginFactoryRegistry` in order to check its whitelist status:

```diff
contract SmartAccountFactory is Initializable, OwnableUpgradeable, UUPSUpgradeable, ISmartAccountFactory {
    // previous code...

+   PluginFactoryRegistry public factoryRegistry;

+   function setFactoryRegistry(address _registry) external onlyOwner {
+       factoryRegistry = PluginFactoryRegistry(_registry);
+   }

+   function isPluginFactoryWhitelisted(address _pluginFactory) external view {
+       return factoryRegistry.isWhitelisted(_pluginFactory);
+   }

    // the rest of the code...
}
```

```diff
contract PluginManager is IPluginManager {
    // previous code...

+   SmartAccountFactory public smartAccountFactory; // TODO: initialize

    function deployPlugin(address factoryAddress, bytes calldata _data) external onlyThis {
+       require(smartAccountFactory.isPluginFactoryWhitelisted(factoryAddress), "Factory not whitelisted");

        address pluginAddress = IPluginFactory(factoryAddress).deployPlugin(_data);
        plugins.add(pluginAddress);

        emit PluginDeployed(pluginAddress);
    }

    // rest of the contract...
}
```

These changes ensure that only approved plugin factories can be used, significantly reducing the risk of malicious plugin deployment.

<br>

## Low Findings

## [L-01] Missing zero check for `maxNumberOfExecutions` in `RecurringPlugin` could lead to task execution failure

### Description

The `RecurringPlugin::_decodeAndValidateTaskData` lacks a zero check for the `maxNumberOfExecutions` parameter. This oversight could lead to the creation of tasks that can never be executed, potentially causing confusion and wasted gas for users.

### Impact

If a user creates a recurring task with `maxNumberOfExecutions` set to 0, the task will be created but can never be executed. This is because the `RecurringPlugin::_shouldDeactivateTask` function checks if the number of executions has reached the maximum:

```javascript
function _shouldDeactivateTask(RecurringTask storage task) private view returns (bool) {
    return task.maxNumberOfExecutions > 0 && task.numberOfExecutions == task.maxNumberOfExecutions;
}
```

When `maxNumberOfExecutions` is 0, this condition will always be true from the start, immediately deactivating the task.

This could lead to:

- Wasted gas on task creation that can never be executed.
- Confusion for users who expect their tasks to run but find them immediately deactivated.
- Potential issues in any dependent systems or UI that assume all created tasks are executable.

### Recommended Mitigation Steps

Add a zero check for `maxNumberOfExecutions` in the `RecurringPlugin::_validateTaskCreation` function:

```diff
-function _validateTaskCreation(address target, uint256 value, uint256 timeInterval, bytes memory callData)
+function _validateTaskCreation(address target, uint256 value, uint256 timeInterval, uint256 maxNumberOfExecutions, bytes memory callData)
    internal
    pure
{
    if (value == 0 && callData.length == 0) {
        revert InvalidValueOrCalldata();
    }
    if (target == address(0)) revert AddressZero();
    if (timeInterval == 0) revert InvalidTimeInterval();
+   if (maxNumberOfExecutions == 0) revert InvalidMaxNumberOfExecutions();
}
```

## [L-02] Lack of `enable` functionality for recurring tasks in `RecurringPlugin`

### Description

`RecurringPlugin` allows for disabling tasks but lacks functionality to re-enable them. This forces users to recreate tasks if they want to resume a previously disabled task, leading to unnecessary gas costs and potential loss of historical data.

### Impact

The absence of a re-enable function has the following impacts:

- Increased gas costs: Users need to recreate tasks from scratch, incurring additional gas fees.
- Loss of historical data: Recreating a task resets its execution history, which might be valuable for tracking or auditing purposes.
- User inconvenience: Users have to remember and re-input all the task details to recreate a disabled task.
- Potential for errors: Manual recreation of tasks increases the risk of input errors.

While these impacts are not critical to the core functionality of the contract, they represent a significant inconvenience and potential source of inefficiency for users.

### Recommended Mitigation Steps

1. Implement an enable function that allows the owner to re-activate a disabled task:

```diff
+function enable(bytes calldata _data) external onlyOwner {
+    uint256 id = abi.decode(_data, (uint256));
+    if (tasks[id].isActive) {
+        revert TaskAlreadyActive();
+    }
+    tasks[id].isActive = true;

+    emit RecurringTaskEnabled(id);
+}
```

2. Consider adding a bulk enable function similar to the existing `disableAll` function:

```diff
+function enableAll() external onlyOwner {
+    for (uint256 i = 0; i < tasks.length; i++) {
+        if (!tasks[i].isActive) {
+            tasks[i].isActive = true;
+            emit RecurringTaskEnabled(i);
+        }
+    }
+}
```

## [L-03] Lack of zero address check for `owner` in `ObservablePlugin` initialization

### Description

`ObservablePlugin::initialize` does not check if the provided `owner` address is the zero address. This could potentially lead to a useless plugin if initialized with the zero address as the owner.

### Impact

If the `ObservablePlugin` is initialized with the zero address as the owner, it would result in a plugin that cannot be controlled or managed. This is because:

- The zero address cannot perform transactions, so no owner-only functions could be called.
- Ownership transfer would be impossible, effectively locking the plugin in this state permanently.

While this scenario is unlikely in normal operations, especially if the initialization is done through a factory contract, it still represents a potential risk if manual deployment or initialization is possible.

### Recommended Mitigation Steps

1. Add a zero address check in the initialize function:

```diff
function initialize(bytes calldata _data) external initializer {
    (address owner,) = abi.decode(_data, (address, bytes));
+   require(owner != address(0), "Owner cannot be zero address");

    __Ownable_init();
    _transferOwnership(owner);
    __UUPSUpgradeable_init();
}
```

2. Consider using OpenZeppelin's `Ownable2Step` instead of `Ownable`. This provides an additional layer of security for ownership transfers by requiring the new owner to accept the transfer. Here's how you could implement it:

```diff
-import {OwnableUpgradeable} from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
+import {Ownable2StepUpgradeable} from "@openzeppelin/contracts-upgradeable/access/Ownable2StepUpgradeable.sol";

-contract ObservablePlugin is Initializable, OwnableUpgradeable, UUPSUpgradeable, IObservablePlugin, IPlugin {
+contract ObservablePlugin is Initializable, Ownable2StepUpgradeable, UUPSUpgradeable, IObservablePlugin, IPlugin {
    // ... other code ...

    function initialize(bytes calldata _data) external initializer {
        (address owner,) = abi.decode(_data, (address, bytes));
        require(owner != address(0), "Owner cannot be zero address");

-       __Ownable_init();
+       __Ownable2Step_init();
        _transferOwnership(owner);
        __UUPSUpgradeable_init();
    }

    // ... other code ...
}
```

## [L-04] Potential zero value transfers in `ObservablePlugin` due to inclusive threshold check

### Description

`ObservablePlugin::_isValid` uses an inclusive comparison (`<=`) when checking if a task is valid. This can lead to unnecessary zero-value transfers when the beneficiary's balance exactly matches the threshold.

### Impact

This issue can result in:

- Increased number of recorded executions for tasks, potentially reaching execution limits prematurely.
- Unnecessary gas consumption for executing zero-value transfers.
- Potential confusion for users or monitoring systems that may interpret these as actual transfers.

### Recommended Mitigation Steps

1. Replace the `<=` comparisons with `<` in the `ObservablePlugin::_isValid` function:

```diff
function _isValid(ObservableTask memory task) internal view returns (bool) {
    if (task.erc20Token != address(0)) {
        uint256 currentTokenBalance = IERC20(task.erc20Token).balanceOf(task.beneficiary);
-       return currentTokenBalance <= task.threshold;
+       return currentTokenBalance < task.threshold;
    } else {
-       return address(task.beneficiary).balance <= task.threshold;
+       return address(task.beneficiary).balance < task.threshold;
    }
}
```

2. Involve this non-inclusive validation in other functions, to prevent unnecessary execution:

```diff
function execute(bytes calldata _data) external onlyOwner {
    uint256 id = abi.decode(_data, (uint256));
    ObservableTask storage task = tasks[id];
-   tasks[id].noOfExecutions++;

+   if (!_isValid(task)) {
+       revert InvalidTaskConditions();
+   }

+   task.noOfExecutions++;

    // note value not used/passed to _executeTransaction, Ether is sent from Smart Account
    (bytes memory encodedData, uint256 value) = _prepareTransferData(task);

    // rest of the code...
}
```
