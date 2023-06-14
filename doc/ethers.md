## ethers

让我们可以访问区块链，读取区块链中的信息  
导入 ethers

```javascript
import { ethers } from "***";
```

## provider

访问区块链，首先需要一个网络提供者让我们可以进入区块链，从而可以获取区块链信息  
RPC provider 可以从 Alcomey 获取  
new ethers.jsonRpcProvider(RPC Provider);这样就有了 provider

**provider 提供的方法**  
getBalance(address); 获取账户余额  
getBlockNumber(); 查看当前区块数量  
getTransactionCount(address); 查看某个账户的交易次数  
getFeeData(); 查询当前建议 gas  
getBlock(uint); 查询某个区块的信息  
getCode(合约地址); 查询合约的 bytecode

**获取指定 slot 的值**  
provider.getStorageAt(contractAddress, slot)

## Contract 合约操作

**创建一个合约实例**  
只读合约：  
new ethers.Contract(`address`, `abi`, `provider`);  
contract.connect(signer); 只读变读写
读写合约：  
new ethers.Contract(`address`, `abi`, `signer`);  
然后就可以通过合约实例调用合约方法，读写

**模拟调用合约函数**  
在真实调用合约方法之前，先看下合约调用能否成功  
contract.函数名.staticCall(args, {override})

- args 调用函数的参数
- override 可选参数
  - from
  - value
  - blockTag
  - gasPrice
  - gasLimit
  - nonce

**获取合约接口**  
const interface = ethers.Interface(abi) 从 abi 获取  
const interface2 = contract.interface 从合约获取  
获取编码：
interface.getSighash(functionName/functionSign); 获取函数选择器  
interface.encodeDeploy(args); 编码构造器函数  
interface.encodeFunctionData(functionName, [args]); 编码 calldata
decodeFunctionResult(functionName, resultData); 解码函数返回结果

## Wallet 钱包操作

新建钱包：  
new ethers.Wallet(privateKey, provider);  
使用钱包发送交易：  
await wallet.sendTransaction(tx);

**钱包方法**  
getAddress(); 获取钱包地址  
privateKey; 获取钱包私钥  
getTransactionCount(); 获取钱包在链上交易次数

## 部署合约

new ethers.ContractFactory(abi, bytecode, signer); 合约工厂  
const contract = await contractFactory.deploy(args); 部署合约  
部署完之后就可以对合约进行读写操作

## 检索事件

await contract.queryFilter('事件名', 起始区块, 结束区块); 起始和结束区块是选填

## 监听事件

contract.on("eventName", function); 持续监听合约中的事件，监听到就执行 function  
contract.once("eventName", function); 只监听一次合约中的事件，监听到执行 function

**过滤事件**  
contract.filters.EVENT_NAME( ...args )

## utils 单位转换

ethers.getBigInt()函数将 string，number 等类型转换为 BigInt  
ethers.formatUnits(1000000000, "gwei") 转换单位 （小-》大）  
ethers.parseUnits("1.0").toString() 转换成 bigInt

## 数字签名

**消息哈希**  
ethers.solidityPackedKeccak256([类型 1, 类型 2...], [值 1, 值 2...])  
**签名**  
const messageHashBytes = ethers.getBytes(msgHash)
const signature = await wallet.signMessage(messageHashBytes)
