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
