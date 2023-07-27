# Solidity application

## Ether wallet 钱包

一个基本钱包的例子

- 任何人都可以发送 ETH
- 只有钱包的拥有者可以取出余额

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.18;

contract EtherWallet {
    address private immutable owner;

    constructor() {
        owner = msg.sender;
    }

    receive() payable external {}

    function withdraw(uint amount) external{
        require(msg.sender == owner, "Only owner can withdraw");
        (bool success,) = owner.call{value: amount}("");
        require(success, "Withdraw fail!");
    }

    function getBalance() external view returns(uint) {
        return address(this).balance;
    }
}
```

## 多重签名钱包

下例创建一个多重签名的钱包，满足以下规格：  
钱包的拥有者可以执行以下操作：

- 提交一笔交易
- 赞同或者撤销一笔正在等待运行的交易
- 任何人都以在一笔交易有足够多的确认后，运行这笔交易

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

error MultiSigWallet__ownersIsEmpty();
error MultiSigWallet__InvalidNumConfirmationRequired();
error MultiSigWallet__NotOwner();
error MultiSigWallet__CannotExecute(uint txIndex);

contract MultiSigWallet {
    address[] private i_owners; // 创建多重签名合约的时候指定都有谁是合约的拥有者
    mapping(address => bool) private isOwner;  // 判断某个地址是否已经在i_owners中了
    uint256 private immutable numConfirmationsRequired; // 创建合约的时候指定交易需要被确认的最小次数
    mapping(uint => mapping(address => bool)) private isConfirmed; // 判断某个owner是否已经确认过某个交易

    struct Transaction {
        address to;
        uint value;
        bytes data;
        bool executed;
        uint numConfirmed;
    }

    Transaction[] private transactions;

    event Deposite(address indexed sender, uint amount, uint balance);
    event SubmitTransaction(address indexed owner, uint indexed txIndex, address indexed to, uint value, bytes data);
    event ConfirmTransaction(address indexed owner, uint indexed txIndex);
    event ExecuteTransaction(address indexed owner, uint indexed txIndex);
    event RevokeConfirmation(address indexed owner, uint txIndex);

    constructor(address[] memory _owners, uint _numConfirmationRequired) {
        if(_owners.length <= 0) {
            revert MultiSigWallet__ownersIsEmpty();
        }
        if(_numConfirmationRequired <= 0 || _numConfirmationRequired > _owners.length) {
            revert MultiSigWallet__InvalidNumConfirmationRequired();
        }

        for(uint i = 0; i < _owners.length; i++) {
            address owner = _owners[i];

            require(owner != address(0), "Invalid owner!");
            require(!isOwner[owner], "Owner not unique!");

            isOwner[owner] = true;
            i_owners.push(owner);
        }

        numConfirmationsRequired = _numConfirmationRequired;
    }

    receive() external payable {
        emit Deposite(msg.sender, msg.value, address(this).balance);
    }

    modifier onlyOwner() {
        if(!isOwner[msg.sender]) {
            revert MultiSigWallet__NotOwner();
        }
        _;
    }

    modifier txExists(uint txIndex) {
        require(txIndex < transactions.length, "tx not exists");
        _;
    }

    modifier notExecuted(uint txIndex) {
        require(!transactions[txIndex].executed, "tx already executed!");
        _;
    }

    modifier notConfirmed(uint txIndex) {
        require(!isConfirmed[txIndex][msg.sender], "You already confirmed this tx!");
        _;
    }

    // Owner可以提交交易
    function submitTransaction(address _to, uint _value, bytes calldata _data) public onlyOwner {
        uint txIndex = transactions.length;
        transactions.push(Transaction({to: _to, value: _value, data: _data, executed: false, numConfirmed: 0}));
        emit SubmitTransaction(msg.sender, txIndex, _to, _value, _data);
    }

    // 确认交易
    function confirmTransaction(uint txIndex) public onlyOwner txExists(txIndex) notExecuted(txIndex) notConfirmed(txIndex) {
        Transaction storage transaction = transactions[txIndex];
        transaction.numConfirmed += 1;
        isConfirmed[txIndex][msg.sender] = true;

        emit ConfirmTransaction(msg.sender, txIndex);
    }

    // 执行交易
    function executeTransaction(uint txIndex) public onlyOwner txExists(txIndex) notExecuted(txIndex) {
        Transaction storage transaction = transactions[txIndex];
        if(transaction.numConfirmed < numConfirmationsRequired) {
            revert MultiSigWallet__CannotExecute(txIndex);
        }
        transaction.executed = true;
        (bool success,) = transaction.to.call{value: transaction.value}(transaction.data);
        require(success, "tx failed");

        emit ExecuteTransaction(msg.sender, txIndex);
    }

    // 撤回确认
    function revokeConfirmation(uint txIndex) public onlyOwner txExists(txIndex) notExecuted(txIndex) {
        Transaction storage transaction = transactions[txIndex];
        require(isConfirmed[txIndex][msg.sender], "You have not confirmed this tx!");

        transaction.numConfirmed -= 1;
        isConfirmed[txIndex][msg.sender] = false;

        emit RevokeConfirmation(msg.sender, txIndex);
    }

    // getter函数们
    function getOwners() public view returns(address[] memory) {
        return i_owners;
    }

    function getTransactionCount() public view returns(uint) {
        return transactions.length;
    }

    function getTransaction(uint txIndex) public view returns(address to, uint value, bytes memory data, bool executed, uint numConfirmations) {
        Transaction storage transaction = transactions[txIndex];
        return (transaction.to, transaction.value, transaction.data, transaction.executed, transaction.numConfirmed);
    }


}

// 测试上面合约发送交易
contract TestMulitWallet {
    uint public i;

    function callMe(uint j) payable public {
        i += j;
    }

    // 获取call时携带的data
    function getData() public pure returns(bytes memory) {
        return abi.encodeWithSignature("callMe(uint256)", 1234);
    }
}
```

