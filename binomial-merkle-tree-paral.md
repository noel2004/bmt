# 存证系统全局状态摘要生成算法

基于需求文档对链上存证“简单证明”实现要点，提出此算法。本算法可满足实现要点及对应实现约束的有限性和完全性需求，即：

+ 对N个存证记录，其对应的全局状态摘要最大长度为 len(key) * lg(N)，其中len(key)为常量，即状态摘要函数输出的摘要长度。

+ 对特定存证产生的存证记录，其最大长度同样为len(key) * lg(N)

除此之外，此算法允许预计算存证记录的过程被并发执行

## 数据结构

Mongodb文档结构：

```
{
   index: <Integer>,
   entry: <ByteArray>,
   ProvisionL: [<ByteArray>],
   ProvisionR: [<ByteArray>],
}
```

index和entry应当被索引

## 数据结构操作

+ 证明生成 Provision：`D:<Document>, PI:[{L:<ByteArray>}|{R:<ByteArray>}] => P:[{L:<ByteArray>}|{R:<ByteArray>}]`

  要求：`<Length of PI> < <Length of D.ProvisionR>`

  实现：`P = PI concat [ {R: D.ProvisionR[len]} ] concat D.ProvisionL [:start from len + 1], with len = length of PI`

  即使用当前文档包含的证明（ProvisionR和ProvisionL）长于传入的证明的部分和当前传入的证明数据合并，
  追加部分的第一个元素取ProvisionR中的值，其它取ProvisionL的值


## 系统设计

+ 文档生成

文档数据由链上合约（chaincode）模块在执行每个索引交易计算全局状态同时产生，并以event形式输出

chaincode的全局状态为`[null | <ByteArray>]`，即旧算法中的rootState文档的state元

全局状态初始化为 `[]`

每增加一个索引 `I:<ByteArray>` 时，**在合约内**执行如下计算


```
# 令当前全局状态为S

let R = I
let ProvL = []
let ProvR = []

for i, M in range of S do

   if M == null then 
       S[i] = R
       setevent(S, I, ProvL, ProvR)
       return
   else
       ProvL = append(ProvL, M)
       ProvR = append(ProvR, R)
       R = sha256(M concat R)

enddo

S = append(S, R)
setevent(S, ProvL, ProvR)

```


易见合约中交易必须是顺序执行的，而交易所产生事件`E（S, ProvL, ProvR)`**可以被并行地读取并且写入数据库**，从事件生成文档的操作为：


```

E(S, I, ProvL, ProvR) => D:<Document>

let i = 0

# 以全局状态数组来计算整数索引的二进制表示：从低位到高位，非空元素代表1，空元素代表0

for n, M in range of S do

   if M != null then i = i + 2^n

enddo

D = {
   index: i,
   entry：I,
   ProvisionL: ProvL,
   ProvisionR: ProvR,
}

```

实际上很显然地，任何index为奇数的文档，其ProvisionL和ProvisionR项均为空；而偶数的文档，ProvisionR的第一项必然是entry
注意事件中的全局状态S仅用于计算对应的整数索引，输出时只有调试的意义。因此也可以考虑在合约中直接计算索引后输出


## 系统接口及实现


### 证明 Prov


接收一个ByteArray元素，搜索其在数据结构中的存在性，并给出证明信息，此接口可以并发执行


接口实现：

```

Def I:<ByteArray> as Input

# 首先定义补数方法 ComplementNum，即计算给定整数二进制表示时低位连续的0，并将这些0全部变成1
# 例如计算40 (101000)得到 ComplementNum(40) = 47 (101111), 而 ComplementNum(N:odd) = N
# 因为奇数末位没有0

  let ComplementNum = func (i) {

      let x = 1
      while i & x == i do
        i = i | x
        x << 1
      enddo
      return i
  }

let S = Find where entry is I

if S notfound then return

let P = S.ProvisionL.map( e => {L: e})

let I = S.index

while S = Find where index is ComplementNum (I) + 1; S existed 

    P = Provision( S, P )

enddo

return P

```

1. 注意返回不存在和空证明是有区别的。对最新存证的一条信息执行查找将可能找到空证明（即证明的root
就是其自身）

2. 上述方法中，while语句位置查找的文档索引总是大于目标索引，而 while循环中 if字句内的索引总是小于
目标索引，例如从索引为40 (101000) 的文档开始，我们首先查找48 (110000)，然后查找64，128，.... 直到
找不到对应文档为止

