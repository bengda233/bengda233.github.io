## 简介：
web3.js库是一个javascript库，是以太坊官方提供的和节点交互的JavaScript SDK (js-sdk) ，可以帮助智能合约开发者使用HTTP或者IPC与本地的或者远程的以太坊节点进行交互(Web3是对JSON RPC请求的封装)。web3.js主要包含下面几个库:

- web3-eth用来与以太坊区块链和智能合约交互。
- web3-shh用来控制whisper协议与p2ρ通信，以及广播

- web3-bzz用来与swarm协议交互。

- web3-utils包含了一些DApp开发应用的功能口



通过web3.js库，可以连接任何暴露了RPC层的以太坊节点

## 开始
### 前期准备：
首先需要将 web3 模块安装在项目中：npm install web3

创建一个 web3 的实例，设置一个 provider。（可以去infura、alchemy获得免费使用的节点，拿到API KEY，就可以访问以太坊网络节点。）

```
var Web3 = require('web3')
//用url链接
const rpcURL = "https://mainnet.infura.io/YOUR_INFURA_API_KEY"
const web3 = new Web3(rpcURL)
```
如果是本地：

```
var Web3 = require('web3');
var web3 = new Web3('http://localhost:8545');
```

查询节点信息
> web3.eth.getNodeInfo().then(
> console.log);
### 访问区块链网络


**读取余额**
```
const address = "0x03118E2c88676d31ee397E1eEf7789fECfbC40b9"
// 读取address中的余额，余额单位是wei
web3.eth.getBalance(address)
.then(console.log);
```

#### 异步回调（callback）

-   如果要发送异步请求，可以在函数的最后一个参数位置上，传入一个回调函数。回调函数是可选（optioanl）的
-   我们一般采用的回调风格是所谓的“错误优先


```
web3.eth.getBlock(48, function(error, result) { 
    if (!error) 
        console.log(JSON.stringify(result));
    else 
        console.error(error); });
```

异步读取余额

```
// 读取address中的余额，余额单位是wei

web3.eth.getBalance(address, (err, wei) => {

    // 余额单位从wei转换为ether
    balance = web3.utils.fromWei(wei, 'ether')
    console.log("balance: " + balance)
})
```

### 智能合约对象

对象可以使用`web3.eth.Contract()`函数获得，此函数需要2个参数: 智能合约ABI、智能合约地址。

```
const abi = [{"constant":true,"inputs":[],"name":"mintingFinished","outputs":[{"name":"","type":"bool"}],"payable":false,"type":"function"},{"constant":true,"inputs":[],"name":"name","outputs":{"indexed":false,"name":"value","type":"uint256"}],"name":"Approval","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"from","type":"address"},{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Transfer","type":"event"}]
const address = "0xd26114cd6EE289AccF82350c8d8487fedB8A0C07"
const contract = new web3.eth.Contract(abi, address)
```

### 调用智能合约读方法
**读函数**

智能合约对象的`methods`属性下，包含了对应智能合约的所有函数。要调用智能合约中的某个函数，例如`myFunction()`，可以使用`contract.methods.myFunction()`的方式调用。

> **注意** 这里调用方法只能调用读操作函数，不能调用写操作函数，写操作函数会改变区块链状态，调用写函数被视为交易。

```
contract.methods.totalSupply().call((err, result) => {   
console.log(result)   
})
contract.methods.name().call((err, result) => {
console.log(result) 
})
```

## 交易操作  
区块链中，交易操作是指会把数据写入区块链，改变区块链状态的操作。例如，以太币转账、调用写数据的智能合约函数，以及部署智能合约，这些操作都会改变区块链状态，被看作是交易。

**1.安装ethereumjs-tx**
>  npm install ethereumjs-tx

用这个库可以在本地签署交易，如果能在本地运行自己的以太坊节点，这样就可以不用使用ethereumjs-tx

**2.准备账号**
> web3.eth.accounts.create()

把私钥设为环境变量  
windows：  

