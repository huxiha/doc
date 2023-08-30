# Ethernaut 22-Dex

## 本关目标

- 取光合约中的任意一种 token

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {

  // 有两种Token
  address public token1;
  address public token2;
  constructor() {}

  // 设置两个token的地址
  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  // 合约的拥有者可以向给当前合约充值某种token
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }

  // 用from token交换出to token, from和to限制为token1或token2
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  // 交换token的算法
  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

## 解决方案

1. 根据交换 token 的算法，推演如何取光一种 token

- 自己账户中 token1(10)、token2(10) ｜ 合约账户中 token1(100)、token2(100)
- 使用 10 个 token1 交换 token2--> 自己账户中 token1(0)、token2(10\*100/100 + 10 = 20) | 合约账户中 token1(110)、token2(90)
- 使用 20 个 token2 交换 token1--> 自己账户中 token1(20 \* 110 / 90 = 24)、token2(0) | 合约账户中 token1(86)、token2(110)
- 使用 24 个 token1 交换 token2--> 自己账户中 token1(0)、token2(24 \* 110 / 86 = 30) | 合约账户中 token1(110)、token2(80)
- 使用 30 个 token2 交换 token1--> 自己账户中 token1(30 \* 110 / 80 = 41)、token2(0) | 合约账户中 token1(69)、token2(110)
- 使用 41 个 token1 交换 token2--> 自己账户中 token1(0)、token2(41 \* 110 / 69 = 65) | 合约账户中 token1(110)、token2(45)
- 使用 45 个 token2 交换 token1--> 自己账户中 token1(45 \* 110 / 45 = 110)、token2(20) | 合约账户中 token1(0)、token2(90)

2. 部署 ERC20 接口使用 At Address token1 和 token2 地址，分别 approve Dex 合约 100token
3. 按照第一步中值调用 Dex 中的 swap 即可

```solidity
interface IERC20{
    function balanceOf(address account) external view returns (uint256);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function approve(address spender, uint256 value) external returns (bool);
    function transfer(address to, uint256 value) external returns (bool);
}
```