## 默克尔树 Merkle Tree

默克尔树允许我们可以以加密的方式证明一个元素是否包含在一个集合中，不用查看整个集合就可以做到  
依次两两向上 hash，验证 root hash 和区块中保存的是否一致  
所有的数据 hash 作为叶子结点，非叶子结点为其下节点两两 hash 组成

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract MerkelProof {
    function verify(bytes32[] memory proof, bytes32 leaf, bytes32 root, uint index) public pure returns(bool) {
        bytes32 computedHash = leaf;
        uint len = proof.length;
        for(uint i = 0; i < len; i++) {
            if(index % 2 == 0) {
                computedHash = keccak256(abi.encodePacked(computedHash, proof[i]));
            } else {
                computedHash = keccak256(abi.encodePacked(proof[i], computedHash));
            }
            index /= 2;
        }

        return computedHash == root;
    }
}

contract CreateMerkel is MerkelProof{
    bytes32[] public hashes;

    // 构造默克尔树 [ "alice -> bob",   "bob -> dave",  "carol -> alice", "dave -> bob" ]
    constructor(string[] memory transactions) {
        uint len = transactions.length;
        for(uint i = 0; i < len; i++) {
            hashes.push(keccak256(abi.encodePacked(transactions[i])));
        }

        uint n = len;
        uint offset = 0;
        while(n > 0) {
            for(uint i = 0; i < n - 1; i += 2) {
                hashes.push(keccak256(abi.encodePacked(hashes[offset + i], hashes[offset + i + 1])));
            }
            offset += n;
            n /= 2;
        }
    }

    // 查询默克尔树根
    function getRoot() public view returns(bytes32){
        return hashes[hashes.length - 1];
    }

    // verify
    // 第三个叶子： 0xdca3326ad7e8121bf9cf9c12333e6b2271abe823ec9edfe42f813b1e768fa57b
    // index 2
    // proof: 第四个叶子和12叶子的组合hash(index3 和 index 4)
    //[0x8da9e1c820f9dbd1589fd6585872bc1063588625729e7ab0797cfc63a00bd950,0x995788ffc103b987ad50f5e5707fd094419eb12d9552cc423bd0cd86a3861433]
}
```

## 可迭代的 mapping

mapping 类型是不能迭代遍历的，本例创建一个可以迭代遍历的 mapping
让 mapping 可以像数组一样，根据下标来索引

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

library IterableMapping {
    // 构造一个存储mapping信息的结构体，方便查询
    struct Map {
        address[] keys;
        mapping(address => uint) values; // 正常mapping功能，key-->value
        mapping(address => uint) indexOf; // 记录key对应的下标
        mapping(address => bool) inserted; // 记录key是否已经存在
    }

    // 根据key查询value
    function get(Map storage map, address key) public view returns(uint) {
        return map.values[key];
    }

    // 获取指定下标的key
    function getKeyAt(Map storage map, uint index) public view returns(address) {
        return map.keys[index];
    }

    // 获取map的大小
    function size(Map storage map) public view returns(uint) {
        return map.keys.length;
    }

    // 添加或更新键值对
    function set(Map storage map, address key, uint value) public {
        if(map.inserted[key]) {
            map.values[key] = value;
        } else {
            map.keys.push(key);
            map.values[key] = value;
            map.indexOf[key] = map.keys.length - 1;
            map.inserted[key] = true;
        }
    }

    // 删除键值对
    function remove(Map storage map, address key) public {
        if(map.inserted[key]) {
            uint index = map.indexOf[key];
            map.keys[index] = map.keys[map.keys.length -1];
            map.keys.pop();

            delete map.values[key];
            delete map.indexOf[key];
            delete map.inserted[key];
        } else {
            return;
        }
    }
}

contract TestIterableMapping {
    using IterableMapping for IterableMapping.Map;

    IterableMapping.Map private map;

    function test() public {
        map.set(address(0), 0);
        map.set(address(10), 10);
        map.set(address(20), 30);
        map.set(address(20), 20);
        map.set(address(30), 30);

        for(uint i = 0; i < map.size(); i++) {
            address key = map.getKeyAt(i);
            assert(map.get(key) == i * 10);
        }

        map.remove(address(20));
        assert(map.size() == 3);
        assert(map.getKeyAt(0) == address(0));
        assert(map.getKeyAt(1) == address(10));
        assert(map.getKeyAt(2) == address(30));
    }
}
```

