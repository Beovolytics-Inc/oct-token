# oct-token

## Function specification

### Contract 'OctToken'

This is a contract based on standard [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) `ERC20` and `Ownable`, with following functions added:

* The token has name `OctToken` and symbol `OCT`.
* The token has fixed total supply - 100 million (100,000,000).
* All of the OCT tokens will be minted to the owner (deployer) of the contract at construction time. After this, there is NO WAY to mint or burn OCT tokens.
* Only the owner of the contract can transfer OCT tokens to other accounts until the function `unlockTransfer()` is called.
* The function `unlockTransfer()` can ONLY be called by the owner of the contract.
* After the owner of the contract call function `unlockTransfer()` the contract will act as a standard ERC20 contract, and there is NO WAY to lock the contract again.

### Contract 'OctFoundationTimelock'

This is a contract based on standard [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) `Ownable`, with the following functions added:

* This contract has the following constant timestamps:
  * `earliest release start time` - the earliest timestamp of token release period. Before this time NO ONE can withdraw any token from this contract. The time should be the start time of a day.
  * `release end time` - the end timestamp of token release period. After this time, ANY ONE can withdraw any amount of tokens they held. The time should be the start time of a day.
* The `beneficiary` of this contract has the following properties:
  * `unreleased balance` - the amount of unreleased balance of the beneficiary
  * `release start time` - the start time when the beneficiary can withdraw their tokens
  * `released balance` - the amount of released balance of the beneficiary
  * `withdrawed balance` - the amount of withdrawed balance of the beneficiary
  * `supervised unreleased balance` - the amount of supervised unreleased balance of the beneficiary, the owner of this contract can decrease it.
* This contract will accept an address of contract `OctToken` at construction time, and the address will be immutable after construction.
* Anyone can call function `token()` to get the address of contract `OctToken` bonded to this contract.
* This contract has a private function `_balanceToReleaseTo(address, supervised)` which implements the following logic:
  * If `block.timestamp` is smaller than {`release start time` of `address`}, return 0
  * Calculate {count of days passed since `release start time` of `address`} : (`block.timestamp` - {`release start time` of `address`}) / 86400
  * Calculate {count of days of release duration} : (`release end time` - {`release start time` of `address`}) / 86400
  * If param `supervised` is `true`, return {`supervised unreleased balance` of `address`} * {count of days passed since `release start time` of `address`} / {count of days of release duration}
  * If param `supervised` is `false`, return {`unreleased balance` of `address`} * {count of days passed since `release start time` of `address`} / {count of days of release duration}
* Anyone can call function `unreleasedBalanceOf(address)` to get the `unreleased balance` of an beneficiary corresponding to param `address`.
* Anyone can call function `withdrawedBalanceOf(address)` to get the withdrawed balance of an beneficiary corresponding to the param `address`.
* Anyone can call function `supervisedUnreleasedBalanceOf(address)` to get the `supervised unreleased balance` of an beneficiary corresponding to param `address`.
* Anyone can call function `releasedBalanceOf(address)` to get the released balance of an beneficiary corresponding to param `address`.
  * The result of this function is calculated by: {`released balance` of `address`} + `_balanceToReleaseTo(address, true)` + `_balanceToReleaseTo(address, false)`
* This contract has a private function `_benefit(address, amount, supervised)` which implements the following logic:
  * If the `block.timestamp` is smaller than `earliest release start time`, update the properties of the beneficiary corresponding to param `address` as follow:
    * `unreleased balance` :
      * If param `supervised` is `true` : NO change
      * If param `supervised` is `false` : {`unreleased balance` of `address`} + `amount`
    * `release start time` : `earliest release start time`
    * `released balance` : NO change
    * `withdrawed balance` : NO change
    * `supervised unreleased balance` :
      * If param `supervised` is `true` : {`supervised unreleased balance` of `address`} + `amount`
      * If param `supervised` is `false` : NO change
  * If the `block.timestamp` is larger than `earliest release start time`, update the properties of the beneficiary corresponding to param `address` as follow:
    * `released balance` : `releasedBalanceOf(address)`
    * `unreleased balance` :
      * If param `supervised` is `true` : {`unreleased balance` of `address`} - `_balanceToReleaseTo(address, false)`
      * If param `supervised` is `false` : {`unreleased balance` of `address`} - `_balanceToReleaseTo(address, false)` + `amount`
    * `withdrawed balance` : NO change
    * `supervised unreleased balance` :
      * If param `supervised` is `true` : {`supervised unreleased balance` of `address`} - `_balanceToReleaseTo(address, true)` + `amount`
      * If param `supervised` is `false` : {`supervised unreleased balance` of `address`} - `_balanceToReleaseTo(address, true)`
    * `release start time` : `block.timestamp` - (`block.timestamp` % 86400)
