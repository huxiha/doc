**相邻小变量最好能凑成一个 slot 32bytes**

- 节省 storage

3 个 slot

```solidity
uint8 num1;
uint256 num2;
uint8 num3;
uint8 num4;
uint8 num5;
```

2 个 slot

```solidity
uint8 num1;
uint8 num3;
uint8 num4;
uint8 num5;
uint256 num2;
```

**storage 比 memory 贵**

- 操作 storage 中的变量比操作 memory 中的变量贵，因此最好先在 memory 中修改变量，等逻辑都完成了再修改 storage

```solidity
contract A {
    uint public counter = 0;

    function count() {
        for(uint i = 0; i < 10; i++) {
            counter++;
        }
    }
}
```

操作 memory 先

```solidity
contract B {
    uint public counter = 0;

    function count() {
        uint copyCounter;
        for(uint i = 0; i < 10; i++) {
            copyCounter++;
        }
        counter = copyCounter;
    }
}
```

**如果可以的话尽可能使用固定长度的变量**

**如果可以尽量使用 external/internal 替换 public**

- external 存放函数入参在 calldata, public 存放函数入参在 memory
- calldata 比 memory 便宜 calldata 中的参数是只读的，不能修改
- internal 以入参引用的方式传递，不会复制入参到 memoy

**使用 library**

- 因为使用 library 中的函数逻辑，不会增加部署合约的代码量，比自己在合约中实现这个功能节省 gas
- library 只需要部署一次，就可以多次使用

**多用短路（&&/｜｜）**

**释放 storage,回收 gas**

- 不用的状态变量使用 delete 回收 storage

**简短 error 信息或使用自定义 error**

- 可以使用 natspec error 自定义 error 并使用///标注错误字符串，这样错误信息字符串不会在链上，节省 gas

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

contract ErrorDemo {
    // netspec error
    /// this is netspec error info,this is netspec error info,this is netspec error info,this is netspec error info,this is netspec error info
    error MyError1();

    /// 这是一个错误！老铁，你的输入参数错啦，必须要大于10的数字才可以通过！
    error MyError2();

    // 21647 gas
    function test1(uint256 _x) external pure {
        if (_x < 10) {
            revert MyError1();
        }
    }

    // 21691 gas
    function test2(uint256 _x) external pure {
        if (_x < 10) {
            revert MyError2();
        }
    }

    // 22036 gas
    function test3(uint256 _x) external pure {
        require(
            _x > 10,
            "this is netspec error info,this is netspec error info,this is netspec error info,this is netspec error info,this is netspec error info"
        );
    }

    // 21974 gas
    function test4(uint256 _x) external pure {
        require(
            _x > 10,
            unicode"这是一个错误！老铁，你的输入参数错啦，必须要大于10的数字才可以通过！"
        );
    }
}
```