```
 SET PRIVATE_KEY_1='b75e2bcaec74857cf9bb6636d66a04784d4c0fcfd908f4a2bc213428eba5af0d'
SET PRIVATE_KEY_2='ac0adfdbaeb0770a479e79aac78779d82fdc2f9262e0c8f751ae70fb63ef6196'
```

linux：

```
 export PRIVATE_KEY_1='b75e2bcaec74857cf9bb6636d66a04784d4c0fcfd908f4a2bc213428eba5af0d'
 export PRIVATE_KEY_2='ac0adfdbaeb0770a479e79aac78779d82fdc2f9262e0c8f751ae70fb63ef6196'
```

**声明账号变量，读取私钥**

```
const account1 = '0xf4Ab5314ee8d7AA0eB00b366c52cEEccC62d6B4B'
const pk1 = 'ac0adfdbaeb0770a479e79aac78779d82fdc2f9262e0c8f751ae70fb63ef6196'
// 实际项目中应该从process.env.PRIVATE_KEY_2中读取
const privateKey1 = Buffer.from(pk1, 'hex')
```

账户2同理

**3.创建交易：**

```
//构建交易对象
const txObject = {
    nonce:    web3.utils.toHex(txCount),
    to:       account2,
    value:    web3.utils.toHex(web3.utils.toWei('0.1', 'ether')),
    gasLimit: web3.utils.toHex(21000),
    gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei'))
  }
```

-   `nonce` – 这是账号的前一个交易计数。这个值必须是十六进制，可以使用Web3.js的`web3.utils.toHex()`转换。
-   `to` – 目标账户。
-   `value` – 要发送的以太币金额。这个值必须以十六进制表示，单位必须是wei。我们可以使用Web3.js工具`web3.utils.toWei()`转换单位。
-   `gasLimit` – 交易能消耗Gas的上限。像这样的基本交易总是要花费21000单位的Gas。
-   `gasPrice` – Gas价格，这里是 10 Gwei。

现在为`nonce`变量赋值，可以使用`web3.eth.getTransactionCount()`函数获取交易`nonce`。将构建交易对象的代码封装在一个回调函数中，如下所示:

```
web3.eth.getTransactionCount(account1, (err, txCount) => {
  const txObject = {
    nonce:    web3.utils.toHex(txCount),
    to:       account2,
    value:    web3.utils.toHex(web3.utils.toWei('0.1', 'ether')),
    gasLimit: web3.utils.toHex(21000),
    gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei'))
  }
})
```

**4.签署交易**

```
const tx = new Tx(txObject)
tx.sign(privateKey1)

const serializedTx = tx.serialize()
const raw = '0x' + serializedTx.toString('hex')
```

**5.广播交易**

```
web3.eth.sendSignedTransaction(raw, (err, txHash) => {
  console.log('txHash:', txHash)
})
```

## 部署智能合约  
部署智能合约也是一种交易操作，所以与交易操作步骤相同：

-   构建交易对象
-   签署交易
-   广播交易

**部署智能合约的区别在于交易对象的参数。**  
**构建交易**：

```
const txObject = {
  nonce:    web3.utils.toHex(txCount),
  gasLimit: web3.utils.toHex(1000000), // 提高Gas上限
  gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
  data: data
}
```

