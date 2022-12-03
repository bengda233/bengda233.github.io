# Uniswap V1源码解析

代码仓库：[bengda233/uniswap-v1-contracts: 🐍Uniswap V1 smart contracts (github.com)](https://github.com/bengda233/uniswap-v1-contracts)


Uniswap V1是用Vyper语言进行开发的，Vyper是一种语法和Python非常接近的语言。


**V1的体系分为两个部分：**  

Exchange: 用于进行ETH和ERC-20代币之间的兑换。   
Factory:用于创建和记录所有的Exchange，也用于查询代币对应的Exchange。


## Exchange

exchange里所有方法的合集：

- `setup(token_addr: address)`     构造函数，但是目前部署不支持使用，因为exchange合约在factory里构造

- `addLiquidity(min_liquidity: uint256, max_tokens: uint256, deadline: timestamp) -> uint256`    向资金池中添加流动性。  

> 流程：  
> 1.用户调用该方法并发送一定量的ETH  
> 
> 2.Uniswap通过比较发送的ETH量和池子中的ETH数量，计算用户需要发送的代币数量，并将该数量从用户转给自己（因为Uniswap Exchange要求用户按照当前资金池中的ETH和代币的比例添加流动性）  
> 需要存入的token数量= 存入的ETH数量 * 当前池子总token数量 / 当前池子中ETH数量（未更新） + 1 
> 
> 3. 向用户发放LP（Liquidity Pool）代币，其数量为AmountLPToken（证明用户确实提供了流动性及用户流动性所占的份额）  
用户拿到的LP数量 = 存入的ETH数量 * 总LP数量 / 当前池子中ETH数量（未更新）

由于Uniswap去中心化的特性，添加流动性的交易发出时和确认时流动性池的兑换比例（或者说价格）可能不同。为了避免这个问题给用户造成的损失，`addLiquidity`函数提供了三个参数进行控制：

 **min_liquidity**：用户期望的LP代币数量。如果最终产生的LP代币数量过少，则交易会回滚避免损失。  

**max_tokens**：用户想要提供的最大代币量。如果计算得出的代币数量大于这个参数，代表用户不愿意提供更多代币，交易也会回滚。  

**deadline**：时限。如果交易确认的区块时间大于deadline，也会回滚。
 
- `removeLiquidity(amount: uint256, min_eth: uint256(wei), min_tokens: uint256, deadline: timestamp) -> (uint256(wei), uint256):`  取回流动性  
> 流程：  
> 1.用户输入想要取出的LP数量  
> 
> 2.uniswap计算返回的ETH数量和token数量
> 返回ETH数量=  想要取出的LP数量* 当前池里ETH数量 / 总LP数量  
> 返回token数量 = 想要取出的LP数量* 当前池里token数量 / 总LP数量  
>
>3.检查是否大于用户的期望值，是就返给用户，否就回滚

- `getInputPrice(input_amount: uint256, input_reserve: uint256, output_reserve: uint256)` 精确指定换入（input）值，确定池子中输入单位和输出单位存量时，精确的输入数量能换出输出数量。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c8abea27f814065b2c52caa504f3e56~tplv-k3u1fbpfcp-watermark.image?)

输入的0.3%作为交易费用，剩下的进入池子。以此来算出能够兑换的Output.


- `getOutputPrice(output_amount: uint256, input_reserve: uint256, output_reserve: uint256) -> uint256:`确定输入种类、输出种类以及输出金额。计算方式为：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/83efad4ddeea49919aa07d454ab8eedb~tplv-k3u1fbpfcp-watermark.image?)



uniswap提供的四个**价格查询**的函数：

-  `getEthToTokenInputPrice(eth_sold: uint256(wei)) -> uint256:` 输入要卖出的ETH数量，返回能得到的token数量
- `getEthToTokenOutputPrice(tokens_bought: uint256) -> uint256(wei):`输入要买的token数量，返回需要给出的ETH数量
- `getTokenToEthInputPrice(tokens_sold: uint256) -> uint256(wei):`输入要卖的token数量，返回得到的token数量
- `getTokenToEthOutputPrice(eth_bought: uint256(wei)) -> uint256`输入得到的ETH数量，返回需要的token数量

**Eth与代币间的互换：**  
来看内部实现的四个函数

`ethToTokenInput`:通过精确的ETH输入量eth_sold 计算价格并交换代币。通过getInputPrice计算输出的代币数量。同样包含了min_tokens最小代币输出量和deadline的时间限制。  
`ethToTokenOutput`:通过精确的代币输出量tokens_bought 计算价格并交换代币。通过getOutputPrice计算输入的ETH数量，并可能在ETH需求量小于用户发送量时产生Refund。同样包含了min_tokens最小代币输出量和deadline的时间限制。
`tokenToEthInput`:与上同理  
`tokenToEthOutput`:与上同理  

**用ETH以兑换代币的四个方法**  
- ethToTokenSwapInput
- ethToTokenTransferInput
- ethToTokenOutput
- ethToTokenSwapOutput

swap与transfer的区别是：swap的接收者为调用者自己，而transfer的调用者可以任意指定。

**用token对换ETH的四个方法**
- tokenToEthSwapInput
- tokenToEthTransferInput
- tokenToEthSwapOutput
- tokenToEthTransferOutput


**代币之间互换**  
- `tokenToTokenInput` 输入卖出的token数量，要买入另外的token的exchange合约地址和最小预期值 得到能买入的数量。  
先算出能够买入的ETH数量，再用这个ETH去另外的token的exchange合约买token。


- `tokenToTokenOutput`输入想要买入的token数量，要买入另外的token的exchange合约地址和最小预期值 得到需要卖出的数量。
先算出需要的ETH数量，再根据ETH数量返回需要的token数量。

**用token对换token的八个方法**
- tokenToTokenSwapInput
- tokenToTokenTransferInput
- tokenToTokenSwapOutput
- tokenToTokenTransferOutput
- tokenToExchangeSwapInput
- tokenToExchangeTransferInput
- tokenToExchangeSwapOutput
- tokenToExchangeTransferOutput

上面四个和下面四个的区别在于上面的是输入需要对换token的地址，下面是输入对换token的exchange合约地址

## Factory
Factory里所有方法的合集：

`initializeFactory(template: address):`初始化Factory，只有创建时被调用。template是链上部署的合约，用于作为后续创建的Exchange的模板。

`createExchange(token: address) -> address:`根据template模板和token地址创建一个新的Exchange,并返回合约地址。随后调用新创建的Exchange的setup函数设置代币地址，并将新创建的Exchange记录在合约内。

**注意**：在做验证的过程中，函数约束每一个代币只能对应一个Exchange，这是为了约束某个代币的所有流动性都划分在一个池子中，增加池子中对应的存储量，降低交易的滑点。