## ERC20

任何遵守 ERC20 标准的合约都是 ERC20 代币  
ERC20 代币提供以下功能：

- 转移代币
- 允许其他人代表代币持有人转移代币  
  ERC20 接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface IERC20 {
    function totalSupply() external view returns (uint);

    function balanceOf(address account) external view returns (uint);

    function transfer(address recipient, uint amount) external returns (bool);

    function allowance(address owner, address spender) external view returns (uint);

    function approve(address spender, uint amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint value);
    event Approval(address indexed owner, address indexed spender, uint value);
}
```

手动实现一个简单的 ERC20 token

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import "./IERC20.sol";

contract MyERC20 is IERC20 {
    // 代币名字
    string public name = "My ERC20";
    // 代币标识
    string public symbol = "ME";
    // 代币使用的位数，1token = 1*10**18
    uint8 public decimals = 18;
    // 代币总发行量
    uint public totalSupply;
    // 账户余额
    mapping(address => uint) balanceof;
    // 某个账户允许其他账户在自己账户提取的最大金额
    mapping(address => mapping(address => uint)) public _allowance;


    function balanceOf(address account) external view returns (uint){
        return balanceof[account];
    }

    // 转移amount数量的代币到recipient账户地址，并处罚Transfer事件
    // 调用者余额不足要抛出错误
    function transfer(address recipient, uint amount) external returns (bool){
        require(balanceof[msg.sender] >= amount, "You dont have enough token");
        balanceof[msg.sender] -= amount;
        balanceof[recipient] += amount;

        emit Transfer(msg.sender, recipient, amount);
        return true;
    }

    function allowance(address owner, address spender) external view returns (uint){
        return _allowance[owner][spender];
    }

    // 允许 spender从调用者的账户多次提取，提取总额度小于amount
    function approve(address spender, uint amount) external returns (bool){
        _allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);

        return true;
    }

    // 调用者从sender账户转移amount到recipient账户
    function transferFrom(
        address sender,
        address recipient,
        uint amount
    ) external returns (bool){
        require(_allowance[sender][msg.sender] >= amount, "Allow amount less than amount");
        _allowance[sender][msg.sender] -= amount;
        balanceof[sender] -= amount;
        balanceof[recipient] += amount;
        emit Transfer(sender, recipient, amount);
        return true;
    }

    // 申领token
    function mint(uint amount) public {
        balanceof[msg.sender] += amount;
        totalSupply += amount;
        emit Transfer(address(0), msg.sender, amount);
    }

    // 销毁token
    function burn(uint amount) public {
        balanceof[msg.sender] -= amount;
        totalSupply -= amount;
        emit Transfer(msg.sender, address(0), amount);
    }


}
```

使用 openzeppelin 合约可以很容易地创建自己的 ERC20 合约

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

// Remix中可以直接从github地址导入合约
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol";

