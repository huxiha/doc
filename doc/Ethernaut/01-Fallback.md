# Ethernaut 01-Fallback

## 本关目标

- 成为合约的 owner
- 将合约余额降为 0

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  // 记录地址和贡献余额的对应关系
  mapping(address => uint) public contributions;
  // 合约owner， 我们的目标是将这个变量值变成我们的账户地址
  address public owner;

  // 构造函数声明owner为合约创建者，合约创建者贡献1000ether
  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  // 只有owner才可以操作的modifier
  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    // 调用该函数要求携带的ether要少于0.001ether
    require(msg.value < 0.001 ether);
    // 添加贡献者及对应的余额
    contributions[msg.sender] += msg.value;
    // 如果贡献的余额大于owner的余额，那么重置owner
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  // 我们的另一个目标是清零合约余额，因此必须成为owner调用withdraw()
  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  // 接受任意账户的转账，只要金额大于0，并且之前有贡献过，就将owner设置成转账的账户地址，我们可以利用这个修改owner
  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

## 解决方案

1. 调用 contribute()，携带少于 0.001ether，添加 contributions[msg.sender]，满足 contributions[msg.sender] > 0
2. 直接向合约 Fallback 转账 1ether, 满足 msg.value > 0
3. 前两步完成 owner 修改
4. 调用 withdraw()，清零 Fallback 合约余额
