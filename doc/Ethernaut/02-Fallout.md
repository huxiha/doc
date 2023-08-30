# Ethernaut 02-Fallout

## 本关目标

- 成为合约的 owner

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0; //注意该合约使用的是solidity 0.6.0版本

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {

  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  // 6.*版本中的构造函数使用的是和合约同名的函数来进行声明
  // 该合约中，构造函数名写错，因此我们可以直接调用该函数进行owner的修改
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

## 解决方案

1. 调用 fal1out()修改 owner

- 在 remix 中部署的时候可以写成接口来进行 At Address 部署

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface Fallout {
  function Fal1out() external payable;
}
```
