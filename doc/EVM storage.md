**stack**

- LIFO
- 宽 256bit 深 1024
- EVM 的操作都在 stack 中进行
- EVM 操作最多只能对 stack 中的最上 16 个元素进行操作，因此一个函数中超过 16 个局部变量会编译报错
- 可以从 memory 和 storage 中读写数据

**memory**

- 可以以 8bit/256bit 大小写入，但是读的时候每次读一个 256bit 的块
- 主要用来存放动态变量，string/array 等

**stotage**

- key-value 存储(256bit-256bit)
- key 表示变量在账户存储中的 slot
- 智能合约中的变量会根据它们在合约中的位置分配给状态变量 slot
- 如果多个状态变量可以挤进一个 slot,那它们就按顺序放到一个 slot 中
- 通过 provider.getStorage(contractAddress, slotNumber)读取变量值
- 在 delegatecall 中被调用合约中对状态变量的使用，使用的都是在原合约中对应 slot 位置的变量，无关类型
