##   web3j基本开发环境搭建及常用查询函数

### 1.环境准备

 - 安装 Java Development Kit (JDK)， 在Windows上，还需要设置JAVA_HOME环境变量，以便在VS Code中正确识别JDK
```  
java -version
```  
 - 安装 Maven， Maven是一个用于构建Java项目的工具。可以从Maven官网下载并安装它
```  
mvn -version
```  
 - 安装Visual Studio Code   
 - 在vs code中安装以下插件：  
   Extension Pack for Java: 该插件包括许多有用的Java开发工具，如语法高亮、代码自动完成和调试支持等。  
   Maven for Java: 支持从Maven原型生成项目， 支持产生有效的POM等  
   Debugger for Java: 支持Java的调试  
   
 - 创建maven项目， 
 在VS Code中，右键"Create Maven Project",  再输入group id, artifact id等

### 2. 安装web3j
```  
# 在pom.xml中添加以下依赖
<!-- web3j库 -->
<dependency>
    <groupId>org.web3j</groupId>
    <artifactId>core</artifactId>
    <version>4.9.7</version>
</dependency>
<!-- gson库 -->
<dependency>
  <groupId>com.google.code.gson</groupId>
  <artifactId>gson</artifactId>
  <version>2.9.0</version>
</dependency>
```  

### 3. web3j常用查询和基础转账
 web3j文档： https://docs.web3j.io/4.9.7/getting_started/run_node_locally/
 
 - Web3j.build(new HttpService("URL")): 建立rpc连接
 - web3.ethBlockNumber(): 获取区块高度
 - web3.ethAccounts(): 获取链上账户信息
 - web3.ethGetBalance("地址", DefaultBlockParameterName.LATEST): 获取账户余额
 - Convert.fromWei(balance.toString(), Convert.Unit.ETHER).toString()： 格式化余额
 - web3.ethChainId(): 获取chainid
 - web3.ethGetBlockByNumber(): 根据区块号获取区块信息
 - web3.ethGetTransactionByHash("交易hash"): 根据交易hash获取交易内容
 - web3.ethGasPrice()： 获取gasprice
 - Credentials.create("用户私钥"): 根据私钥获取转账凭证
 - Convert.toWei("转账数量", Convert.Unit.ETHER).toBigInteger()： 格式化转帐金额
 - RawTransaction.createEtherTransaction(nonce, gasPrice, gasLimit, to, value)： 创建交易对象
 - TransactionEncoder.signMessage(交易对象, chainid, 转账凭证)： 签名交易
 - web3.ethSendRawTransaction()： 交易上链
 

## web3j合约部署和读写

### 1. 合约编写
- 编写一个ERC1155的智能合约，继承了OpenZeppelin ERC1155和Ownable合约
- 合约中有两个私有映射变量 _tokenSupply 和 _tokenURIs，分别用于存储每个ERC1155代币的供应量和URI
- 在构造函数中，传入了基础URI，用于确定代币的元数据
- mint 函数可供所有者调用来铸造新的ERC1155代币。该函数调用了 _mint 函数将新代币的供应量添加到特定帐户中
- _tokenSupply 和 _tokenURIs 映射都会更新，并触发 TokenMinted 事件。 该事件记录了代币 ID，代币供应量和代币 URI。tokenId 参数使用 indexed 修饰符进行标记，以便在事件日志中可以更快速地搜索并访问该参数。 
```  
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/token/ERC1155/ERC1155.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v4.3.0/contracts/access/Ownable.sol";

contract MyERC1155 is ERC1155, Ownable {

    mapping(uint256 => uint256) private _tokenSupply;
    mapping(uint256 => string) private _tokenURIs;

    event TokenMinted(uint256 indexed tokenId, uint256 supply, string uri);

    constructor(string memory uri) ERC1155(uri) {}

    function mint(uint256 tokenId, uint256 supply, string memory uri) public onlyOwner {
        _mint(msg.sender, tokenId, supply, "");
        _tokenSupply[tokenId] += supply;
        _tokenURIs[tokenId] = uri;
        emit TokenMinted(tokenId, supply, uri);
    }

    function tokenSupply(uint256 tokenId) public view returns (uint256) {
        return _tokenSupply[tokenId];
    }

    function uri(uint256 tokenId) public view override returns (string memory) {
        return _tokenURIs[tokenId];
    }
}
```  

### 2. 合约编译
编译工具: 
- [[在线remix]](https://remix.ethereum.org/)
- solc工具

编译结果
- 字节码(bytecode): 是solidity代码被翻译以后的信息，包含了二进制的计算机指令。
- ABI: 应用程序二进制接口,以json文件表示。

### 3. 合约部署和读写操作
 - Keys.createEcKeyPair(): 生成eckeypair,用于下面的私钥和地址生成
 - Numeric.toHexStringNoPrefix(eckeypair.getPrivateKey()): 生成私钥
 - Numeric.toHexStringNoPrefix(eckeypair.getPublicKey()): 生成公钥
 - Credentials.create(eckeypair).getAddress(): 获取地址
 - FunctionEncoder.encodeConstructor: 合约构造函数encode
 - RawTransaction.createContractTransaction: 创建部署合约交易对象
 - RawTransaction.createTransaction: 创建调用合约交易对象
 - Transaction.createEthCallTransaction：查询合约方法

 具体使用可以参考同级目录下的: \src\main\java\com\DeployAndCall.java

##   web3j事件监听


### 1. 查询合约历史事件  
web3j.ethGetLogs(filter)
过滤器对象：
- fromBlock: 起始区块（最小值支持从1开始）
- toBlock: 终止区块， 这个参数不带代表一直到最大区块
- address: 合约地址
- event: 事件名和参数

示例： 
```  
// 要查询的事件
Event event = new Event("TokenMinted", Arrays.asList(new TypeReference<Uint256>() {},
        new TypeReference<Uint256>() {},
        new TypeReference<Utf8String>() {}));

EthFilter filter = new EthFilter(
                DefaultBlockParameterName.EARLIEST, // 日志的起始区块
                DefaultBlockParameterName.LATEST, // 日志的结束区块
                contractAddress // 查询的智能合约地址
);

// 添加event类型及其参数
filter.addSingleTopic(EventEncoder.encode(event));

// 获取符合过滤器条件的所有日志
EthLog log = web3j.ethGetLogs(filter).send();
```  

### 2. 订阅区块链中的指定事件  
web3j.ethLogFlowable(filter).subscribe(log -> {})

示例：
```  
// 要监听的事件
Event event = new Event("TokenMinted", Arrays.asList(new TypeReference<Uint256>() {},
        new TypeReference<Uint256>() {},
        new TypeReference<Utf8String>() {}));

// 过滤开始区块，结束区块，合约地址
EthFilter filter = new EthFilter(
    DefaultBlockParameterName.EARLIEST, DefaultBlockParameterName.LATEST, contractAddress);
        
filter.addSingleTopic(EventEncoder.encode(event));

web3j.ethLogFlowable(filter).subscribe(log -> {

    System.out.println(log.getTopics());

    // 解码非index参数
    List<Type> results = FunctionReturnDecoder.decode(log.getData(), event.getNonIndexedParameters());
    BigInteger value = (BigInteger) results.get(0).getValue();
    System.out.println("supply is: " +  value);
});
```  
