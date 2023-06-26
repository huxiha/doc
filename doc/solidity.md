# solidty by example

## Hello world

solidty 基本格式：

```solidity
//SPDX-License-Identifier: MIT
//一个注释标识license

pragma solidity ^0.8.17;
//pragma指定solidity使用的编译版本，^表示之后版本都可以编译

contract SayHello {
    //合约关键字contract 合约名
    string public hello = "Hello world!";
}
```

## 第一个 app

实现合约中存储数值的增加和减少（步长为 1）

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Counter {
    //因为要减所以可能是负值，数据类型定义成int
    //private表示该变量外部不可见，用户不可见
    int256 private count = 0;

    //function定义一个函数 function functionName (parameters...)
    //public访问修饰，表示公共的，大家都可以访问
    //view表示这个函数是查数值
    //returns指定返回类型
    function getCurrentCount() public view returns(int256) {
        return count;
    }

    function increaseCount() public  {
        count += 1;
    }

    function decreaseCount() public  {
        count -= 1;
    }
}
```

## 基本数据类型

- bool
- uint256
- int256
- maxInt(type(int).max)
- minint(type(int).min)
- address
- bytes

不赋初值的时候，这些类型都有默认值

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract DataType {
    //boolean 类型，true/false
    bool public boo = true;

    //值范围：uint8:0-2**8-1; uint256:0-2**256 - 1
    uint256 public u256 = 123;
    uint public u = 2456; //uint等价于uint256

    //值范围：int8: -2**7~2**7-1
    int256 public i256 = -100;
    int8 public i8 = 10;
    int public i = 100; //int等价于int256

    //最大值最小值
    int public maxInt = type(int).max;
    int public minInt = type(int).min;

    // 地址类型
    address public addr = 0xCA35b7d915458EF540aDe6068dFe2F44E8fa733c;

    // byte: 表示一个组位序列，可以看成是bit的数组, bytes有两种类型：
    // 一种是固定长度的byte数组
    // 一种是可动态变化长度的byte数组

    // solidity中bytes数据类型，表示动态byte数组，byte[]
    bytes1 public b = 0x5d; //[010111101]
    bytes2 public a = 0x5d2e;

    // 默认值
    bool public b0; // false
    uint256 public u0; // 0
    int256 public i0; // 0
    address public addr0; // 0x0000000000000000000000000000000000000000
    bytes public by0;
}
```

## 变量

solidity 中有三种类型的变量

- local 本地临时变量 (在函数中声明，不会存储到区块链中)
- state 状态变量 (在函数外声明，会存储到区块链中)
- global 全局变量 (提供区块链信息的变量)

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Varaiables {

    // 状态变量，存储到区块链上
    string public s_text = "abcdefg";
    uint public s_num = 123;

    function doSomething() public {
        // 本地临时变量，不会存储到区块链上
        uint256 i = 256;

        // 描述区块链信息的全局变量
        uint256 timestamp = block.timestamp; // 当前区块时间戳
        address sender = msg.sender; // 合约请求者的地址
    }
}
```

## 常量

关键字 constant

不能被修改的变量，常量的值被硬编码，使用常量可以节省 gas 费

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Constants {
    // 习惯性使用大写字母命名常量
    address public constant MY_ADDRESS = 0x777788889999AaAAbBbbCcccddDdeeeEfFFfCcCc;
    uint public constant MY_UINT= 123;
}
```

## Immutable

关键字 immutable
这种类型的变量，只能在构造函数中赋值，有值后不能在构造函数之外被修改

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Immutable {
    address public immutable i_MY_ADDRESS;
    uint public immutable i_MY_UINT;

    constructor(uint _myUint) {
        i_MY_UINT = _myUint;
        i_MY_ADDRESS = msg.sender;
    }
}
```

## 读写状态变量

写或者更新状态变量，需要发送交易；读状态变量，不需要发送交易，是免费的

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract SimpleStorage{
    uint256 public num;

    //写状态变量或者修改状态变量值的操作，需要发送交易
    function setNum(uint256 _num) public{
        num = _num;
    }

    // 读取状态变量当前值，不需要发送交易
    function getNum() public view returns(uint256) {
        return num;
    }
}
```

## Ether 和 Wei

交易费用是使用 Ether 支付的，1 ether = 10\*\*18wei

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract EtherUnits {
    uint public oneWei = 1 wei;

    // 1wei等于1
    bool public oneWeiEqualsOne = oneWei == 1;

    uint public oneEther = 1 ether;

    // 1ether等于10**18
    bool public oneEtherEquals18 = oneEther == 1e18;
}
```

## Gas(汽油费)

如何计算一笔交易需要花费的汽油费?  
需要支付 gas spent \* gas price 数量的 ether, 其中 gas 是消耗的单位，gas spent 表示这个交易中一共消耗的 gas 数量，gas price 是每一消耗单位，你想支付的费用。  
拥有较高 gas price 的交易被打包到区块中被发布的优先级会更高。  
没有使用完的 gas 费会被退回。

**Gas Limit(汽油费限制)**  
有两种汽油费上限：

- gas limit (这个是发布交易的人自己定的，自己愿意为这笔交易支付的最大费用)
- block gas limit (区块的费用上限，这个区块中允许的最大汽油费，由网络决定)

如果交易用光了发布者定义的最大汽油费，那么交易会回滚，状态变量值不会发生变化，并且汽油费不会退回。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Gas {
    uint256 public ui = 0;

    //这里有个无限循环，当消费光所有gas费后，状态变量ui的状态不会更新，还是0; 并且消耗的汽油费不会退回
    function forever() public {
        while(true) {
            ui += 1;
        }
    }
}
```

## 条件语句 If/Else

solidity 支持条件判断语句 if, else 和三目判断

```solidity
//SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract IfElse {
    function foo(uint x) public pure returns(uint) {
        if(x < 10) {
            return 0;
        } else if(x < 20 ) {
            return 1;
        } else {
            return 2;
        }
    }

    function ternay(uint _x) public pure returns(uint) {
        return _x > 10 ? 1 : 0;
    }
}
```

## For 和 While 循环

