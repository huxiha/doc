## Flash Loans

是一种特殊的借贷方式，你可以借贷一些 token 进行一些交易不需要抵押，但是在总交易的最后你需要归还借贷的所有 token 并支付相应的利息已完成这笔交易，否则，整个交易回滚。也就是借还都在同一个交易中。

**简单应用**

- 套利
- defi hack

**套利**  
假设有两个交易所，A 和 B,A 出售的某个 token 的价格比 B 低，因此你可以借一些 DAI 从 A 买入一些 token,然后在 B 卖，获得更多的 DAI,还完借贷的 DAI 已经利息后，还有盈余

**闪电贷合约调用基本步骤**

- 首先你的合约要闪电贷提供者（比如 Aave）的功能，告诉它你要贷什么，以及贷多少
- 闪电贷提供者会发送资产到你的合约，并回调你合约中的 executeOperation()
- executeOperation()是你自定义的功能，这里你可以使用借贷到的资产进行一系列操作，完成后，approve 闪电贷提供者收回它给你的资产以及利息
- 闪电贷提供者收回它给你的资产以及利息
- 如果中间有任意一环不成功，整个交易回滚，就像你没有借贷过一样

**aave flashloans**

- 你的合约要符合接口 IFlashloanSimpleReceiver.sol 或者 IFlashloanReceiver.sol 接口，并实现接口的 executeOperation()函数，在其中允许 POOL 去收回资产
- 在你的合约中调用制定池子的 flashloan()或者 flashloanSimple()

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import "@aave/core-v3/contracts/flashloan/base/FlashLoanSimpleReceiverBase.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FlashLoansExample is FlashLoanSimpleReceiverBase {

    // https://docs.aave.com/developers/deployed-contracts/v3-testnet-addresses在这里可以找到pooladdressProvider的地址
    constructor(
        IPoolAddressesProvider provider
    ) FlashLoanSimpleReceiverBase(provider) {}



    function createFlashLoan(address asset, uint256 amount) external {
        address receiver = address(this);
        bytes memory params = "";
        uint16 referralCode = 0;

        POOL.flashLoanSimple(receiver, asset, amount, params, referralCode);
    }

    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool) {
        //一些操作

        uint256 amountOwing = amount + premium;
        IERC20(asset).approve(address(POOL), amountOwing);
        return true;
    }
}

```
