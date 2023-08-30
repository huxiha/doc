# Ethernaut 08-Vault

## 本关目标

- 调用 unlock 成功

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  // 只有知道密码才能unlock
  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

## 解决方案

1. 在合约中即使是 private 的状态变量，我们也可以获取，因为区块链上的内容就在那里，password 位于合约的 slot1
2. 在 ethernaut 浏览器 console 中使用 await web3.eth.getStorageAt(contract.address, 1),获取 password 的值
3. 调用 unlock(获取到的 passeord)解锁