solidity 支持 for, while 和 do while 循环。
不要写没有停止边界的循环，会导致耗光 gas limit，交易失败, 基于这种原因，while 和 do while 很少被使用。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Loop {
    function loop() public {

        for(uint i = 0; i <  10; i++) {
            if(i == 3) {
                continue; //忽略之后没有被执行的代码，继续下一次循环
            }

            if(i == 5) {
                break;//退出for循环，结束循环
            }
        }

        uint j;
        while(j < 10) {
            j++;
        }
    }
}
```

## 映射 Mapping

mapping 创建语法：mapping(关键字类型 => 值类型)  
关键字类型可以是任意内置数据类型，bytes, string,或者任意合约；  
值类型可以是任意类型，也包括另一个 mapping,或者一个数组。
mapping 是不可迭代的。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Mapping {

    // 地址到uint类型的映射
    mapping(address => uint256) public myMap;

    function getMyMapWith(address _addr) public view returns(uint256) {
        return myMap[_addr]; // mapping永远都会返回一个值，如果之前没有设置过，会返回默认值
    }

    function setMyMap(address _addr, uint256 _i) public {
        myMap[_addr] = _i; //更新或者设置某一个键值的值
    }

    function removeFromMyMap(address _addr) public {
        delete myMap[_addr];
    }
}

contract NestedMapping {
    // 嵌套mapping, mapping的key值指向另一个mapping
    mapping(address => mapping(uint => bool)) public nested;

    function get(address _addr, uint _i) public view returns(bool) {
        return nested[_addr][_i];
    }

    function set(address _addr, uint _i, bool _boo) public {
        nested[_addr][_i] = _boo;
    }

    function remove(address _addr, uint _i) public {
        delete nested[_addr][_i];
    }
}
```

## 数组

数组有两类：编译时大小固定的数组和动态大小的数组。

数组基本使用

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Array {
    //初始化数组的方式
    //动态大小的数组声明
    uint[] public arr;
    uint256[] public arr2 = [1, 2, 3];

    // 固定大小的数组声明
    uint[10] public myFixedSizeArry;

    // 按照数组索引返回数组元素
    function get(uint i) public view returns(uint) {
        return arr2[i];
    }

    // solidity可以返回整个数组，但是要避免返回长度无限增长的数组
    function getArray() public view returns(uint[] memory) {
        return arr;
    }

    function push(uint i) public {
        //数组中追加元素，数组长度增加1
        arr.push(i);
    }

    function pop() public {
        // 删除数组的最后一个元素，数组长度减少1
        arr.pop();
    }

    function getLength() public view returns(uint) {
        // 返回数组的长度
        return arr.length;
    }

    function remove(uint index) public {
        //不会减少数组长度，只是把指定坐标的数组元素设置为默认值
        delete arr[index];
    }

    function examples() external {
        //创建一个内存中的数组，只有固定大小的数组可以这样创建
        uint[] memory a = new uint[](5);
    }
}
```

**从数组中移除指定元素**

通过将删除元素之后所有元素前移的方式

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract ArrayRemovedByShifting {
    uint[] public arr;

    function remove(uint index) public {
        require(index < arr.length, "index out of bound!");

        // 移除元素之后的所有元素依次前移
        for(uint i = index; i < arr.length - 1; i++) {
            arr[i] = arr[i + 1];
        }
        // 移动完后删除数组最后一个元素，数组长度减少1
        arr.pop();
    }

    function test() external {
        arr = [1, 2, 3, 4, 5];
        remove(2);

        assert(arr[0] == 1);
        assert(arr[1] == 2);
        assert(arr[2] == 4);
        assert(arr[3] == 5);
        assert(arr.length == 4);

        arr = [1];
        remove(0);
        assert(arr.length == 0);
    }
}
```

通过复制数组最后一个元素到删除位置

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract ArrayReplaceFromEnd {

    uint256[] public arr = [1, 3, 5, 7, 9];

    function remove(uint256 index) public {
        require(index < arr.length, "index out of bound!");
        // 将数组最后一个元素覆盖要删除的元素
        arr[index] = arr[arr.length - 1];
        arr.pop();
    }

    function test() external {
        arr = [1, 2, 3, 4];
        remove(1);
        assert(arr[0] == 1);
        assert(arr[1] == 4);
        assert(arr[2] == 3);
        assert(arr.length == 3);

        remove(2);
        assert(arr[0] == 1);
        assert(arr[1] == 4);
        assert(arr.length == 2);
    }
}
```

## 枚举 Enum

solidity 支持枚举类型，枚举对模型选择和状态跟踪很有用。  
enum 可以声明在合约外部。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Enum {
    // 定义表示运输状态的枚举 最后一项后面没有,
    enum Status {
        Pending,
        Shipped,
        Accepted,
        Rejected,
        Canceled
    }

    // 定义枚举类型的变量
    Status public status; // 枚举类型默认值为枚举第一个值，这里为Pending


    // 返回的实际是uint
    // Pending-0
    // Shipped-1
    // Accepted-2
    // Rejected-3
    // Canceled-4
    function get() public view returns(Status) {
        return status;
    }

    // 传递uint即可
    function set(Status _status) public {
        status = _status;
    }

    //  可以使用枚举的某一指定状态更新枚举变量
    function cancel() public {
        status = Status.Canceled;
    }

    // 重置枚举变量为默认值0，即枚举第一个状态
    function reset() public {
        delete status;
    }

}
```

在合约外部声明 enum 和导入 enum  
这是一个声明 enum 的文件 EnumDeclaration.sol

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

enum Status {
        Pending,
        Shipped,
        Accepted,
        Rejected,
        Canceled
    }
```

从其他文件导入 enum

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

// 导入enum
import "./EnumDeclaration.sol";

contract Enum2 {
    Status public status;
}
```

## Struct

