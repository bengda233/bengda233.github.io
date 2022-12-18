# UniswapV2 源码解析
代码仓库：  
core:
[bengda233/uniswap-v2-core: 🎛 Core smart contracts of Uniswap V2 (github.com)](https://github.com/bengda233/uniswap-v2-core)  
Periphery:
[bengda233/uniswap-v2-periphery: 🎚 Peripheral smart contracts for interacting with Uniswap V2 (github.com)](https://github.com/bengda233/uniswap-v2-periphery)

**uniswap v2分为2个部分：**  

Core:包含Factory、Pair、Pair ERC20  
Periphery:Routers

core主要为核心逻辑，单个swap的逻辑；Periphery是外围的服务，在所有swap的基础上构建服务。


## Core
### UniswapV2 Factory
Factory主要用来创建Pair（交易池）

Factory里所有的方法合集：  
- `allPairsLength`查出目前的所有交易对个数。
- `createPair`创建一个交易对。  
> 流程：
> 1.检查输入的两种token是否一样  
> 2.对两种token进行一个简单的排序，检查小的地址是否为0地址，检查这两种token的交易对是否已经存在  
> 3.获取模板合约UniswapV2Pair的字节码
> 4.用两种token的地址生成盐值  
> 5.用create2构造出Pair交易对的地址  
> 6.将pair记录

- `setFeeTo`这个方法就是上面`第5点协议手续费`的实现，只有feeToSetter（初始为合约部署者，可转让），可以设置feeTo的地址。如果feeTo未设地址，就表示0.05%的手续费未开启

- `setFeeToSetter`该方法用来修改feeToSetter


### UniswapV2  ERC20
该合约主要是Uniswap V2代币的信息和操作方法。定义Uniswap V2代币的名称、代币的标识符、代币的精度等.

DOMAIN_SEPARATOR主要用来不同Dapp之间区分相同结构和内容的签名消息，该值也有助于用户辨识哪些为信任的Dapp

PERMIT_TYPEHASH是根据事先约定使用permit函数的部分定义计算得出的哈希值

nonces值用于记录合约中每个地址使用链下签名消息交易的数量，用来防止重放攻击

V2 ERC20里所有的方法合集：
- `_mint`用来增发代币
- `_burn`用来销毁代币
- `approve`授权的函数
- `transfer`转账函数
- `transferFrom`授权转账的函数
- `permit`用户实现验证与授权，因为用户需要借助周边合约才能和核心合约。  
当用户需要燃烧流动性代币提取代币时，需要将自己的流动性代币销毁掉，但是由于调用的是周边合约，未经授权的时候是无法进行操作的。  
如果按照正常操作，用户需要先对周边合约进行授权，再再调用周边合约进行燃烧操作。这个过程需要两个交易。  
但是如果通过线下消息签名，就可以减少一个交易。  
在周边合约里，减小流动性来提取资产时，周边合约在一个函数内先调用交易对的permit函数进行授权，接着再进行转移流动性代币到交易对合约，提取代币等操作，所有操作都在周边合约的同一个函数中进行，达成了交易的原子性和对用户的友好性。


### UniswapV2 Pair
UniswapV2Pair即交易对合约的核心逻辑。

MINIMUM_LIQUIDITY 表示最小流动值  
SELECTOR 是transfer的函数选择器的值  

blockTimestampLast 最新交易区块  
reserve0、reserve1 两种资产的数量

price0CumulativeLast、price1CumulativeLast  记录交易中两种价格的累计值  
kLast 最新时刻的k值

UniswapV2 Pair里的方法合集

- `lock()`是一个锁机制的函数修饰符，用来防止重入
- `getReserves` 获取当前两种token的数量和最近一次交易的时间
- `_safeTransfer` 内部转账的一个逻辑，调用token的转账函数并检查返回值是否都为true
- `initialize`初始化token0和token1的地址
- `_update`内部的更新reserve的方法
> 流程：  
> 1.检查当前两个token的余额是否超出限制  
> 2.更新blockTimestamp为当前区块（占32位，由于区块时间是uint类型，所以可能会超过uint32的最大值，于是这里进行取模。）  
> 3.计算与当前与上一次block的时间差，检查是否大于0，大于0就更新交易价格累计值（因为是同一个区块的第二笔及以后的交易，timeElapsed就为0，不会算到交易累加值，对应到上面的第2点价格预言）  
> 4.更新reserve和blockTimestampLast

- `_mintFee`内部函数，用来生成协议手续费。将1/6的流动性给feeto。V2的手续费会全部累计起来，流动性发生改变时才会对手续费进行分配.
> 流程：  
> 1.拿到feeTo的地址，判断是否打开  
> 2.如果打开了，就拿到当前reserve乘积出的k，和之前保存的k    
> 3.如果k增加了，就计算出返回的fee(计算过程涉及到之前的第五点协议手续费）  
> 4.如果没打开就会把kLast归零（因为feeOn是可以随时打开和关闭的，klast的更新位于mint函数与burn函数中，如果打开了更新了klast，但是没有清零又关闭了，下一次打开就是用的之前的数据）

公式：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/941ee92bd87d440a9d8a867c168720a7~tplv-k3u1fbpfcp-watermark.image?)

- `mint`用来生成流动性代币的方法（流动性增加时调用）
> 流程：  
> 1.拿到reserve0和reserve1  
> 2.算出目前的两种token的余额，将他们与reserve相减算出增加的余额。  
> 3.生成feeto的流动性代币  
> 4.如果是初次提供流动性，会生成MINIMUM_LIQUIDITY个代币给0地址（对应上面第10点初始化流动性供给），再根据恒定乘积公式中积的平方根来计算流动性：S-mint=根号(X-deposit*Y-deposit)  
> 5.如果不是初次提供流动性，根据两种token的增加比例计算并取两者间最小的。流动性=存入的token数量/token之前的总数 * 总的流动性  
> 6.更新reserve和Klast

- `burn`销毁流动性将token归还流动性提供者。  
> 流程：  
> 1.拿到reserve0和reserve1   
> 2.拿到合约实际两种token的余额，以及合约的流动性代币总数（一般情况下，合约是没有流动性代币的，因为mint是直接mint给流动性提供者，但是burn之前需要把流动性代币转给合约，于是合约就有流动性代币了）  
> 3.生成feeto的流动性代币  
> 4.计算返回的token0和token1的数量，按照比例计算：要燃烧的流动性代币/总流动性代币数量*目前token的数量  
> 5.燃烧代币  
> 6.将token0和token1转给to  
> 7.更新reserve和Klast

- `swap`用于交易对中资产的交换

> 流程：  
> 1.输入要购买的token0和token1数量，验证他们是否超过合约的余额  
> 2.对地址进行to校验  
> 3.将想要的amount数量转过去  
> 4.调用合约的uniswapV2Call回调函数将data传递过去  
> 5.再拿到两种token目前的balance  
> 6.算出注入的token0和token1的数量（比较他们是否在借出去的基础上增加以及增加的数目）  
> 7.计算是否还完(比较是否大于原来的k)。算法：目前的token1和token2数量-手续费（0.03%）将他们两相乘算出现在的k，比较是否大于原来的k。  
> 8.更新reserve

- `skim`当合约token余额超出了112位就可以调用这个方法，取出提出合约资产值和uint112最大值之间的差值

- `sync`用来将合约中缓存的资产数量同步为合约的当前值

## Periphery
Periphery周边合约是外部账户和核心合约之间的桥梁，周边合约包含了接口定义、工具类库、Router和示例实现四部分内容。

挨个解析：  
### libraries：
#### 1.UniswapV2Library.sol
- `sortTokens`该方法的作用是对上下文合约地址进行排序，检查两个地址是否相等，是否是0地址
- `pairFor`用于计算生成的交易对地址
- `getReserves`得到交易对合约token的恒定积产的值
- `quote`根据一种资产依据比例算出另一种资产
- `getAmountOut` 给定一个资产的输入金额和交易对储备，计算出能够置换到的资产。


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/656c0cc30eaf436f8e584f1d8f3932ab~tplv-k3u1fbpfcp-watermark.image?)
- `getAmountIn` 给定一个想要的资产金额和交易对储备，计算出需要给出的资产

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80cee99f760b40b586a2666f2037a657~tplv-k3u1fbpfcp-watermark.image?)

