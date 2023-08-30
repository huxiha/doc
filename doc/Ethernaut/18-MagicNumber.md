# Ethernaut 18-MagicNumber

## 本关目标

- 给 MagicNumber 合约提供一个合约地址，这个合约里实现一个函数 whatIsTheMeaningOfLife()返回 42
- 要求这个合约的实现要在 10opcode 之内

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  // 只需要提供能返回42的小于等于10 opcode的合约地址即可
  function setSolver(address _solver) public {
    solver = _solver;
  }

  // 哈哈哈，42

  /*
    ____________/\\\_______/\\\\\\\\\_____
     __________/\\\\\_____/\\\///////\\\___
      ________/\\\/\\\____\///______\//\\\__
       ______/\\\/\/\\\______________/\\\/___
        ____/\\\/__\/\\\___________/\\\//_____
         __/\\\\\\\\\\\\\\\\_____/\\\//________
          _\///////////\\\//____/\\\/___________
           ___________\/\\\_____/\\\\\\\\\\\\\\\_
            ___________\///_____\///////////////__
  */
}
```

## 解决方案

1. 在 solidity-by-example 中有个例子使用汇编实现返回 42 的合约，偷个懒可以直接借鉴

```solidity
/ SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./MagicNumber.sol";

contract Hack {

    constructor(MagicNum magicAddress){
        address solverAddr;
        bytes memory bytecode = hex"69602a60005260206000f3600052600a6016f3";
        assembly {
            // create(value, offset, size)
            solverAddr := create(0, add(bytecode, 0x20), 0x13) //前32位存长度，从+32位位置定位到bytecode, byteCode长度19bytes 0x13
        }
        require(solverAddr != address(0));
        magicAddress.setSolver(solverAddr);
    }
}

/*
Run time code - return 42
602a60005260206000f3

// Store 42 to memory
mstore(p, v) - store v at memory p to p + 32

PUSH1 0x2a
PUSH1 0
MSTORE

// Return 32 bytes from memory
return(p, s) - end execution and return data from memory p to p + s

PUSH1 0x20
PUSH1 0
RETURN

Creation code - return runtime code
69602a60005260206000f3600052600a6016f3

// Store run time code to memory
PUSH10 0X602a60005260206000f3
PUSH1 0
MSTORE

// Return 10 bytes from memory starting at offset 22
PUSH1 0x0a
PUSH1 0x16
RETURN
*/
```