可以通过创建 struct 定义自己的类型；  
struct 对于聚合有关联的数据很有帮助；  
struct 可以在合约外部声明，也可以从其他合约导入。

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Todos {
    struct Todo {
        string text;
        bool completed;
    }

    // Todo struct类型的数组
    Todo[] public todos;

    function create(string calldata _text) public {

        // 初始化struct, 方式1:像调用函数一样传参
        todos.push(Todo(_text, false));

        // 初始化struct, 方式2: 使用键值对方式传参
        todos.push(Todo({text: _text, completed: true}));

        // 初始化struct, 方式3: 初始化一个空的struct，然后更新赋值
        Todo memory todo;
        todo.text = _text;
        // 不配置completed,默认为false

        todos.push(todo);
    }

    // 当todos是public时，solidity会自动生成以下的getter函数
    function get(uint _index) public view returns(string memory, bool) {
        Todo storage todo = todos[_index];
        return (todo.text, todo.completed);
    }

    // 更新text
    function updateText(uint _index, string calldata _text) public {
        Todo storage todo = todos[_index];
        todo.text = _text;
    }

    // 更新completed
    function updateCompleted(uint _index) public {
        Todo storage todo = todos[_index];
        todo.completed = !todo.completed;
    }

}
```

**外部声明和导入 struct**  
文件 StructDeclaration.sol 中声明了 struct

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;


struct Todo {
    string text;
    bool completed;
}

```

在其他合约中导入 struct

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import './StructDeclaration.sol';

contract Other {
    Todo[] public todos;
}
```

## 数据位置 - Storage, Memory, 和 Calldata

变量可以被声明成 storage, memory 或者 calldata 三者之一，来显式指定数据存放的位置。

- storage 表示变量是状态变量（存储在区块链上）
- memory 表示变量在内存中，并且只在函数被调用的过程中存在
- calldata 表示包含函数参数的特殊数据位置

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract DataLocations {
    uint[] public arr;
    mapping(uint => address) map;
    struct MyStruct {
        uint foo;
    }
    mapping(uint => MyStruct) myStructs;

    function f() public {
        // 使用状态变量调用_f()
        _f(arr, map, myStructs[1]);

        // 从状态变量myStructs中获取的Struct,是存在区块链上的
        MyStruct storage myStruct = myStructs[1];

        //  创建一个在内存中的MyStruct
        MyStruct memory myMemStruct = MyStruct(1);
    }

    function _f(uint[] storage _arr, mapping(uint => address) storage _map, MyStruct storage _myStruct) internal {
        // 一些使用状态变量进行的操作
    }

    // 可以返回memory类型的
    function g(uint[] memory _arr) public returns(uint[] memory) {
        // 做一些操作使用memory的变量
    }

    function h(uint[] calldata _arr) public {
        // 做一些操作使用calldata的变量
    }

}
```

## 函数 Function

从一个函数中返回多个数据有多种方式  
在函数输入输出中不能使用 map  
调用多输入函数也有多种方式

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Function {
    // 函数返回多个值
    function returnMany() public pure returns(uint, bool, uint) {
        return (1, true, 2);
    }

    // 返回值可命名
    function named() public pure returns(uint x, bool b, uint y) {
        return (1, true, 2);
    }

    // 返回值可以通过命名赋值的方式，这样可以省去return关键字
    function assigned() public pure returns(uint x, bool b, uint y) {
        x = 1;
        b = true;
        y = 2;
    }

    // 调用其他返回多值的函数时，可以使用解构赋值
    function destructuringAssignents() public pure returns(uint, bool, uint, uint, uint) {
        (uint j, bool b, uint i)  = returnMany();

        // 可以有选择性的解构赋值，第二个就没有用到
        (uint x, ,uint y) = (4, 5, 6);
        return (i, b, j, x, y);
    }

    // 可以在函数输入中使用数组
    function arrayInput(uint[] memory _arr) public {

    }

    // 可以在函数输出中使用数组
    uint[] public arr;
    function arraryOutput() public view returns(uint[] memory) {
        return arr;
    }

    function funcWithManyInputs(uint x, uint y, uint z, address a, bool b, string memory s) public pure returns(uint){
        // 一些操作
    }

    // 直接一对一传参调用多入参函数
    function callFunc() public pure returns(uint) {
        return funcWithManyInputs(1, 2, 3, address(0), true, "c");
    }

     // 使用键值对的方式调用多入参函数
    function callFuncWithKeyValue() public pure returns(uint) {
        return funcWithManyInputs({a: address(1), b: true, x: 1, s: "s", y: 3, z: 100});
    }

}
```

## view 和 pure 函数

getter 函数可以被声明称 view 或者 pure

- view 函数声明表示函数中没有状态会被改变
- pure 函数表明表示函数中没有状态变量会改变或没有读取状态变量

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract ViewAndPure {
    uint public x = 1;

    // 函数不会修改状态变量，可能会读取用到状态变量
    function addToX(uint y) public view returns(uint) {
        return x + y;
    }

    // 函数不会修改状态变量， 也可能不会读取用到状态变量
    function add(uint i, uint j) public pure returns(uint) {
        return i + j;
    }

}
```

## 错误 Error

错误会会滚交易中对状态的修改  
你可以通过调用 require, assert, revert 来抛出错误

- require 用来在执行某些操作前验证输入或某些条件是否满足
- revert 和 require 类似，详细见下面的代码
- assert 用来判断程序中永远不可能是 false 的代码，如果断言失败，则可能程序有 bug

使用用户自定义错误可以节省 Gas 费

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Error {
    // require一般用来验证以下场景：
    // 输入是否满足条件
    // 执行前条件是否满足
    // 调用其他函数的返回值是否满足条件
    function testRequire(uint _i) public pure{
        require(_i > 10, "Input must be greater than 10!");
    }

    // 当检查条件复杂时， revert会有用
    function testRevert(uint _i) public pure {
        if(_i <= 10) {
            revert("Input must be greater than 10!");
        }
    }

    uint public num;

    function testAssert() public view {
        // assert应该只用来检查代码内部错误和不变量

        // 这里检查num应该一直是0
        assert(num == 0);
    }

    // 用户自定义错误

    error InsufficientBalance(uint balance, uint withdrawAmount);

    function testCustomError(uint withdrawAmount) public view {
        uint bal = address(this).balance;
        if(bal < withdrawAmount) {
            revert InsufficientBalance({balance: bal, withdrawAmount: withdrawAmount});
        }
    }

}
```

另一个例子

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Account{
    uint public balance;
    uint public constant MAX_UINT = 2 ** 256 - 1;

    function deposit(uint amount) public {
        uint oldBalance = balance;
        uint newBalance = balance + amount;

        // 如果newBalance小于oldBalance，则表示溢出
        require(newBalance >= oldBalance, "Overflow");
        balance = newBalance;

        assert(balance >= oldBalance);
    }

    function withdraw(uint amount) public {
        uint oldBalance = balance;

        // 要保证balance大于amount，才能取成功
        require(amount <= balance, "Underflow");
        balance -= amount;

        assert(balance <= oldBalance);
    }
}
```