contract OERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        // 调用ERC20中的mint 100个token
        _mint(msg.sender, 100 * 10 ** uint(decimals()));
    }

}
```

**交换代币合约**

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/IERC20.sol";

/**
例子：A有一些token1,B有一些token2, A或者B运行这个合约，可以从A账户转移10到B账户，从B账户转移20到A账户
前提： token1 approve了TokenSwap合约地址可以从A账户取10，token2 approve了TokenSwap合约地址可以从B账户取20

*/
contract TokenSwap {
    IERC20 public immutable token1;
    IERC20 public immutable token2;
    address public immutable owner1;
    address public immutable owner2;
    uint public immutable amount1;
    uint public immutable amount2;

    // 0x417Bf7C9dc415FEEb693B6FE313d1186C692600F,0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2,0x9bF88fAe8CF8BaB76041c1db6467E7b37b977dD7,0x5B38Da6a701c568545dCfcB03FcB875f56beddC4,10,20
    constructor(address _token1, address _owner1, address _token2, address _owner2, uint _amount1, uint _amount2) {
        token1 = IERC20(_token1);
        token2 = IERC20(_token2);
        owner1 = _owner1;
        owner2 = _owner2;
        amount1 = _amount1;
        amount2 = _amount2;
    }

    function swap() public {
        // 只能owner1或者owner2可以调用改方法
        require(msg.sender == owner1 || msg.sender == owner2, "Not Authorized!");
        // approve当前合约可以从owner1和owner2取对应的amount
        require(token1.allowance(owner1, address(this)) >= amount1 && token2.allowance(owner2, address(this)) >= amount2, "Token allowance less!");
        _safeTransferFrom(token1, owner1, owner2, amount1);
        _safeTransferFrom(token2, owner2, owner1, amount1);

    }

    function _safeTransferFrom(IERC20 token, address sender, address recipient, uint amount) private {
        bool success = token.transferFrom(sender, recipient, amount);
        require(success, "transfer failed!");
    }

}
```

## ERC721

非同质化代币标准（每一个代币都是独一无二，与众不同的）

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

// ERC165: 检查智能合约是否实现了某些接口，它只提供supportsInterface函数，调用者传入接口ID就可查询该智能合约是否支持某些接口
interface IERC165 {
    function supportsInterface(bytes4 interfaceID) external view returns (bool);
}

// ERC721的标准接口，继承ERC165
interface IERC721 is IERC165 {
    // 查询某个账户owner持有的NFT个数
    function balanceOf(address owner) external view returns (uint balance);

    // 查询指定tokenId的NFT的拥有者（每个NFT由它的tokenID唯一标识）
    function ownerOf(uint tokenId) external view returns (address owner);

    // 安全转移【当to为合约地址时，要求合约实现ERC721Receiver】，将指定tokenID的NFT从账户from转移到账户to
    function safeTransferFrom(address from, address to, uint tokenId) external;

    function safeTransferFrom(
        address from,
        address to,
        uint tokenId,
        bytes calldata data
    ) external;

    // 普通转移，将指定tokenID的NFT从账户from转移到账户to
    function transferFrom(address from, address to, uint tokenId) external;

    // 允许地址to可以使用指定tokenId的NFT
    function approve(address to, uint tokenId) external;

    // 查询指定tokenId的NFT被授权给了哪个地址使用
    function getApproved(uint tokenId) external view returns (address operator);

    // 允许或不允许地址operator操作自己所有的NFT
    function setApprovalForAll(address operator, bool _approved) external;

    // 查询某个账户owner是否授权过operator可以操作他的所有NFT
    function isApprovedForAll(
        address owner,
        address operator
    ) external view returns (bool);
}

// safeTransferFrom会检查目标地址是否实现了以下接口，只有实现了以下接口的合约才能接受NFT,不然直接revert
// 这个接口包含一个函数，返回receiver的函数选择器
interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint tokenId,
        bytes calldata data
    ) external returns (bytes4);
}

