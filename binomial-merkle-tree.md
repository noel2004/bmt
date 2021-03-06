# 存证系统全局状态摘要生成算法

基于需求文档对链上存证“简单证明”实现要点，提出此算法。本算法可满足实现要点及对应实现约束的有限性和完全性需求，即：

+ 对N个存证记录，其对应的全局状态摘要最大长度为 len(key) * lg(N)，其中len(key)为常量，即状态摘要函数输出的摘要长度。

+ 对特定存证产生的存证记录，其最大长度同样为len(key) * lg(N)


## 数据结构

Mongodb文档结构：

```
{
   root: <ByteArray>,
   leafs: [<ByteArray>],
   body: [<ByteArray>],
   level: <interger>,
}
```

root和leafs需要索引，其中leafs使用数组索引（从数组内任一元素均可检索到对应文档）

## 数据结构操作

+ 生成 Init：`E:<ByteArray> => D:<Document>`

  实现：`D = {root: E, leafs: [E], level: 0, Body: []}`

  *此操作由一个ByteArray类型的参数产生一个文档结构*

+ 合并 Merge：`D1:<Document, level = a>, D2:<Document, level = a> => DD:<Document>`

  实现：

```
   DD = {
       root: sha256(root in D1 | root in D2),
       leafs: (leafs in D1) concat (leafs in D2),
       body: if a == 0 then [] else (body in D1) concat (root in D1) concat (body in D2) concat (root in D2),
       level: a+1,
       }
```

  *此操作从两个level为a的文档产生一个level为2a的文档*

+ 分裂 MergeSplit：`D1:<Document, level = a>, D2:<Document, level = a> => DD:<Document>`

  实现：

```
   DD = {
       root: sha256(root in D1 | root in D2),
       leafs: (root in D1) concat (root in D2),
       body: [],
       level: 1,
       }
```

  *此操作从两个level为a的文档产生一个level为1的新文档*

+ 证明 Provision：`D:<Document>, E:<ByteArray, in leafs of D> => [<ByteArray>]`

  实现：

```
   let i = <index of E in leafs of D>, n = <level of D>

   if n == 0 then return [E]

   #找出索引为i的元素的邻元
   let getneighbor = func (i, arr) {
       if i is odd then
         return {'L':arr[i-1] }
       else
         return {'R':arr[i+1] }
   }
   let ret = [getneighbor(i, <leafs of D>)]

   let search = <body of D>

   n = n - 1

   for ;n > 0; n = n - 1, i = i / 2 do

      let l = length of search
      (assert l is even)

      if i < 2^n then 
          search = search[ : end in l/2]
      else
          search = search[start from l/2: ]
          i = i - 2^n

      ret = ret concat [getneighbor(i / 2, search)]

      search = search[start from 2^(n-1): ]

   enddo

   return ret

```

  *此操作从文档的一个leaf元素中提出其在文档中的存在性证明，长度不超过level *

+ 选择 Select: `[D:<Document>] => D in which level of D is maxium`

  *此操作从一组文档中选择level最大的一个*


## 系统设计

数据库表中包含上述数据结构提及的一组文档元素以及一个额外的特殊文档rootState：

```
{
  _id: ObjectId(0),
  state: [<ByteArray|null>]
}
```

初始化系统在表中加入rootState文档：

```
{
  _id: ObjectId(0),
  state: []
}
```

系统定义一个全局的文档值上限levelLimit，推荐为8 (2^7 = 128)

## 系统接口及实现

### 加入元素 Add

接收一个ByteArray元素，更新整个数据库结构。

**此接口必须保证互斥执行（顺序化）**

接口实现：

```
Def I:<ByteArray> as Input


let R = Read rootState

let D = Init (I)

for i, M in range of <state in R> do

    if M == null then
        Write D
        state[i] = <root of D>
        return

    let DM = Find where root == M

    if <level of D> < levelLimit then
        D = Merge(D, DM)
        Delete DM
    else
        Write D
        D = MergeSplit(D, DM)
        
    state[i] = null
enddo

    Write D
    <state in R> = <state in R> concat [<root of D>]
```

### 搜索并证明 Find

接收一个ByteArray元素，搜索其在数据结构中的存在性，并给出证明信息

此接口可以并发执行，包括Add方法正在执行的时候也可以同时执行

接口实现：

```
Def F:<ByteArray> as Input

prerequise: Find where root == F IS empty

let Pr = []

let S = Select( Find where leaf has F )

while S is not null do

    Pr = Pr concat [Provision(S, F)]

    F = <root of S>

    S = Select( Find where leaf has F )

endo

return Pr

```

+ 如果实现返回的证明数组为空，说明查找的元素不存在

+ 通常来说，prerequise的一步是可以省略的（因为调用方在业务上不可能构造出某个根哈希对应的材料）