## 函数修饰符 function modifier

修饰 modifier 就是在调用函数前或后/前和后会执行的代码
modifier 可以用来：

- 限制访问
- 验证输入
- 防止重入攻击

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract functionModifier {
    address public owner;
    uint public x = 30;
    bool public locked;

    constructor() {
        // 初始owner是合约的创建者
        owner = msg.sender;
    }

    // 检查合约调用者是否是合约创建者的modifier
    modifier onlyOwner() {
        require(msg.sender == i_owner, "Not ownner");
        // 下划线是用在modifier中的特殊字符，它告诉solidity继续执行函数中的其他代码
        _;
    }

    // modifier可以传参
    // 下例展示modifier检查传入的地址不能为0地址
    modifier validAddress(address _addr) {
        require(_addr != address(0), "Not valid address!");
        _;
    }

    // 使用modifier
    function changeOwner(address newOwner) public onlyOwner validAddress(newOwner) {
        owner = newOwner;
    }

    // modifier可以在函数执行前后执行
    // 下例的modifier防止函数在执行过程中再次被执行
    modifier noReentrancy() {
        require(!locked, "No reentrancy");
        locked = true;
        _;
        locked = false;
    }

    function decrement(uint i) public noReentrancy {
        x -= i;

        if( i > 1) {
            decrement(i - 1);
        }
    }
}
```

## 事件 events

事件允许在区块链中记录 log，使用事件的场景:

- 监听事件并更新用户界面（前端）
- 是一种便宜的存储方式

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Events {
    // 声明事件
    // 最多3个参数可以被定义成indexed
    // indexed参数可以被用来过滤log
    event Log(address indexed sender, string message);
    event AnotherLog();

    function test() public {
        emit Log(msg.sender, "Hello, first event!");
        emit Log(msg.sender, "Hi, EVM");
        emit AnotherLog();
    }
}
```

## 构造函数 constructor

构造函数是可选函数，这个函数在合约创建的时候运行  
展示构造函数传参的例子

```solidity
//SPDX-License-Identifier:MIT

pragma solidity ^0.8.17;

// 基合约X
contract X {
    string public name;
    constructor (string memory _name) {
        name = _name;
    }

}

// 基合约Y
contract Y {
    string public text;
    constructor (string memory _text) {
        text = _text;
    }

}

// 两种初始化父合约的方式
// 在合约继承列表中传递参数
contract B is X("Input to X"), Y("Input to Y") {

}

// 在子合约构造函数中传递参数
contract C is X,Y {
    constructor(string memory _name, string memory _text) X(_name) Y(_text) {

    }
}

// 父合约构造函数调用顺序按照继承顺序，不按照子合约构造函数后面赋值的顺序

// 下例构造函数调用顺序
// X->Y->D
contract D is X, Y {
    constructor() X("X was called") Y("Y was called") {}
}

// 下例构造函数调用顺序
// X->Y->E
contract E is X, Y {
    constructor() Y("Y was called") X("X was called") {}
}
```

## 继承 inheritance

solidity 支持多继承，合约通过使用 is 关键字继承其他合约  
可能会被子合约重写的函数，要声明称 virtual  
子合约中重写父合约的函数必须使用 override 关键字  
继承的顺序很重要，要从最基础的到最衍生的顺序列出继承的父合约列表

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

/* 继承图
    A
   / \
  B   C
 / \ /
F  D,E
 */

contract A {
    function foo() public pure virtual returns(string memory){
        return "A";
    }
}

// 使用is关键字继承其他合约
contract B is A{
    // 重写A的foo函数
    function foo() public pure virtual override returns(string memory) {
        return "B";
    }
}

contract C is A {
    //重写A.foo()
    function foo() public pure virtual override returns(string memory) {
        return "C";
    }
}

// 合约可以继承自多个父合约
// 当要调用的函数在多个父合约中都定义过，父合约的搜索方式按照从右向左，深度优先的方式
contract D is B, C {
    // D.foo() 返回"C"
    // 因为C是最右边且饱含foo的父合约
    function foo() public pure virtual override(B, C) returns(string memory) {
        return super.foo();
    }
}

contract E is C, B {
    // E.foo() 返回"B"
    // 因为B是最右边且饱含foo的父合约
    function foo() public pure virtual override(C, B) returns(string memory) {
        return super.foo();
    }
}

// 继承顺序一定要是最基类到最衍生的顺序
// 交换AB的顺序会抛出编译错误
contract F is A, B {
    function foo() public pure virtual override(A, B) returns(string memory) {
        return super.foo();
    }
}

```

## 隐藏继承的状态变量

不像函数，状态变量不能通过重新在子类中声明同名状态变量复写  
下例展示如何在子类中重写父类的状态变量

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract A {
    string public name = "Contract A";
    function getName() public view returns(string memory) {
        return name;
    }
}

contract C is A {
    // 正确的重写继承环境变量的方式
    constructor() {
        name = "Contract C";
    }

    // C.getName() 会返回"Contract C"
}
```

## 调用父合约

父合约可以被直接调用，也可以使用 super 关键字  
使用 super 关键字，所有的直接父合约都会被调用

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

/* 继承树
   A
 /  \
B   C
 \ /
  D
*/

contract A {
    // 这里定义一个事件，可以在函数中触发事件，触发的事件会在交易log中记录log，对跟踪函数调用有帮助
    event Log(string message);

    function foo() public virtual {
        emit Log("A.foo called");
    }

    function bar() public virtual {
        emit Log("A.bar called");
    }
}

contract B is A{
    function foo() public virtual override {
        emit Log("B.foo called");
        A.foo();
    }

    function bar() public virtual override {
        emit Log("B.bar called");
        super.bar();
    }
}

contract C is A {
    function foo() public virtual override {
        emit Log("C.foo called");
        A.foo();
    }

    function bar() public virtual override {
        emit Log("C.bar called");
        super.bar();
    }
}

contract D is B, C {
    // D.foo只会调用C和A的foo
    function foo() public virtual override(B, C) {
        super.foo();
    }

    // D.bar()会调用 C->B->A的bar
    function bar() public virtual override(B, C) {
        super.bar();
    }
}