// ERC721主合约
contract ERC721 is IERC721{
    // 账户-》拥有的NFT数量
    mapping(address => uint) internal _balanceOf;
    // tokenId => owner
    mapping(uint => address) internal _ownerOf;
    // tokenId => 授权地址
    mapping(uint => address) internal _approves;
    // owner => operator => 是否授权所有
    mapping(address => mapping(address => bool)) internal _isApprovedAll;


    // 需要包含的事件
    // NFT转账事件
    event Transfer(address indexed from, address indexed to, uint indexed id);
    // NFT授权事件
    event Approval(address indexed owner, address indexed spender, uint indexed id);
    // 授权所有NFT事件
    event ApprovalForAll(
        address indexed owner,
        address indexed operator,
        bool approved
    );

    // 查询某个账户owner持有的NFT个数
    function balanceOf(address owner) external view returns (uint balance) {
        require(owner != address(0), "Invalid owner address");
        balance = _balanceOf[owner];
    }

    // 查询指定tokenId的NFT的拥有者（每个NFT由它的tokenID唯一标识）
    function ownerOf(uint tokenId) external view returns (address owner) {
        owner = _ownerOf[tokenId];
        require(owner != address(0), "TokenId not exist");
    }

    // 查询指定tokenId的NFT被授权给了哪个地址使用
    function getApproved(uint tokenId) external view returns (address operator) {
        require(_ownerOf[tokenId] != address(0), "tokenId not exist");
        operator = _approves[tokenId];
    }

    // 允许或不允许地址operator操作自己所有的NFT
    function setApprovalForAll(address operator, bool _approved) external {
        _isApprovedAll[msg.sender][operator] = _approved;
        emit ApprovalForAll(msg.sender, operator, _approved);
    }

    // 查询某个账户owner是否授权过operator可以操作他的所有NFT
    function isApprovedForAll(
        address owner,
        address operator
    ) external view returns (bool) {
        return _isApprovedAll[owner][operator];
    }

    // 允许地址to可以使用指定tokenId的NFT
    function approve(address to, uint tokenId) external {
        address owner = _ownerOf[tokenId];
        require(owner == msg.sender || _isApprovedAll[owner][msg.sender], "Not Authorized");
        _approves[tokenId] = to;

        emit Approval(owner, to, tokenId);
    }

    // 普通转移，将指定tokenID的NFT从账户from转移到账户to
    function transferFrom(address from, address to, uint tokenId) public {
        address owner = _ownerOf[tokenId];
        require(owner == from, "from do not have tokenId");
        // 操作者必须满足：是tokenId持有者或者被授权过这个tokenId或者被授权过所有的tokenId
        require(msg.sender == owner || _approves[tokenId] == msg.sender || _isApprovedAll[owner][msg.sender], "Not autorized");
        _balanceOf[from] -= 1;
        _ownerOf[tokenId] = to;
        _balanceOf[to] += 1;

        delete _approves[tokenId];
        emit Transfer(from, to, tokenId);
    }


    // 安全转移【当to为合约地址时，要求合约实现ERC721Receiver】，将指定tokenID的NFT从账户from转移到账户to
    function safeTransferFrom(address from, address to, uint tokenId) public {
        transferFrom(from, to, tokenId);
        //判断目标地址：不是合约地址 或者 是合约地址且实现了IERC72Receiver接口
        require(to.code.length == 0 ||
        IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, "") == IERC721Receiver.onERC721Received.selector, "unsafe received");
    }

    function safeTransferFrom(
        address from,
        address to,
        uint tokenId,
        bytes calldata data
    ) public {
        transferFrom(from, to, tokenId);
        //判断目标地址：不是合约地址 或者 是合约地址且实现了IERC72Receiver接口
        require(to.code.length == 0 ||
        IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, data) == IERC721Receiver.onERC721Received.selector, "unsafe received");
    }

    function supportsInterface(bytes4 interfaceID) external pure returns (bool){
        // 实现IERC721和IERC165接口
        return interfaceID == type(IERC721).interfaceId || interfaceID == type(IERC165).interfaceId;
    }

    function _mint(address to, uint tokenId) internal {
        require(to != address(0), "Invalid mint address");
        require(_ownerOf[tokenId] == address(0), "already minted");

        _ownerOf[tokenId] = to;
        _balanceOf[to] += 1;

        emit Transfer(address(0), to, tokenId);

    }

    function _burn(uint id) internal {
        address owner = _ownerOf[id];
        require(owner != address(0), "not minted");

        _balanceOf[owner] -= 1;

        delete _ownerOf[id];
        delete _approves[id];

        emit Transfer(owner, address(0), id);
    }

}

