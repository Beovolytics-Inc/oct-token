# oct-token

## Function specification

### Contract 'OctToken'

This is a contract based on standard [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) `ERC20` and `Ownable`, with following functions added:

* The token is with name `OctToken` and symbol `OCT`.
* The token is With fixed total supply - 100 million (100,000,000).
* All of the OCT tokens will be minted to the owner (deployer) of the contract at construction time. After this, there is NO WAY to mint or burn OCT tokens.
* Only the owner of the contract can transfer OCT tokens to other accounts until the function `unlockTransfer` is called.
* The function `unlockTransfer` can ONLY be called by the owner of the contract.
* After the owner of the contract call function `unlockTransfer` the contract will act as a standard ERC20 contract, and there is NO WAY to lock the contract again.

### Contract 'OctFoundationTimelock'

This is a contract based on standard [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) `Ownable`, with following functions added:

* This contract will have following constant timestamp:
  * `earliest release start time` - the earliest timestamp of token release period. Before this time NO ONE can withdraw any token from this contract. The time should be the start time of a day.
  * `release end time` - the end timestamp of token release period. After this time, ANY ONE can withdraw any amount of tokens they held. The time should be the start time of a day.
* The `beneficiary` of this contract will have following properties:
  * `held balance` - the balance held by the beneficiary
  * `release start time` - the start time when the beneficiary can withdraw their tokens
  * `released balance` - the amount of released balance of the beneficiary
  * `withdrawed balance` - the amount of withdrawed balance of the beneficiary
* This contract will accept an address of contract `OctToken` at construction time, and the address will be immutable after construction.
* Anyone can call function `token()` to get the address of contract `OctToken` bonded to this contract.
* This contract has a private function `_benefit(address, amount)` which implements the following logic:
  * If the `block.timestamp` is smaller than `earliest release start time`, update the properties of the beneficiary corresponding to param `address` as follow:
    * `held balance` - {`held balance` of `address`} + {`amount`}
    * `release start time` - `earliest release start time`
    * `released balance` - 0
    * `withdrawed balance` - 0
  * If the `block.timestamp` is larger than `earliest release start time`, update the properties of the beneficiary corresponding to param `address` as follow:
    * `release start time` - `block.timestamp` - (`block.timestamp` % 86400)
    * `released balance` - `releasedBalanceOf(address)`
    * `held balance` - {`held balance` of `address`} - {`released balance` of `address`} + {`amount`}
    * `withdrawed balance` - NO change
* Only the owner (deployer) of this contract can call function `benefit(address, amount)` to increase held balance of an beneficiary.
  * This function is a simple wraper of private function `_benefit(address, amount)`.
  * The address of a beneficiary MUST be an EOA address. (This will be verified by the owner of this contract rather than by contract code.)
* Anyone can call function `heldBalanceOf(address)` to get the `held balance` of an beneficiary corresponding to param `address`.
* Anyone can call function `releasedBalanceOf(address)` to get the released balance of an beneficiary corresponding to param `address`. The result of this function is calculated by: {`released balance` of `address`} + {`held balance` of `address`} * {the days passed since `release start time` of `address`} / {the days of release duration}
  * {the days passed since `release start time` of `address`} is calculated by: ({`block.timestamp`} - {`release start time` of `address`}) / 86400
  * {the days of release duration} is calculated by: ({`release end time`} - {`release start time` of `address`}) / 86400
* Any beneficiary can call function `withdraw(amount)` to withdraw a certain amount tokens to the address of himself.
  * The param `amount` must be less or equal to avaialable balance, which is calculated by: {`releasedBalanceOf(_msgSender())`} - {`withdrawed balance` of `_msgSender()`}
  * The {`withdrawed balance` of `_msgSender()`} will be increased by param `amount`, if the token transfer is success.
* Anyone can call function `withdrawedBalanceOf(address)` to get the withdrawed balance of an beneficiary corresponding to the param `address`.
* Any beneficiary can call function `transferUnreleasedBalance(address, amount)` to transfer a part or whole of his held balance which is NOT released yet to another account (address).
  * The param `amount` must be less or equal to unreleased balance, which is calculated by: {`held balance` of `_msgSender()`} - {`releasedBalanceOf(_msgSender())`}
  * Call private function `_benefit(address, amount)`.
  * If the `block.timestamp` is smaller than `earliest release start time`, update the properties of the beneficiary corresponding to `_msgSender()` as follow:
    * `held balance` - {`held balance` of `_msgSender()`} - {`amount`}
    * `release start time` - `earliest release start time`
    * `released balance` - 0
    * `withdrawed balance` - 0
  * If the `block.timestamp` is larger than `earliest release start time`, update the properties of the beneficiary corresponding to `_msgSender()` as follow:
    * `release start time` - `block.timestamp` - (`block.timestamp` % 86400)
    * `released balance` - {`releasedBalanceOf(_msgSender())`}
    * `held balance` - {`held balance` of `address`} - {`released balance` of `address`} - {`amount`}
    * `withdrawed balance` - NO change

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