```

## 可视性 visibility

函数和状态变量要声明它们是否可以被其他合约访问  
函数可以被声明成

- public 任何合约和账户都可以访问调用
- private 只在声明函数的合约内部可访问
- internal 只在继承内部函数的合约内部可以访问（相对于 private，多了继承子合约）
- external 只有其他合约和账户可以访问

状态变量可以是 public/private/internal 但不能是 external

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Base {
    // private函数只能在这个合约内部调用，不能在继承这个合约的合约内调用
    function privateFun() private pure returns(string memory) {
        return "private function called";
    }

    function testPrivateFunc() public pure returns(string memory) {
        return privateFun();
    }

    // internal函数可以在这个函数内部和继承了这个合约的合约内部访问
    function internalFun() internal pure returns(string memory) {
        return "internal function called";
    }

    function testInternalFunc() public pure virtual returns(string memory) {
        return internalFun();
    }

    // public函数可以被这个合约/继承了这个合约的合约/其他合约和账户访问
    function publicFun() public pure returns(string memory) {
        return "public function called";
    }

    // external函数只能被其他合约和账户访问
    function externalFun() external pure returns(string memory) {
        return "external function called";
    }

    // 这个函数不会被编译，external函数不能在这个合约内部调用
    // function testExternalFun() public pure returns(string memory) {
    //     return externalFun();
    // }

    // 状态变量
    string public publicVar = "public var";
    string private privateVar = "private var";
    string internal internalVar = "internal var";
    //不能声明external状态变量
    //string external externalVar = "external var";

}

contract Child is Base {
    // 不能访问父类的private函数和状态变量
    // function testPrivateFunc() public pure returns(string memory) {
    //     return privateFun();
    // }

    // 可以访问父类的internal函数和状态变量
    function testInternalFunc() public pure virtual override returns(string memory) {
        return internalFun();
    }
}
```

## 接口 interface

你可以声明接口和其他合约交互  
接口

- 不能有函数实现
- 可以继承自其他接口
- 所有声明的函数都必须是 external
- 不能声明构造函数 constructor
- 不能声明状态变量

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Counter {
    uint public count;

    function increment() external {
        count += 1;
    }
}

interface ICounter {
    function count() external view returns(uint);
    function increment() external;
}

contract MyContract {

    //通过传入实现了接口的合约地址，调用接口函数
    function incrementCounter(address _counter) external {
        ICounter(_counter).increment();
    }

    function getCount(address _counter) external view returns(uint) {
        return ICounter(_counter).count();
    }
}

// Uniswap 例子
interface UniswapV2Factory {
    function getPair(address tokenA, address tokenB) external view returns (address pair);
}

interface UniswapV2Pair {
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
}

contract UniswapExample {
    address private factory = 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f;
    address private dai = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address private weth = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;

    function getTokenReserves() external view returns (uint, uint) {
        address pair = UniswapV2Factory(factory).getPair(dai, weth);
        (uint reserve0, uint reserve1, ) = UniswapV2Pair(pair).getReserves();
        return (reserve0, reserve1);
    }
}

```

## 可支付 payable

函数或者地址声明成 payable，可以接收 ether 到合约

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Payable {
    // payable的地址可以接收ether
    address payable public owner;

    // payable的构造函数可以接收ether
    constructor() payable {
        owner = payable(msg.sender);
    }

    // 向合约里存钱的函数, 调用这个函数的时候带一些ether，这个合约账户的余额会自动增加
    function deposite() public payable {

    }

    // 如果带一些ether调用这个函数，会抛错，因为这个函数不是payable
    function notPayable() public {}

    // 从当前合约取走所有余额
    function withdraw() public {
        //获取合约账户的余额
        uint amount = address(this).balance;

        // 发送所有的ether到owner，因为owner的地址是payable的，所以owner可以收到ether
        (bool success,) = owner.call{value: amount}("");
        require(success, "Fail to send Ether");
    }

    // 转移指定余额到指定账户
    function transfer(address payable _to, uint amount) public {
        // _to被声明成payable
        (bool success,) = _to.call{value: amount}("");
        require(success, "Fail to send Ether");
    }
}
```

## 发送 Ether(transfer,send,call)

**如何发送 Ether**  
你可以向其他账户发送 Ether,通过以下方法：

- transfer (2300gas, 发送失败抛出错误)
- send (2300gas, 返回 bool)
- call (发送所有的 gas 或设置的 gas，返回 bool)

**如何接收 Ether**  
 一个要接收 Ether 的合约，至少要有一个以下函数：

- receive() external payable
- fallback() external payable  
  当 msg.data 为空时，调用 receive()，否则调用 fallback()  
  **应该使用哪个函数发送 Ether**  
  call 中集成了防止重入攻击，自 2019 年 12 月后推荐使用该方法  
  防重入攻击，通过：
- 在调用其他合约前修改状态
- 使用防重入攻击的 modifier

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract ReceiveEther {
    /*
    调用哪个函数, fallback() or receive()?

           发送 Ether
               |
         msg.data 是空?
              / \
            yes  no
            /     \
receive() 存在?  fallback()
         /   \
        yes   no
        /      \
    receive()   fallback()
    */

    // 接收ether的函数，msg.data必须为空
    receive() external payable {}

    // 当msg.data不为空时调用fallback
    fallback() external payable {}

    function getBalance() public view returns(uint) {
        return address(this).balance;
    }
}

contract SendEther {
    function sendViaTransfer(address payable _to) public payable {
        // transfer函数已经不再推荐使用
        _to.transfer(msg.value);
    }

    function sendViaSend(address payable _to) public payable {
        // send 返回bool表示转账成功和失败
        // 这个函数也已经不再推荐使用
        bool sent = _to.send(msg.value);
        require(sent, "Fail to send");
    }

    function sendViaCall(address payable _to) public payable {
        // call 返回bool值指示转账成功或失败
        // 目前推荐使用的方法
        (bool success, bytes memory data) = _to.call{value: msg.value}("");
        require(success, "Fail to send");
    }
}
```

## fallback 函数

fallback 是一个特殊的函数，它在以下场景执行：

- 被调用的函数不存在
- Ether 直接发送到合约账户并且合约中没有 receive()，或者 msg.data 不为空

当被 transfer, send 调用时，fallback 有 2300 的 gas 限制

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Fallback {
    event Log(string func, uint gas);

    // fallback一定要声明称external
    fallback() external payable {
        // transfer/send (发送2300gas到这个fallback函数)
        // call (发送所有gas费)
        emit Log("fallback", gasleft());
    }

    // receive() 是fallback的变形，当msg.data为空时调用
    receive() external payable {
        emit Log("receive", gasleft());
    }

    function getBalance() public view returns(uint) {
        return address(this).balance;
    }

}

contract SendToFallback {
    function transferToFallback(address payable _to) public payable {
        _to.transfer(msg.value);
    }

    function callFallback(address payable _to) public payable {
        (bool success,) = _to.call{value: msg.value}("");
        require(success, "Fail to send");
    }
}
```

