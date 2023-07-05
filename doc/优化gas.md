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
- calldata 比 memory 便宜
- internal 以入参引用的方式传递，不会复制入参到 memoy

**使用 library**

- 因为使用 library 中的函数逻辑，不会增加部署合约的代码量，比自己在合约中实现这个功能节省 gas
- library 只需要部署一次，就可以多次使用

**多用短路（&&/｜｜）**

**释放 storage,回收 gas**

- 不用的状态变量使用 delete 回收 storage

**简短 error 信息或使用自定义 error**