contract MyERC721 is ERC721 {
    uint private tokenId;
    function mint() public {
        _mint(msg.sender,tokenId);
        tokenId++;
    }
}
```

## 使用 create2 提前计算合约地址

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Factory {
    function deploy(address _owner, uint _foo, bytes32 _salt) public returns(address) {
        return address(new TestContract{salt: _salt}(_owner, _foo));
    }
}

contract TestContract {
    address public owner;
    uint public foo;

    constructor(address _owner, uint _foo) payable {
        owner = _owner;
        foo = _foo;
    }

    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

## 代理合约

solidity 合约部署到链上之后，是不能进行修改的，如果有 bug 需要修复就需要创建一个新的合约，进行数据迁移的话就需要花费大量的 gas 费用，可以使用代理合约来解决合约 bug 修复和更新的问题。  
大体思路就是：

- 代理合约指向一个逻辑运行合约，代理合约里存放状态变量，通过 delegatecall 调用逻辑运行合约，逻辑运行合约会执行并修改代理合约的状态变量
- 当逻辑运行合约有 bug 需要修复更新的时候，重新部署一个新的逻辑合约，并让代理合约重新指向新的逻辑合约即可，原来的数据状态变量也不用迁移，节省 gas
- 调用者只需要调用代理合约即可

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Proxy {
    // 存储当前指向的逻辑运行合约
    address public implementation;
    uint public count;



    constructor(address _implementation) {
        implementation = _implementation;
    }

    function upgradeTo(address _implementation) external {
        implementation = _implementation;
    }

    function _delegate() internal {
        assembly {
            // 获取指向的逻辑合约地址，位于storage0位置
            let _implementation := sload(0)

            // calldatacopy(t, f, s) - 从calldata的f位置，复制s bytes到内存位置t
            //  calldatasize() - calldata的大小bytes
            // 复制整个calldata到内存0位置
            calldatacopy(0, 0, calldatasize())

            // 通过delegatecall调用逻辑合约的方法
            // delegatecall(g, a, in, insize, out, outsize) -
            // - 调用地址a的逻辑运行合约（这里是_implementation）
            // - 输入内存范围[in…(in+insize)) -- 这里是0-calldatasize
            // - g gas费
            // - 输出内存范围[out…(out+outsize)) -- 0,0
            // - 返回0表示有错误(例如gas用光)， 返回1调用成功
            let result := delegatecall(gas(), _implementation, 0, calldatasize(), 0, 0)

            // 复制返回值.
            // returndatacopy(t, f, s) - 从f返回值的位置f复制s bytes到内存t位置
            // returndatasize() - 返回值的大小（bytes）
            returndatacopy(0, 0, returndatasize())

            switch result
            // delegatecall 返回0表示有错误.
            case 0 {
                // revert(p, s) - 回滚，并返回返回数据
                revert(0, returndatasize())
            }
            default {
                // return(p, s) - 返回返回数据
                return(0, returndatasize())
            }
        }
    }

    // fallback(),当调用者调用逻辑运行合约的任意方法时，都跳到fallback捕获
    fallback() external payable {
       _delegate();
    }
}

contract CounterV1 {
    // 切记切记，逻辑运行合约的状态变量一定要和代理合约保持一致，不然就会有问题
    address public implementation;
    uint public count;

    event Increment(uint count);
    function inc() external {
        count += 1;
        emit Increment(count);
    }
}

contract CounterV2 {
    address public implementation;
    uint public count;

    function inc() external {
        count += 1;
    }

    function dec() external {
        count -= 1;
    }
}

contract Caller{
    address public proxy; // 代理合约地址
    constructor(address proxy_){
        proxy = proxy_;
    }

    // 通过代理合约调用 inc()函数
    function inc() external returns(uint) {
        (bool success,) = proxy.call(abi.encodeWithSignature("inc()"));
        require(success, "call failed");
    }
}
```

## 部署任意合约

通过合约的二进制编码部署合约

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

contract Proxy {
    event Deploy(address deployedAddress);

    // 通过合约的bytes code(合约的创建码)部署指定合约
    function deploy(bytes memory _code) external payable returns(address addr) {
        assembly {
            //create(v, p, n)
            //v = 发送的eth
            //p = 内存中执行代码开始位置的指针
            //n = 代码大小
            addr := create(callvalue(), add(_code, 0x20), mload(_code))
        }
        require(addr != address(0), "Invalid address!");
        emit Deploy(addr);

    }

    // 执行部署完的合约方法
    function execute(address _addr, bytes memory _data) external payable {
        (bool success,) = _addr.call{value: msg.value}(_data);
        require(success, "call failed!");
    }
}

contract TestContract1 {
    address public owner = msg.sender;

    event Log(address owner);
    function setOwner(address _owner) public {
        require(msg.sender == owner, "not owner");
        owner = _owner;
        emit Log(owner);
    }
}

contract TestContract2 {
    address public owner = msg.sender;
    uint public value = msg.value;
    uint public x;
    uint public y;

    event Log(uint a, uint b);

    constructor(uint _x, uint _y) payable {
        x = _x;
        y = _y;
        emit Log(x, y);
    }
}