fallback 可以携带 bytes 作为输入输出

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract FallbackInputOutput {
    address public immutable target;
    constructor(address _target) {
        target = _target;
    }

    fallback(bytes calldata data) external payable returns(bytes memory) {
        (bool success, bytes memory res) = target.call{value: msg.value}(data);
        require(success, "Fail to send");
        return res;
    }
}

contract TestFallbackInputOutput {
    event Log(bytes res);

    function test(address _fallback, bytes calldata data) external {
        (bool ok, bytes memory res) = _fallback.call(data);
        require(ok, "call failed");
        emit Log(res);
    }
}
```

## Call 函数

call 是和其他合约交互的底层函数  
是当发送 Ether 给其他合约时推荐使用的函数  
但是不推荐使用 call 调用已经存在的函数  
**不推荐使用 call 调用已存在函数的几点原因**

- 回滚不会向上传递提示
- 绕过类型检查
- 函数存在性检查被忽略

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Receiver {
    event Received(address caller, uint amount, string message);

    fallback() external payable {
        emit Received(msg.sender, msg.value, "Fallback called");
    }

    function foo(string memory _text, uint _x) public payable returns(uint) {
        emit Received(msg.sender, msg.value, _text);
        return _x + 1;
    }
}

contract Caller {
    event Response(bool success, bytes data);

    // 合约的调用者不知道合约接收者的源码，但是知道合约接收者的地址和要调用的函数
    function testCallFoo(address payable _addr) public payable {
        //可以在发送ether时传入自定义gas量
        (bool success, bytes memory data) = _addr.call{value: msg.value, gas:5000}(abi.encodeWithSignature("foo(string,uint256)", "call foo", 123));

        emit Response(success, data);
    }

    // 调用不存在的函数，触发fallback
    function testCallNotExit(address payable _addr) public payable {
        (bool success, bytes memory data) = _addr.call{value: msg.value}(abi.encodeWithSignature("doesntExit()"));

        emit Response(success, data);
    }
}
```

## Delegatecall

delegatecall 是和 call 相似的底层函数  
当合约 A 对合约 B 调用 delegatecall，B 的代码会使用 A 的存储,msg.sender 和 msg.value 运行

```solidity
 // SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

// 先部署该合约
contract B {
    // 该合约的存储布局要和A一样
    uint public num;
    address public sender;
    uint public value;

    function setVars(uint _num) public payable {
        num = _num;
        sender = msg.sender;
        value = msg.value;
    }
}

contract A {
    uint public num;
    address public sender;
    uint public value;

    function setVars(address _contract, uint num) public payable {
        // A的存储被修改，B的未被修改
        (bool success, bytes memory data) = _contract.delegatecall(abi.encodeWithSignature("setVars(uint256)", num));
    }
}
```

## 函数选择器 function selector

当一个函数被调用时，calldata 的前四个 bytes 决定是哪个函数被调用  
这 4 bytes 就是函数选择器  
如下例，在一个合约中使用 call 去执行地址 addr 的 transfer 函数

```solidity
addr.call(abi.encodeWithSignature("transfer(address, uint256)", 0x一些地址, 123));
```

abi.encodeWithSignature(...)返回的前四个 bytes 就是函数选择器

函数选择器如何运算的例子：

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract FunctionSelector {
    function getSelector(string calldata _func) external pure returns(bytes4) {
        return bytes4(keccak256(bytes(_func)));
    }
}
```

## 调用其他合约

有两种方式调用其他合约, 最简单的就是直接调用，比如 A.foo(x, y, z); 另一种方式就是使用 call（不推荐）

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Callee {
    uint public x;
    uint public value;

    function setX(uint _x) public returns(uint) {
        x = _x;
        return x;
    }

    function setXAndEther(uint _x) public payable returns(uint, uint) {
        x = _x;
        value = msg.value;
        return (x, value);
    }
}

contract Caller {
    function setX(Callee _callee, uint _x) public{
        uint x = _callee.setX(_x);
    }

    function setXFromAddress(address _addr, uint _x) public {
        Callee callee = Callee(_addr);
        callee.setX(_x);
    }

    function setXAndEther(Callee callee, uint _x) public payable{
        (uint x, uint v) = callee.setXAndEther{value:msg.value}(_x);
    }
}
```

## 创建其他合约的合约

合约可以在其他合约中使用 new 关键字创建  
从 0.8.0 开始，new 关键字可以通过制定 salt 来支持 create2 特性

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Car {
    address public owner;
    string public model;
    address public carAddress;

    constructor(address _owner, string memory _model) payable {
        owner = _owner;
        model = _model;
        carAddress = address(this);
    }
}