- `getAmountsOut`链式交易计算出能够置换到的资产,例如 A/B=>B/C=>C/D，用A对换B

- `getAmountsIn`链式交易给出想要置换到的资产数，计算需要付出资产。例如 A/B=>B/C=>C/D，给出想要的B计算需要的A

#### UniswapV2OracleLibrary.sol
- `currentBlockTimestamp`返回当前的区块时间
- `currentCumulativePrices`用于计算当前区块累积价格

#### UniswapV2LiquidityMathLibrary.sol
这个依赖用来做交易对里的liquidity的数学计算

- `computeProfitMaximizingTrade`计算最大化利润的交易方向
- `getReservesAfterArbitrage`在外部观察到真实价格的情况下，套利后的准备金将价格移动到利润最大化比率
- `computeLiquidityValue`计算流动性值
- `getLiquidityValue`拿到流动值
- `getLiquidityValueAfterArbitrageToPrice`计算当给定两个Token——TokenA和TokenB，和它们的"真实价格"和流动性金额，计算以TokenA和TokenB的流动性值

### UniswapV2Router02.sol
- `ensure`用于检查是否超过截止时间
- `receive`只接受WETH传来的ETH
- `_addLiquidity`用于增加流动性
> 流程：  
> 1.检查交易对是否存在，不存在就创建  
> 2.拿到tokenA和tokenB的reserve  
> 3.如果是第一次增加流动性就直接将注入的资产全部转换为流动性
> 4.如果不是第一次，根据池子的比例计算。根据传入的A的数量算出需要传入的B的数量。  
> 5.如果需要的B小于传入的B，那么流动性就是用输入的A数量和需要传入的B的数量  
> 6.如果需要的B大于传入的B，那么流动性就是用输入的B数量和需要传入的A的数量

