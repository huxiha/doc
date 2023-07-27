v1: ETH<>token token<>token 需要把一个 token 转换成 eth 然后交换  
v2: token<>token 直接交换  
v3: 允许流动性提供者从池子中移走大部分他们提供的资产  
v4： 允许更多池子的定制化

**v1 的简单原理**
xy=k  
x，y: 池子中某种 token 的数量
k： 一个常量

**交换 token 的算法**  
比如要用 a 的 tokenA 换取 tokenB  
(x+a)(y-b) = k = xy  
xy - bx + ay - ab = xy  
ay = b(x+a)  
b = ay/(x+a)  
所以可以取出 b 的 tokenB

**存 token 算法**  
都是转换成 eth 来成比例计算，然后给 lp_token

第一次存入一种类型的 token,就按想存多少存多少，然后给相应的 lp_token  
第二次及之后就要按比例存入
这个比例按照 eth 对应 token 的比例存  
比如 eth x 数量，现在已经有 token y  
y/x = k  
存入的时候要带 eth msg.value  
(y+b)/(x+msg.value) = k  
y + b = y + y/x\*msg.value  
b = y \* msg.value/x  
你可以存入的 token 数量: b=msg.value \* y/x  
lp_token = lp 总量 \* msg.value/x

**取自己存的 token（拿 lptoken 去取）**  
LP_token(取的 eth 和 token 都和 lp_token 关联)

现在有 x ether 和 y Lp_token  
y-b/x-a = y/x  
y - b = (x-a) \* y / x  
y - b = y - ay/x  
b = ay/x  
a = bx/y 可取走这么多 ether

同理 x 替换成总 TOKEN  
可取走的 token = b\*x/y

根据每个人持有的 LP_token 数量分发利息  
收取交换手续费 0.03  
所以交换实际取走的 token = b \* 0.997