contract CarFactory {
    Car[] public cars;

    function create(address _owner, string memory _model) public {
        Car car = new Car(_owner, _model);
        cars.push(car);
    }

    function createAndSendEther(address _owner, string memory _model) public payable {
        Car car = (new Car){value: msg.value}(_owner, _model);
        cars.push(car);
    }

    // create2就是可以通过设置盐值，在部署合约之前，创建合约的时候就可以预知合约地址
    function create2(address _owner, string memory _model, bytes32 _salt) public {
        Car car = (new Car){salt: _salt}(_owner, _model);
        cars.push(car);
    }

    function create2AndSendEther(address _owner, string memory _model, bytes32 _salt) public payable {
        Car car = (new Car){value: msg.value, salt: _salt}(_owner, _model);
        cars.push(car);
    }

    function getCar(uint index) public view returns(address owner, string memory model, address carAddr, uint balance) {
        Car car = cars[index];
        return (car.owner(), car.model(), car.carAddress(), address(car).balance );
    }
}
```

## try/catch 异常捕获

try/catch 只能捕获外部调用时抛出的错误，以及创建合约时抛出的错误

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Foo {
    address public owner;

    constructor(address _owner) {
        require(_owner != address(0), "invalid address");
        assert(_owner != 0x0000000000000000000000000000000000000001);
        owner = _owner;
    }

    function myFunc(uint x) public pure returns (string memory) {
        require(x != 0, "require failed");
        return "my func was called";
    }
}

contract Bar {
    event Log(string message);
    event LogBytes(bytes data);

    Foo public foo;

    constructor() {
        foo = new Foo(msg.sender);
    }

    // try catch用于外部函数调用
    function tryCatchExternalCall(uint _x) public {
        try foo.myFunc(_x) returns(string memory result) {
            // 调用成功
            emit Log(result);
        } catch {
            // 调用失败
            emit Log("External call failed!");
        }
    }

    // try catch 用于创建合约
    function tryCatchNewContract(address _owner) public {
        try new Foo(_owner) returns(Foo foo) {
            emit Log("Foo created");
        } catch Error(string memory reason){
            // 创建失败， require返回的字符串
            emit Log(reason);
        } catch (bytes memory reason) {
            // 创建失败，assert返回
            emit LogBytes(reason);
        }
    }
}
```

## import 导入

可以在 solidity 中使用 import 导入本地和外部文件  
**本地导入**  
文件结构

```
|--Import.sol
|--Fool.sol
```

Foo.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

struct Point {
    uint x;
    uint y;
}

error Unauthorized(address caller);

function add(uint x, uint y) pure returns (uint) {
    return x + y;
}

contract Foo {
    string public name = "Foo";
}
```

Import.sol

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

// 从当前路径导入Foo.sol
import "./Foo.sol";

// 导入本地文件中的部分内容
import {Unauthorized, add as func, Point} from "./Foo.sol";

contract Import {
    // 初始化一个Foo
    Foo public foo = new Foo();

    // 测试Foo
    function getFooName() public view returns(string memory) {
        return foo.name();
    }
}
```

**外部导入**  
可以通过复制 github 中合约的 url 直接导入 github 文件

```solidity
// https://github.com/owner/repo/blob/branch/path/to/Contract.sol
import "https://github.com/owner/repo/blob/branch/path/to/Contract.sol";

// https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.5/contracts/utils/cryptography/ECDSA.sol
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.5/contracts/utils/cryptography/ECDSA.sol";
```

## Library

库 library 和合约相似，但是不能声明任何状态变量，不能发送 Ether  
如果所有的库函数都是内部的，库会内嵌到合约中，否则必须要在合约部署之前部署库并链接到合约

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