* Only the owner (deployer) of this contract can call function `benefit(address, amount, supervised)` to increase `unreleased balance` or `supervised unreleased balance` of an beneficiary corresponding to param `address`.
  * This function is a simple wraper of private function `_benefit(address, amount, supervised)`.
  * The param `address` MUST be an EOA address. (This will be verified by the owner of this contract rather than by contract code.)
* Any beneficiary can call function `withdraw(amount)` to withdraw a certain amount tokens to the address of himself.
  * The param `amount` must be less or equal to avaialable balance, which is calculated by: `releasedBalanceOf(_msgSender())` - {`withdrawed balance` of `_msgSender()`}
  * The {`withdrawed balance` of `_msgSender()`} will be increased by `amount`, if the token transfer is success.
* Any beneficiary can call function `transferUnreleasedBalance(address, amount)` to transfer a part or whole of his unreleased balance which is NOT released yet to another account (address).
  * The param `amount` must be less or equal to unreleased balance, which is calculated by: {`unreleased balance` of `_msgSender()`} - `_balanceToReleaseTo(_msgSender(), false)`
  * Call private function `_benefit(address, amount, false)`.
  * If the `block.timestamp` is smaller than `earliest release start time`, update the properties of the beneficiary corresponding to `_msgSender()` as follow:
    * `unreleased balance` : {`unreleased balance` of `_msgSender()`} - `amount`
    * `release start time` : `earliest release start time`
    * `released balance` : NO change
    * `withdrawed balance` : NO change
    * `supervised unreleased balance` : NO change
  * If the `block.timestamp` is larger than `earliest release start time`, update the properties of the beneficiary corresponding to `_msgSender()` as follow:
    * `released balance` : `releasedBalanceOf(_msgSender())`
    * `unreleased balance` : {`unreleased balance` of `_msgSender()`} - `_balanceToReleaseTo(_msgSender(), false)` - `amount`
    * `withdrawed balance` : NO change
    * `supervised unreleased balance` : {`supervised unreleased balance` of `_msgSender()`} - `_balanceToReleaseTo(_msgSender(), true)`
    * `release start time` : `block.timestamp` - (`block.timestamp` % 86400)
* Only the owner (deployer) of this contract can call function `decreaseBenefitOf(address, amount)` to decrease the benefit of a certain beneficiary corresponding to param `address`.
  * The param `amount` must be less or equal to unreleased balance, which is calculated by: {`supervised unreleased balance` of `address`} - `_balanceToReleaseTo(address, true)`
  * If the `block.timestamp` is smaller than `earliest release start time`, update the properties of the beneficiary corresponding to `address` as follow:
    * `unreleased balance` : NO change
    * `release start time` : `earliest release start time`
    * `released balance` : NO change
    * `withdrawed balance` : NO change
    * `unreleased balance` : {`supervised unreleased balance` of `address`} - `amount`
  * If the `block.timestamp` is larger than `earliest release start time`, update the properties of the beneficiary corresponding to `address` as follow:
    * `released balance` : `releasedBalanceOf(address)`
    * `unreleased balance` : {`unreleased balance` of `address`} - `_balanceToReleaseTo(address, false))`
    * `withdrawed balance` : NO change
    * `supervised unreleased balance` : {`supervised unreleased balance` of `address`} - `_balanceToReleaseTo(address, true))` - `amount`
    * `release start time` : `block.timestamp` - (`block.timestamp` % 86400)

## Installation

### Install dependencies

Install openzeppelin/contracts.

```shell
npm install @openzeppelin/contracts
```

### Install dependencies for development

Install hardhat for testing.

```shell
npm install --save-dev hardhat
```

Install modules for running testing scripts compatible with Waffle.

```shell
npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
```

## Test

You can run the tests by:

```shell
npx hardhat test
```

or

```shell
npm run test
```
