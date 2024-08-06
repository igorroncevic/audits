# First Flight #16: Mafia Takedown - Findings Report

# Table of contents

- [Contest Summary](#contest-summary)
- [Results Summary](#results-summary)
- High Risk Findings
  - [H-01. Incorrect dependency configuration in `Laundrette::configureDependencies` renders `MoneyShelf` unusable](#H-01)
  - [H-02. Arbitrary `account` parameter in `MoneyShelf::depositUSDC` allows gang members to steal USDC from each other](#H-02)
- Medium Risk Findings
  - [M-01. Failure to transfer USDC during emergency migration leaves funds vulnerable in `MoneyShelf`](#M-01)
  - [M-02. Unauthorized role revocation in `Laundrette::quitTheGang` allows any gang member to remove others from the gang](#M-02)

## <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #16: Mafia Takedown

### Dates: May 23rd, 2024 - May 30th, 2024

An undercover AMA agent (anti-mafia agency) discovered a protocol used by the Mafia. In several days, a raid will be conducted by the police and we need as much information as possible about this protocol to prevent any problems. But the AMA doesn’t have any web3 experts on their team. Hawks, they need your help! Find flaws in this protocol and send us your findings.

[See more contest details here](https://www.codehawks.com/contests/clwgiehgu00119zwn2xx92ay8)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 2
- Medium: 2
- Low: 0

<br>

# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect dependency configuration in `Laundrette::configureDependencies` renders `MoneyShelf` unusable

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/main/src/policies/Laundrette.sol#L26

## Summary

`Laundrette::configureDependencies` incorrectly uses the same index for two different dependencies, causing the first dependency to be overwritten. This results in only one dependency being recorded, which can lead to improper configuration and potential malfunction of the contract.

## Vulnerability Details

`Laundrette::configureDependencies` is designed to set up the dependencies for the `Laundrette` contract. However, it uses the same index (`dependencies[0]`) for both `MONEY` and `WEAPN` dependencies, causing the first dependency to be overwritten by the second. The relevant code is as follows:

```javascript
function configureDependencies() external override onlyKernel returns (Keycode[] memory dependencies) {
    dependencies = new Keycode[](2);

    dependencies[0] = toKeycode("MONEY");
    moneyShelf = MoneyShelf(getModuleAddress(toKeycode("MONEY")));

    dependencies[0] = toKeycode("WEAPN");
    weaponShelf = WeaponShelf(getModuleAddress(toKeycode("WEAPN")));
}
```

## Impact

This vulnerability leads to only one dependency being recorded instead of two, which results in improper configuration of the protocol, where the `Laundrette` contract may not properly configure its dependencies and the protocol's overall access control.

## Tools Used

Manual code review

## Recommendations

To resolve this issue, the `Laundrette::configureDependencies` function should be corrected to use separate indexes for each dependency. Here is the updated implementation:

```diff
function configureDependencies() external override onlyKernel returns (Keycode[] memory dependencies) {
    dependencies = new Keycode[](2);

    dependencies[0] = toKeycode("MONEY");
    moneyShelf = MoneyShelf(getModuleAddress(toKeycode("MONEY")));

-   dependencies[0] = toKeycode("WEAPN");
+   dependencies[1] = toKeycode("WEAPN");
    weaponShelf = WeaponShelf(getModuleAddress(toKeycode("WEAPN")));
}
```

## <a id='H-02'></a>H-02. Arbitrary `account` parameter in `MoneyShelf::depositUSDC` allows gang members to steal USDC from each other

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/main/src/modules/MoneyShelf.sol#L27

## Summary

`MoneyShelf::depositUSDC` allows arbitrary addresses to be specified for the account parameter. This could potentially enable gang members to exploit the function and steal USDC from each other by specifying another gang member's address as the `account` from which to transfer funds.

## Vulnerability Details

`MoneyShelf::depositUSDC` allows any address to call the function and specify an arbitrary `account` address from which USDC should be transferred. This can be exploited by malicious actors to transfer USDC from an account they do not own, as long as that account has approved the `MoneyShelf` contract to spend its USDC.

```javascript
function depositUSDC(address account, address to, uint256 amount) external {
    deposit(to, amount);
@>  usdc.transferFrom(account, address(this), amount);
    crimeMoney.mint(to, amount);
}
```

Steps to Reproduce

- Gang member A approves the `MoneyShelf` contract to spend their USDC.
- Gang member B calls the `depositUSDC` function, specifying Gang member A's address as the account parameter.
- USDC is transferred from Gang member A's account to the `MoneyShelf` contract, without Gang Member A's consent, and receives `CrimeMoney` to their account
- Gang member B can call `withdrawUSDC` and retrieve locked USDC from the contract

## Impact

Malicious gang members can exploit this to drain USDC from other members' accounts, as long as the victim has approved the `MoneyShelf` contract to spend their USDC.

## Tools Used

Manual code review

## Recommendations

`depositUSDC` function should be modified to ensure that the account parameter is always the caller (`msg.sender`). This will ensure that gang members can only deposit their own USDC and cannot specify arbitrary addresses. Here is the updated implementation:

```diff
-function depositUSDC(address account, address to, uint256 amount) external {
+function depositUSDC(address to, uint256 amount) external {
    deposit(to, amount);
-   usdc.transferFrom(account, address(this), amount);
+   usdc.transferFrom(msg.sender, address(this), amount);
    crimeMoney.mint(to, amount);
}
```

<br>

# Medium Risk Findings

## <a id='M-01'></a>M-01. Failure to transfer USDC during emergency migration leaves funds vulnerable in `MoneyShelf`

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/main/script/EmergencyMigration.s.sol#L29-L32

## Summary

`EmergencyMigration::migrate` function correctly upgrades the `MoneyShelf` module to `MoneyVault`, however, it fails to move the USDC tokens from the `MoneyShelf` to the `MoneyVault`. This omission can lead to a situation where the funds remain in the old contract, potentially exposed to the issues the migration was meant to address.

## Vulnerability Details

`EmergencyMigration::migrate` function is intended to transition control of USDC from the `MoneyShelf` contract to the `MoneyVault` contract in case of an emergency. However, the function only upgrades the module without transferring the actual USDC balance to the new contract. The relevant code snippet is as follows:

```javascript
function migrate(Kernel kernel, IERC20 usdc, CrimeMoney crimeMoney) public returns (MoneyVault) {
    vm.startBroadcast(kernel.executor());
    MoneyVault moneyVault = new MoneyVault(kernel, usdc, crimeMoney);

@>  kernel.executeAction(Actions.UpgradeModule, address(moneyVault));
    vm.stopBroadcast();

    return moneyVault;
}
```

Without transferring the USDC, the funds remain in the `MoneyShelf`, defeating the purpose of the emergency migration.

## Impact

USDC remains in the vulnerable `MoneyShelf` contract, which may be exposed to the security issue the migration was meant to mitigate. Also, not transferring USDC to `MoneyVault` undermines its role as a secure fallback during emergencies, potentially leading to financial losses or operational failures.

## Tools Used

Manual Inspection

## Proof of Code

Place the following test into the `EmergencyMigration.t.sol`:

```javascript
function test_migrate() public {
    assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyShelf));

    vm.startPrank(deployer.godFather());
    usdc.approve(address(moneyShelf), type(uint256).max);
    laundrette.depositTheCrimeMoneyInATM(deployer.godFather(), address(moneyShelf), 1_000_000e6);
    vm.stopPrank();

    EmergencyMigration migration = new EmergencyMigration();
    uint256 balance_moneyShelf_before = usdc.balanceOf(address(moneyShelf));

    MoneyVault moneyVault = migration.migrate(kernel, laundrette, usdc, crimeMoney);

    uint256 balance_moneyShelf_after = usdc.balanceOf(address(moneyShelf));
    uint256 balance_moneyVault_after = usdc.balanceOf(address(moneyVault));

    console.log("~~~ Before ~~~");
    console.log("moneyShelf - before: ", balance_moneyShelf_before);
    console.log("moneyVault - before: ", 0);
    console.log("~~~~~~~~~~~~~~");
    console.log("~~~ After ~~~");
    console.log("moneyShelf - after: ", balance_moneyShelf_after);
    console.log("moneyVault - after: ", balance_moneyVault_after);
    console.log("~~~~~~~~~~~~~~");

    assertNotEq(address(moneyShelf), address(moneyVault));
    assertEq(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))), address(moneyVault));
    assertEq(balance_moneyShelf_before, balance_moneyShelf_after); // <--- Wrong assumption
}
```

Running this test yields the following output:

```bash
[PASS] test_migrate() (gas: 2338186)

~~~ Before ~~~
moneyShelf - before:  1000000000000
moneyVault - before:  0
~~~~~~~~~~~~~~
~~~ After ~~~
moneyShelf - after:  1000000000000
moneyVault - after:  0
~~~~~~~~~~~~~~
```

## Recommendations

To resolve this issue, modify the `EmergencyMigration::migrate` function to include the transfer of USDC from the `MoneyShelf` to the `MoneyVault`.

```diff
function deploy() public returns (Kernel, IERC20, CrimeMoney, WeaponShelf, MoneyShelf, Laundrette) {
    // Code above stays the same...

    kernel.grantRole(Role.wrap("moneyshelf"), address(moneyShelf));
+   kernel.grantRole(Role.wrap("gangmember"), address(godFather)); // Add godfather to the gang

    // Code below stays the same...
}
```

```diff
-function migrate(Kernel kernel, IERC20 usdc, CrimeMoney crimeMoney) public returns (MoneyVault) {
+function migrate(Kernel kernel, Laundrette laundrette, IERC20 usdc, CrimeMoney crimeMoney) public returns (MoneyVault) {
    vm.startBroadcast(kernel.executor());
    MoneyVault moneyVault = new MoneyVault(kernel, usdc, crimeMoney);

+   MoneyShelf moneyShelf = MoneyShelf(address(kernel.getModuleForKeycode(Keycode.wrap("MONEY"))));
+   laundrette.withdrawMoney(address(moneyShelf), address(moneyVault), usdc.balanceOf(address(moneyShelf)));

    kernel.executeAction(Actions.UpgradeModule, address(moneyVault));
    vm.stopBroadcast();

    return moneyVault;
}
```

Running the test again would yield the following outcome:

```bash
[FAIL. Reason: assertion failed: 1000000000000 != 0] test_migrate() (gas: 2464336)

~~~ Before ~~~
moneyShelf - before:  1000000000000
moneyVault - before:  0
~~~~~~~~~~~~~~
~~~ After ~~~
moneyShelf - after:  0
moneyVault - after:  1000000000000
~~~~~~~~~~~~~~
```

## <a id='M-02'></a>M-02. Unauthorized role revocation in `Laundrette::quitTheGang` allows any gang member to remove others from the gang

### Relevant GitHub Links

https://github.com/Cyfrin/2024-05-mafia-take-down/blob/main/src/policies/Laundrette.sol#L88-L90

## Summary

`Laundrette::quitTheGang` allows any gang member to revoke the `gangmember` role from any other member. This can lead to unauthorized removals of gang members, disrupting the functionality and security of the protocol.

## Vulnerability Details

`Laundrette::quitTheGang` is intended to allow gang members to remove themselves from the gang. However, the current implementation allows any gang member to call this function and specify an arbitrary account to revoke the `gangmember` role from. The relevant code is as follows:

```javascript
function quitTheGang(address account) external onlyRole("gangmember") {
@>  kernel.revokeRole(Role.wrap("gangmember"), account);
}
```

## Impact

This vulnerability allows any gang member to revoke the `gangmember` role from other members without their consent. This can lead to disruption of normal workflow, because gang members can constantly revoke other people's roles and prevent them from using the protocol.

## Tools Used

Manual code review

## Recommendations

`Laundrette::quitTheGang` should be modified to ensure that gang members can only remove their own role. This can be done by restricting the account parameter to `msg.sender`. Here is the updated implementation:

```diff
-function quitTheGang(address account) external onlyRole("gangmember") {
+function quitTheGang() external onlyRole("gangmember") {
-   kernel.revokeRole(Role.wrap("gangmember"), account);
+   kernel.revokeRole(Role.wrap("gangmember"), msg.sender);
}
```