-   `nonce` – 这是账号的前一个交易计数。这个值必须是十六进制，可以使用Web3.js的`web3.utils.toHex()`转换。与[前面章节](https://www.qikegu.com/docs/5147)中的交易操作相同。
-   `to` – 目标账户。此参数不需要，部署智能合约没有目标账户。
-   `value` – 要发送的以太币金额。
-   `gasLimit` – 消耗Gas上限。部署合约比转账会消耗更多Gas，需要提高上限。
-   `gasPrice` – Gas价格，这里是 10 Gwei。与[前面章节](https://www.qikegu.com/docs/5147)中的交易操作相同。
-   `data` – 要部署的智能合约字节码。

**拿到合约的abi和字节码**

将bytecode里的object赋值给data  
设置`nonce`，可以使用`web3.eth.getTransactionCount()`函数获取交易`nonce`。将构建交易对象的代码封装在一个回调函数中，如下所示:

```
web3.eth.getTransactionCount(account1, (err, txCount) => {
  const data = "0x" +"608060405234801561001057600080fd5b506040518060400160405280600781526020017f6d7956616c7565000000000000000000000000000000000000000000000000008152506000908051906020019061005c929190610062565b50610107565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f106100a357805160ff19168380011785556100d1565b828001600101855582156100d1579182015b828111156100d05782518255916020019190600101906100b5565b5b5090506100de91906100e2565b5090565b61010491905b808211156101005760008160009055506001016100e8565b5090565b90565b61030f806101166000396000f3fe608060405234801561001057600080fd5b50600436106100365760003560e01c80634ed3885e1461003b5780636d4ce63c146100f6575b600080fd5b6100f46004803603602081101561005157600080fd5b810190808035906020019064010000000081111561006e57600080fd5b82018360208201111561008057600080fd5b803590602001918460018302840111640100000000831117156100a257600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f820116905080830192505050505050509192919290505050610179565b005b6100fe610193565b6040518080602001828103825283818151815260200191508051906020019080838360005b8381101561013e578082015181840152602081019050610123565b50505050905090810190601f16801561016b5780820380516001836020036101000a031916815260200191505b509250505060405180910390f35b806000908051906020019061018f929190610235565b5050565b606060008054600181600116156101000203166002900480601f01602080910402602001604051908101604052809291908181526020018280546001816001161561010002031660029004801561022b5780601f106102005761010080835404028352916020019161022b565b820191906000526020600020905b81548152906001019060200180831161020e57829003601f168201915b5050505050905090565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f1061027657805160ff19168380011785556102a4565b828001600101855582156102a4579182015b828111156102a3578251825591602001919060010190610288565b5b5090506102b191906102b5565b5090565b6102d791905b808211156102d35760008160009055506001016102bb565b5090565b9056fea265627a7a72315820dca9a73743312f852f1a355f75d5710294d8f878089664fabd9049813c9aa3f464736f6c63430005110032"
      const txObject1 = {
      nonce:    web3.utils.toHex(txCount),
      gasLimit: web3.utils.toHex(1000000),
      gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
      data: data
    }
  ```
  
 整体：

 ```
var Tx     = require('ethereumjs-tx').Transaction
const Web3 = require('web3')
const web3 = new Web3('https://ropsten.infura.io/v3/YOUR_INFURA_API_KEY')
const account1 = '0xf4Ab5314ee8d7AA0eB00b366c52cEEccC62d6B4B' // Your account address 1
const account2 = '0xff96B8B43ECd6C49805747F94747bFfa3A960b69' // Your account address 2
const pk1 = 'b75e2bcaec74857cf9bb6636d66a04784d4c0fcfd908f4a2bc213428eba5af0d' // 实际项目中应该从process.env.PRIVATE_KEY_1中读取
const pk2 = 'ac0adfdbaeb0770a479e79aac78779d82fdc2f9262e0c8f751ae70fb63ef6196' // 实际项目中应该从process.env.PRIVATE_KEY_2中读取
const privateKey1 = Buffer.from(pk1, 'hex')
const privateKey2 = Buffer.from(pk2, 'hex')
web3.eth.getTransactionCount(account1, (err, txCount) => {
  const data = "0x608060405234801561001057600080fd5b506040518060400160405280600781526020017f6d7956616c7565000000000000000000000000000000000000000000000000008152506000908051906020019061005c929190610062565b50610107565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f106100a357805160ff19168380011785556100d1565b828001600101855582156100d1579182015b828111156100d05782518255916020019190600101906100b5565b5b5090506100de91906100e2565b5090565b61010491905b808211156101005760008160009055506001016100e8565b5090565b90565b61030f806101166000396000f3fe608060405234801561001057600080fd5b50600436106100365760003560e01c80634ed3885e1461003b5780636d4ce63c146100f6575b600080fd5b6100f46004803603602081101561005157600080fd5b810190808035906020019064010000000081111561006e57600080fd5b82018360208201111561008057600080fd5b803590602001918460018302840111640100000000831117156100a257600080fd5b91908080601f016020809104026020016040519081016040528093929190818152602001838380828437600081840152601f19601f820116905080830192505050505050509192919290505050610179565b005b6100fe610193565b6040518080602001828103825283818151815260200191508051906020019080838360005b8381101561013e578082015181840152602081019050610123565b50505050905090810190601f16801561016b5780820380516001836020036101000a031916815260200191505b509250505060405180910390f35b806000908051906020019061018f929190610235565b5050565b606060008054600181600116156101000203166002900480601f01602080910402602001604051908101604052809291908181526020018280546001816001161561010002031660029004801561022b5780601f106102005761010080835404028352916020019161022b565b820191906000526020600020905b81548152906001019060200180831161020e57829003601f168201915b5050505050905090565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f1061027657805160ff19168380011785556102a4565b828001600101855582156102a4579182015b828111156102a3578251825591602001919060010190610288565b5b5090506102b191906102b5565b5090565b6102d791905b808211156102d35760008160009055506001016102bb565b5090565b9056fea265627a7a72315820f5df86185a4d1d6fa207ed96829d0ff774b214b849e6af605bfc345c5f769ae864736f6c634300050b0032"
  // 创建交易对象
  const txObject = {
    nonce:    web3.utils.toHex(txCount),
    gasLimit: web3.utils.toHex(1000000),
    gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
    data: data
  }
  // 签署交易
  const tx = new Tx(txObject, { chain: 'goerli', hardfork: 'petersburg' })
  tx.sign(privateKey1)
  const serializedTx = tx.serialize()
  const raw = '0x' + serializedTx.toString('hex')
  // 广播交易
  web3.eth.sendSignedTransaction(raw, (err, txHash) => {
    console.log('txHash:', txHash)
    // 可以去goerli.etherscan.io查看交易详情
  })
})
```

### 调用智能合约写函数

调用之前部署的合约的`set()`函数

构建交易对象

```
const txObject = {
  nonce:    web3.utils.toHex(txCount),
  gasLimit: web3.utils.toHex(800000),
  gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
  to: contractAddress,
  data: data
}
```

-   `to` – 此参数将是已部署智能合约的地址。可以从[etherscan](https://ropsten.etherscan.io/tx/0xc6b30a882cd6998326b7b7e30b6e64bc87887608e39602a38713f7ebd282a934)中获取。
-   `data` – 被调用的智能合约函数的十六进制表示。

通过地址和abi拿到合约。

完整：

```
var Tx     = require('ethereumjs-tx').Transaction
const Web3 = require('web3')
const web3 = new Web3('https://ropsten.infura.io/v3/YOUR_INFURA_API_KEY')
const account1 = '0xf4Ab5314ee8d7AA0eB00b366c52cEEccC62d6B4B' // Your account address 1
const account2 = '0xff96B8B43ECd6C49805747F94747bFfa3A960b69' // Your account address 2
const pk1 = 'b75e2bcaec74857cf9bb6636d66a04784d4c0fcfd908f4a2bc213428eba5af0d' // 实际项目中应该从process.env.PRIVATE_KEY_1中读取
const pk2 = 'ac0adfdbaeb0770a479e79aac78779d82fdc2f9262e0c8f751ae70fb63ef6196' // 实际项目中应该从process.env.PRIVATE_KEY_2中读取
const privateKey1 = Buffer.from(pk1, 'hex')
const privateKey2 = Buffer.from(pk2, 'hex')
// 读取已部署的契约 -- 从Etherscan获取地址
const contractAddress = '0x30951343d6d80d2c94897f1a81c53cc030aef879'
const contractABI = [
    {
        "constant": false,
        "inputs": [
            {
                "internalType": "string",
                "name": "_value",
                "type": "string"
            }
        ],
        "name": "set",
        "outputs": [],
        "payable": false,
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "constant": true,
        "inputs": [],
        "name": "get",
        "outputs": [
            {
                "internalType": "string",
                "name": "",
                "type": "string"
            }
        ],
        "payable": false,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [],
        "payable": false,
        "stateMutability": "nonpayable",
        "type": "constructor"
    }
]

const contract = new web3.eth.Contract(contractABI, contractAddress)
web3.eth.getTransactionCount(account1, (err, txCount) => {
  // 创建交易对象
  const txObject = {
    nonce:    web3.utils.toHex(txCount),
    gasLimit: web3.utils.toHex(8000000),
    gasPrice: web3.utils.toHex(web3.utils.toWei('10', 'gwei')),
    to: contractAddress,
    data: contract.methods.set("qikegu").encodeABI()
  }
  // 签署交易
  const tx = new Tx(txObject, { chain: 'goerli', hardfork: 'petersburg' })
  tx.sign(privateKey1)
  const serializedTx = tx.serialize()
  const raw = '0x' + serializedTx.toString('hex')
  // 广播交易
  web3.eth.sendSignedTransaction(raw, (err, txHash) => {
    console.log('txHash:', txHash)
    // 可以去goerli.etherscan.io查看交易详情
  })
})
```

### 智能合约事件
使用web3.js获取智能合约事件的步骤如下：

1.  创建智能合约对象。
1.  使用`getPastEvents`函数获取事件

**创建智能合约对象**  
和之前一样

**使用getPastEvent函数获取事件**

```
contract.getPastEvents(
  'AllEvents', // 过滤事件参数，这里获取全部事件
  {
    fromBlock: 0, // 起始块
    toBlock: 'latest' // 终止块
  },
  (err, events) => { console.log(events) } // 回调函数
)
```

可以限制一下要查询区块的范围，以免函数执行失败。

完整代码
```
const Web3 = require('web3')
 // YOUR_INFURA_API_KEY替换为你自己的key
const rpcURL = "https://eth-mainnet.g.alchemy.com/v2/o2XGPqgGMh7n0CjPQqYHUuQC_Xh6XnK4"
const web3 = new Web3(rpcURL)
// OMG Token Contract
const abi = require("./ERC20.json");
const address = '0xdAC17F958D2ee523a2206206994597C13D831ec7'
const contract = new web3.eth.Contract(abi, address)
// 获取事件
contract.getPastEvents(
  'AllEvents',
  {
    fromBlock: 'latest',
    toBlock: 'latest'
  },
).then(function(events) {
    console.log(events)
});
```

订阅一个事件并在第一次事件触发或错误发生后立即取消订阅。一个事件仅触发一次。

```
contract.once('Transfer', {
    filter: {myIndexedParam: [20,23], myOtherIndexedParam: '0x123456789...'}, // 使用数组表示 或：比如 20 或 23。
    fromBlock: 'latest'
}, function(error, event){ console.log(event); });

```

### 区块查询：
**查询最新区块**

```
web3.eth.getBlockNumber().then(console.log)
```
**查询最新区块**
```
web3.eth.getBlock('latest').then(console.log)
```
**查询指定哈希值的区块**
```
web3.eth.getBlock('0x25d9f9dd736ebaac3c99cb0edb5df22745c354d208c6e2b5b43f426f754adfbe').then(console.log)
```
**查询指定序号区块**
```
web3.eth.getBlock(0).then(console.log)
```
**查询某个区块中的指定交易信息**
```
const hash = '0x66b3fd79a49dafe44507763e9b6739aa0810de2c15590ac22b5e2f0a3f502073'
web3.eth.getTransactionFromBlock(hash, 2).then(console.log)
```
### 查询平均gas价格
```
// 根据最近几个区块，计算平均Gas价格
web3.eth.getGasPrice().then((result) => {
    console.log("wei: " + result)
    console.log("ether: " + web3.utils.fromWei(result, 'ether'))
})
```

### sha3、keccack256、randomHex
-   `web3.utils.sha3` – sha256哈希函数
-   `web3.utils.keccak256` – keccak256哈希函数
-   `web3.utils.randomHex` – 生成十六进制随机数

```
// sha256哈希函数
console.log(web3.utils.sha3('yh.com'))
// keccak256哈希函数
console.log(web3.utils.keccak256('yh.com'))
// 生成十六进制随机数
console.log(web3.utils.randomHex(32))
```