library Math {
    function sqrt(uint x) internal pure returns(uint z) {
        if (y > 3) {
            z = y;
            uint x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }
}

contract TestMath {
    function testSqrt(uint x) public returns(uint) {
        return Math.sqrt(x);
    }
}

library Array {
    function remove(uint[] storage arr, uint index) public {
        require(arr.length > 0, "Can't remove from empty array");
        arr[index] = arr[arr.length - 1];
        arr.pop();
    }
}

contract TestArray {
    // 链接库
    using Array for uint[];

    uint[] public arr;

    function testArrayRemove() public {
        for (uint i = 0; i < 3; i++) {
            arr.push(i);
        }

        arr.remove(1);

        assert(arr.length == 2);
        assert(arr[0] == 0);
        assert(arr[1] == 2);
    }
}
```

## ABI Encode

将数据编码称 bytes

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface IERC20 {
    function transfer(address, uint) external;
}

contract Token {
    function transfer(address, uint) external {}
}

contract AbiEncode {
    function test(address _contract, bytes calldata data) external {
        (bool ok, ) = _contract.call(data);
        require(ok, "call failed");
    }

    function encodeWithSignature(
        address to,
        uint amount
    ) external pure returns (bytes memory) {
        // 不检查拼写错误 - "transfer(address, uint)"
        return abi.encodeWithSignature("transfer(address,uint256)", to, amount);
    }

    function encodeWithSelector(
        address to,
        uint amount
    ) external pure returns (bytes memory) {
        // 类型不检查 - (IERC20.transfer.selector, true, amount)
        return abi.encodeWithSelector(IERC20.transfer.selector, to, amount);
    }

    function encodeCall(address to, uint amount) external pure returns (bytes memory) {
        // 拼写错误和类型错误都会在编译时检查
        return abi.encodeCall(IERC20.transfer, (to, amount));
    }
}
```

## ABI Decode

abi.decode 解码 bytes 到原数据

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract AbiDecode {
    struct MyStruct{
        string name;
        uint[2] nums;
    }

    function encode(uint x, address addr, uint[] calldata arr, MyStruct calldata myStruct) public pure returns(bytes memory) {
        return abi.encode(x, addr, arr, myStruct);
    }

    function decode(bytes calldata data) public pure returns(uint x, address addr, uint[] calldata arr, MyStruct calldata myStruct) {
        (x, addr, arr, myStruct) = abi.decode(data, (uint, address, uint[], Mystruct));
    }
}
```

## 使用 Keccak256 哈希

keccak256 计算输入的 Keccak-256 哈希  
使用场景：

- 创建输入的确定且唯一的 Id
- 提交-显示主题
- 紧凑的加密签名（签名哈希，而不是长输入）

```solidity
// SPDX-License-Identifier:MIT

pragma solidity ^0.8.17;

contract HashFunction{
    function hash(string memory _text, uint num, address _addr) public pure returns(bytes32) {
        return keccak256(abi.encodePacked(_text, num, _addr));
    }

    // 哈希碰撞的例子
    // 当传入abi.encodePacked()多个动态数据类型值时，有可能会出现哈希碰撞，这种情况下要替换使用abi.encode
    function collision(string memory _text, string memory _anotherText) public pure returns(bytes32) {
        // encodePacked(AAA, BBB) -> AAABBB
        // encodePacked(AA, ABBB) -> AAABBB
        return keccak256(abi.encodePacked(_text, _anotherText));
    }

}

contract GuessTheMagicWord {
    bytes32 public answer =
        0x60298f78cc0b47170ba79c10aa3851d7648bd96f2f8e46a19dbc777c36fb0c00;

    // Magic word is "Solidity"
    function guess(string memory _word) public view returns (bool) {
        return keccak256(abi.encodePacked(_word)) == answer;
    }
}
```

## 验证签名

消息在链下签名，在链上使用智能合约验证签名

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

/* 签名验证

如何签名和验证
# 签名
1. 创建要签名的消息
2. 对消息进行哈希
3. 对哈希进行签名 (链下要保护好自己的私钥)

# 验证
1. 对原始消息重建哈希
2. 从签名和哈希恢复出签名者
3. 比较恢复出的签名者和声称的签名者
*/
contract VerifySignature {
    /* 1. 要有MetaMask账户
    ethereum.enable()
    */

    /* 2. 哈希的消息
    getMessageHash(
        0x14723A09ACff6D2A60DcdF7aA4AFf308FDDC160C,
        123,
        "coffee and donuts",
        1
    )

    hash = "0xcf36ac4f97dc10d91fc2cbb20d718e94a8cbfe0f82eaedc6a4aa38946fb797cd"
    */

    function getMessageHash(
        address _to,
        uint _amount,
        string memory _message,
        uint _nonce
    ) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(_to, _amount, _message, _nonce));
    }

    /* 3. 签名消息哈希
    # 使用浏览器
    account = "这里写自己钱包账户地址"
    ethereum.request({ method: "personal_sign", params: [account, hash]}).then(console.log)

    # 使用web3
    web3.personal.sign(hash, web3.eth.defaultAccount, console.log)
    */
    function getEthSignedMessageHash(
        bytes32 _messageHash
    ) public pure returns (bytes32) {
        /*
        签名使用keccak256对以下格式的消息进行哈希，产生签名:
        "\x19Ethereum Signed Message\n" + len(msg) + msg
        */
        return
            keccak256(
                abi.encodePacked("\x19Ethereum Signed Message:\n32", _messageHash)
            );
    }

    /* 4. 验证签名
    signer = 0xB273216C05A8c0D4F0a4Dd0d7Bae1D2EfFE636dd
    to = 0x14723A09ACff6D2A60DcdF7aA4AFf308FDDC160C
    amount = 123
    message = "coffee and donuts"
    nonce = 1
    signature =
        0x993dab3dd91f5c6dc28e17439be475478f5635c92a56e17e82349d3fb2f166196f466c0b4e0c146f285204f0dcb13e5ae67bc33f4b888ec32dfe0a063e8f3f781b
    */
    function verify(
        address _signer,
        address _to,
        uint _amount,
        string memory _message,
        uint _nonce,
        bytes memory signature
    ) public pure returns (bool) {
        bytes32 messageHash = getMessageHash(_to, _amount, _message, _nonce);
        bytes32 ethSignedMessageHash = getEthSignedMessageHash(messageHash);

        return recoverSigner(ethSignedMessageHash, signature) == _signer;
    }

    function recoverSigner(
        bytes32 _ethSignedMessageHash,
        bytes memory _signature
    ) public pure returns (address) {
        (bytes32 r, bytes32 s, uint8 v) = splitSignature(_signature);

        return ecrecover(_ethSignedMessageHash, v, r, s);
    }

    function splitSignature(
        bytes memory sig
    ) public pure returns (bytes32 r, bytes32 s, uint8 v) {
        require(sig.length == 65, "invalid signature length");

        assembly {
            /*
            前32个字节跳过，存储签名长度

            add(sig, 32) = 指针指向 sig + 32
            跳过签名的前32字节

            mload(p) 将接下来的32字节内存中的内容加载到内存中
            */

            // 前缀之后的32bytes
            r := mload(add(sig, 32))
            // 上面之后的 32 bytes
            s := mload(add(sig, 64))
            // 最后一个字节 (上面之后的32字节的第一个字节)
            v := byte(0, mload(add(sig, 96)))
        }
    }
}
```

## Gas 费节省技巧

一些节省 Gas 费的技巧

- 用 calldata 替代 memory
- 加载状态变量到内存
- 替换循环中的 i++为++i
- 缓存数组元素
- 短路

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract GasGolf {
    // 初始 - 50860 gas
    // 使用calldata - 49115 gas
    // 加载状态变量到内存 - 48904 gas
    // 短路 - 48586 gas
    // 循环变量++i - 48214 gas
    // 缓存数组长度 - 48179 gas
    // 加载数组元素到内存 - 48113 gas
    // uncheck i 溢出 - 47393 gas

    uint public total;

    // start - 没有gas优化
    // function sumIfEvenAndLessThan99(uint[] memory nums) external {
    //     for (uint i = 0; i < nums.length; i += 1) {
    //         bool isEven = nums[i] % 2 == 0;
    //         bool isLessThan99 = nums[i] < 99;
    //         if (isEven && isLessThan99) {
    //             total += nums[i];
    //         }
    //     }
    // }

    // gas 优化
    // [1, 2, 3, 4, 5, 100]
    function sumIfEvenAndLessThan99(uint[] calldata nums) external {
        uint _total = total;
        uint len = nums.length;

        for (uint i = 0; i < len; ) {
            uint num = nums[i];
            if (num % 2 == 0 && num < 99) {
                _total += num;
            }
            unchecked {
                ++i;
            }
        }

        total = _total;
    }
}
```

## 位运算符

- ｜ 按位或
- & 按位与
- ^ 按位异或
- ～ 按位取反
- << bits 左移
- \>\> bits 右移

## unchecked 不检查溢出

在 solidity 0.8.0 之后，变量上下溢出会抛出异常，使用 unchecked 可以忽略溢出检查，不报错  
unchecked 可以节省 gas 费，见 gas 费节费那一节的例子

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

contract UncheckedMath {
    function add(uint x, uint y) external pure returns (uint) {
        // 22291 gas
        // return x + y;

        // 22103 gas
        unchecked {
            return x + y;
        }
    }

    function sub(uint x, uint y) external pure returns (uint) {
        // 22329 gas
        // return x - y;

        // 22147 gas
        unchecked {
            return x - y;
        }
    }

    function sumOfCubes(uint x, uint y) external pure returns (uint) {
        // Wrap complex math logic inside unchecked
        unchecked {
            uint x3 = x * x * x;
            uint y3 = y * y * y;

            return x3 + y3;
        }
    }
}
```