- `addLiquidity`用户增加流动性，调用之前的 _addLiquidity函数计算出需要提供的A、B的实际数量。将token转给交易对合约，生成流动性  

- `addLiquidityETH`与addLiquidity函数类似，只不过将其中一种token换成ETH。将amountToken量的token代币转移到交易对中，之后将ETH兑换成WETH，将刚刚兑换的WETH转移至交易对合约

- `removeLiquidity`移除流动性，拿到交易对，将燃烧流动性代币转给合约， 调用交易对的burn函数燃烧掉转进去的流动性代币，提取相应的两种代币给接收者，检查是否提取的对应代币是否大于最小的提取代币数量

- `removeLiquidityETH`移除流动性，返回给流动性提供者ETH和token

- `removeLiquidityWithPermit`移除流动性，允许下签名消息来进行授权验证，从而不需要提前进行授权

- `removeLiquidityETHWithPermit`同理，返回的是ETH和token
- `removeLiquidityETHSupportingFeeOnTransferTokens`支持使用转移代币支付手续费
- `_swap`交换代币，支持链式交易
- `swapExactTokensForTokens` 用特定数量的某种token来换取其他的token
- `swapTokensForExactTokens`其他Token来换取指定数量的期望token
- `swapExactETHForTokens`，`swapTokensForExactETH``swapExactTokensForETH`,`swapETHForExactTokens`同理

- `_swapSupportingFeeOnTransferTokens`和_Swap功能类似,该函数支持在转移代币时支付手续费

### UniswapV2Router01.sol
与02类似，是不打开交易费的时候用的

### UniswapV2Migrator
- `migrate`于将UniswapV1交易对中的流动性迁移到V2交易对中