contract Helper {
    function getBytecode1() external pure returns (bytes memory) {
        bytes memory bytecode = type(TestContract1).creationCode;
        return bytecode;
    }

    function getBytecode2(uint _x, uint _y) external pure returns (bytes memory) {
        bytes memory bytecode = type(TestContract2).creationCode;
        // 有构造函数的合约，需要同时打包构造函数入参
        return abi.encodePacked(bytecode, abi.encode(_x, _y));
    }

    function getCalldata(address _owner) external pure returns (bytes memory) {
        return abi.encodeWithSignature("setOwner(address)", _owner);
    }
}
```

## 变量写入任意一个存储位置

solidity 存储是一个长度为 2^256 的数组，每一个卡槽可以存放 32bytes 的数据  
状态变量声明的顺序以及状态变量的类型会决定它用哪个卡槽  
我们可以使用 assembly，将状态变量写进任意一个卡槽

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

library StorageSlot {
    // 定义一个slot的结构体，描述一个slot,存放值value
    struct AddressSlot {
        address value;
    }

    function getAddressSlot(bytes32 slot) public returns(AddressSlot storage pointer) {
        assembly {
            // 返回一个指定的slot给这个结构体
            pointer.slot := slot
        }
    }
}

contract TestSlot {
    // 一个solot
    bytes32 public constant TEST_SLOT = keccak256("TEST_SLOT");

    //  slot里写值
    function write(address _addr) external {
        // 设置data的slot为TEST_SLOT
        StorageSlot.AddressSlot storage data = StorageSlot.getAddressSlot(TEST_SLOT);
        // 这个slot里放指定地址
        data.value = _addr;
    }

    // 获取slot里的值
    function get() external returns (address) {
        StorageSlot.AddressSlot storage data = StorageSlot.getAddressSlot(TEST_SLOT);
        return data.value;
    }
}
```

## 单向支付通道

支付通道允许付收双方在链下多次进行 ether 交易  
如何使用这个例子的合约：

- A 部署这个合约，并向合约里存入一些 ether
- A 在链下通过给一条支付 ether 消息签名认证一笔交易，然后把这个签名发送给 B
- B 可以拿着这个签名给智能合约取出这笔 ether
- 当合约超时，即使 B 没有取走他的 ether, A 也可以撤回他在合约里存的所有资产  
  之所以叫单向支付通道，是因为支付流动永远是从 A 到 B

部署以下合约使用 metamask 账户：  
签名：浏览器中执行分配 account 和 hash(通过 gethash 获取) ethereum.request({ method: "personal_sign", params: [account, hash]}).then(console.log)

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.17;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol";

contract UniDirectionPayment is ReentrancyGuard {
    using ECDSA for bytes32; // 使用ECDSA签名算法

    address payable public sender; // 合约创建者，A
    address payable public receiver; // 拿签名取钱的B

    uint public expiresAt; // 合约超时
    uint public constant DURATION = 7 * 24 * 60 *60; // 合约存在时长

    constructor (address _receiver) payable{
        require(_receiver != address(0), "Invalid address!");
        sender = payable(msg.sender);
        receiver = payable(_receiver);
        expiresAt = block.timestamp + DURATION;
    }

    // 获取消息哈希
    function _getHash(uint _amount) private view returns (bytes32) {
        return keccak256(abi.encodePacked(address(this), _amount));
    }
    // 对外可见
    function getHash(uint _amount) public view returns (bytes32) {
        return _getHash(_amount);
    }

    // 获取消息哈希的签名
    function _getEthSignedHash(uint _amount) private view returns(bytes32) {
        return _getHash(_amount).toEthSignedMessageHash();
    }

    // 对外可见
    function getEthSignedHash(uint _amount) public view returns(bytes32) {
        return _getEthSignedHash(_amount);
    }

    // 验证签名
    function _verify(uint _amount, bytes memory _sig) private view returns(bool, address) {
        return (_getEthSignedHash(_amount).recover(_sig) == sender, _getEthSignedHash(_amount).recover(_sig));
    }


    // 对外可见
    function verify(uint _amount, bytes memory _sig) public view returns(bool, address) {
        return _verify(_amount, _sig);
    }

    // B可以调用这个方法取ETH
    function close(uint _amount, bytes memory _sig) external nonReentrant {
        // 发送者要是receiver
        require(msg.sender == receiver, "Not receiver");
        // 签名验证要通过
        (bool verified,) = _verify(_amount, _sig);
        require(verified, "Invalid sig");
        // 还没有超时
        require(block.timestamp < expiresAt, "expired");

        (bool success,) = receiver.call{value: _amount}("");
        require(success, "Send fail");
    }

}

```

## 双向支付通道

双向支付通道，允许 A 和 B 转移 ether  
支付可以从 A 到 B，也可以从 B 到 A  
和单向支付差不多，多一个签名认证

## 英式拍卖

英式拍卖 NFT

**拍卖规则**

1. NFT 出售者部署合约
2. 拍卖持续 7 天
3. 拍卖参与者通过向合约存入大于当前最高价的 ETH 来竞标
4. 所有参与者都可以收回自己的出价（出价不是当前最高的时候可以）

**拍卖结束**

1. 最高出价者称为 NFT 新的拥有者
2. 出售者可以收到最高出价的 ETH

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface IERC721 {
    function safeTransferFrom(address from, address to, uint tokenId) external;

    function transferFrom(address, address, uint) external;
}

contract EnglishAuction {
    event Start();
    event Bid(address indexed sender, uint amount);
    event Withdraw(address indexed bidder, uint amount);
    event End(address winner, uint amount);

    IERC721 public nft;
    uint public nftId;

    address payable public seller;
    uint public endAt;
    bool public started;
    bool public ended;

    address public highestBidder;
    uint public highestBid;
    mapping(address => uint) public bids;

    constructor(address _nft, uint _nftId, uint _startingBid) {
        nft = IERC721(_nft);
        nftId = _nftId;

        seller = payable(msg.sender);
        highestBid = _startingBid;
    }

    function start() external {
        require(!started, "started");
        require(msg.sender == seller, "not seller");

        nft.transferFrom(msg.sender, address(this), nftId);
        started = true;
        endAt = block.timestamp + 7 days;

        emit Start();
    }

    function bid() external payable {
        require(started, "not started");
        require(block.timestamp < endAt, "ended");
        require(msg.value > highestBid, "value < highest");

        if (highestBidder != address(0)) {
            bids[highestBidder] += highestBid;
        }

        highestBidder = msg.sender;
        highestBid = msg.value;

        emit Bid(msg.sender, msg.value);
    }

    function withdraw() external {
        uint bal = bids[msg.sender];
        bids[msg.sender] = 0;
        payable(msg.sender).transfer(bal);

        emit Withdraw(msg.sender, bal);
    }

    function end() external {
        require(started, "not started");
        require(block.timestamp >= endAt, "not ended");
        require(!ended, "ended");

        ended = true;
        if (highestBidder != address(0)) {
            nft.safeTransferFrom(address(this), highestBidder, nftId);
            seller.transfer(highestBid);
        } else {
            nft.safeTransferFrom(address(this), seller, nftId);
        }

        emit End(highestBidder, highestBid);
    }
}
```

## 荷兰拍卖

荷兰拍卖 NFT  
**拍卖规则**

1. NFT 出售者部署合约，并设置起始价格
2. 拍卖持续 7 天
3. NFT 的价格随时间递减
4. 参与者可以以存入高于当前价格的 ETH 来买入 NFT
5. 当有人买入这个 NFT 后拍卖结束

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface IERC721 {
    function transferFrom(address _from, address _to, uint _nftId) external;
}

contract DutchAuction {
    uint private constant DURATION = 7 days;

    IERC721 public immutable nft;
    uint public immutable nftId;

    address payable public immutable seller;
    uint public immutable startingPrice;
    uint public immutable startAt;
    uint public immutable expiresAt;
    uint public immutable discountRate;

    constructor(uint _startingPrice, uint _discountRate, address _nft, uint _nftId) {
        seller = payable(msg.sender);
        startingPrice = _startingPrice;
        startAt = block.timestamp;
        expiresAt = block.timestamp + DURATION;
        discountRate = _discountRate;

        require(_startingPrice >= _discountRate * DURATION, "starting price < min");

        nft = IERC721(_nft);
        nftId = _nftId;
    }

    function getPrice() public view returns (uint) {
        uint timeElapsed = block.timestamp - startAt;
        uint discount = discountRate * timeElapsed;
        return startingPrice - discount;
    }

    function buy() external payable {
        require(block.timestamp < expiresAt, "auction expired");

        uint price = getPrice();
        require(msg.value >= price, "ETH < price");

        nft.transferFrom(seller, msg.sender, nftId);
        uint refund = msg.value - price;
        if (refund > 0) {
            payable(msg.sender).transfer(refund);
        }
        selfdestruct(seller);
    }
}
```
