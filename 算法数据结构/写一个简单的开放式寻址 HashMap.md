# 写一个简单的开放式寻址 HashMap

OK, Let's "go"

相信散列表平时大家用的是十分的多的, java里面的HashMap<K,V>, c++的unordered_map<K,V>,等一系列的散列表。

散列表底层使用类似于数组的方式来查找元素，所以每一个传递进来的key值需要得到一个唯一的整数下标。然后每一次查找删除，都会把key散列成这个下标然后再去查找。哈希表的平均查找时间复杂度为O(1),但是最坏的情况还是会有O(n), 这依赖于空间的大小和散列函数的随机性.

下面用Go来实现一个简单的散列表，使用开放寻址解决冲突，链式解决冲突的方法跟他有所不同

### ①构建基本的散列结构

```go
//HashTable is a key-value store space
//it will accelerate the speed of searching

type HashTable struct {
    keyType reflect.Type  //key的类型
    valueType reflect.Type //value的类型
    size uint64 //桶的大小
    cursize uint64 //桶已经装了多少元素
    items []*Item //桶
}
//Item is a element of HashTable
type Item struct {
    key interface{}
    value interface{}
}


```

 

 

HashTable是一个记录元素存储位置的结构，Item则是对应的元素

### ②来写一个简单的散列函数

散列函数是一个将key散列成一个整数的函数，散列函数的随机性很大程度的能够降低增删改查的碰撞率。

来散列一个字符串，函数为

```go
func hashString(s string, size uint64) uint64 {
    var hashcode uint64
    for _, v := range s {
        hashcode = hashcode<<5 + hashcode + uint64(v)
    }
    return hashcode
}

```

这个散列函数每次吧自己扩大2^5倍数并且和原来的数相加并且再加上对应位置上字符对应的码

因为散列出来的值很大又要防止溢出为负数所以使用uint64

### ③增删改查^__^

上面已经能够获得一个字符串的散列值了，但是由于散列值过于大，不可能直接当桶的下标。所以应该用取余的方式hashcode % p (p <= size)得到在桶范围内的下标, 这个p尽量要为素数.

 

插入/更新

源代码中大量用了反射，这里就单独把字符串拿出来处理

```go
func (ht *HashTable)Set(k string, v string) {

        var i uint64 //碰撞次数
        for {
            hash := hash(k, ht.size, i)
            item := ht.items[hash]
            if item == nil || item.key == k {
                newItem := &Item{k, v}
                ht.items[hash] = newItem
                ht.cursize++
                //expend hashtable if rate is bigger than increment factor
                //如果桶中元素数量比上桶的最大尺寸大于incrFactor(增长因子)，则给hash表扩容
                rate := float64(ht.cursize) * 1.0 / float64(ht.size)
                if rate > incrFactor {
                    ht.resize()
                }
                return
            }
            i++
}

```

获取

```go
//Get get the element by key
//if not exist, return <nil>
func (ht *HashTable) Get(k interface{}) interface{} {
    var i uint64
    for {
        hash := hash(k, ht.size, i)
        item := ht.items[hash]
        if item == nil {
            return nil
        } else if item.key == k {
            return item
        }
        i++
    }
}

```

下面就来谈一谈开放式寻址这种方式

现在有7个桶， 下标从0开始到6

[] [] [] [] [] [] [] []

现在有一组数据要插入进散列表，这组数据为1,11 ,8,12,18

1. 先看1, hashcode = 1 % 7 = 1, 插入到下标为1的地方, 现在桶为[][1][][][][][]
2. 再看11, hashcode = 11 % 7 = 4, 插入到下标为4的地方，现在桶为[][1][][][11][][]
3. 再看8, hashcode = 8 % 7 = 1, 本来应该插入到下标为1的地方，但是下标为1的位置已经被1占了，所以寻找(hashcode + 1) % 7 的位置2， 发现为空，则现在的桶为[][1][8][][11][][]
4. 再看12, hashcode = 12 % 7 = 5, 应该插入到下标为5的位置，[][1][8][][11][12][]
5. 最后一个18, hashcode = 18 % 7 = 5, 应该插入到下标为4的位置，但是被11占了，(hashcode+1)%7得到5，但是5的位置也被占了，所以再往右移动1为，得到6，6的位置为空，所以现在的桶为[][1][8][][11][12][18]

所以开放式寻址就是在不断的探测哪里有空的位置能够存放对应的键值对

上面的代码也是一样，先hash := hash(key), 然后在对应下标去寻找这个item，如果item已经存在或者为空，则选择覆盖和新增，否则一直hash，直到能够找到一个为空的位置。

### ④解决碰撞的方法

上面的例子就是典型的碰撞，散列表的平均查询复杂度是O(1), 对，只是平均查询复杂度，最坏的情况依旧是O(n), 这就是当碰撞概率急剧升高的时候。

举个栗子,以下是一个桶

[][1][2][3]

现在要插入一个数5, 如果按照上文中探测的方法，每次增加1，那么就需要探测4次，为整个散列表的长度了，这时候时间复杂度上升到了O(n)

解决碰撞的方法有两种，拉链法，开放地址法。本文探讨开放地址法。

开放地执法有一个公式:Hi=(H(key)+di) MOD m i=1,2,...,k(k<=m-1)， i为碰撞次数

线性探测再散列:

上文中每冲突一次下边左移1次, 即di = i,。这种方法很容易造成冲突位置聚集在一堆，导致分布不均匀，碰撞率并不能很显著的减少.

二次探测再散列:

即di=-1 1 -2 2 -4 4 -9 9 16 -16, 向两边同时探测且按一定比例扩大寻找范围

此代码采用二次探测再散列

```go
//二次探测

func collisionShift(i uint64) uint64{

distant := (i + 1) / 2

    symbol := uint64(math.Pow(-1, float64(i)))
    if i == 0 {
        return 0
    }
    if i > 0 && i < 4 {
        return distant * symbol
    }
    return (distant – 1) * (distant – 1) * symbol
}

//再散列+

func hash(k String, size, collision uint64) uint64 {

return (hashString(k) + collisionShift(collision) ) % size

}

```

###  ⑤散列表扩容

散列表是可以维持固定的大小进行添加元素的，这要采用拉链法，即在每个下标下面设置链表，相同下标的全部添加到这个链表里面。但是即便是这样，也需要扩容，因为扩容可以很当程度上的减少碰撞几率.

如何扩容？

通常会设置一个增长因子(factor), 比如在开放式寻址的方法下, 当 cursize / size > 0.7, 比率大于0.7的时候，进行扩容，比如将原来的容量扩大两倍。

但是这个扩容不是简单的扩容，因为有以下一个问题 ：

原始的下标建立在原始的容量上的，即hashcode mod size, 扩容后size发生变化，计算出来的下标就不是原来的下标了.所以不能单纯的把一个表里面的Item按照原始下标存放在新的表里面去，所有的下标都应该经过重新运算，即通过另外一张表，一个一个调用Set加入

```go
//resize the hashtable when cursize/size > increment factory
//there will expend twice
func (ht *HashTable) resize() {
    newSize := ht.size << 2
    newSize = nextPrime(newSize)
    newht := NewHashTable(ht.keyType, ht.valueType, newSize)
    for i, v := range ht.items {
        if ht.items[i] != nil {
            newht.Set(v.key, v.value)
        }
    }
    ht.cursize = newht.cursize
    ht.size = newht.size
    ht.items = newht.items
}

```

### ⑥前面谈到除留取余为什么尽量取模数为素数

这个问题我自己也十分的疑惑，大多数的说法给出的答案都是:合数因为存在额外的因数。但是由于不是很了解具体的例子，暂不分析.

 

结语:

开放式寻址方式空间利用率相对于拉链式低，但是得益于多出的空间，各个操作的效率在其他条件相等的情况下相对于拉链式要好一些.

 

参考资料：

1. > <https://github.com/jamesroutley/write-a-hash-table>

2. > <https://en.wikipedia.org/wiki/Hash_table>

3. > <https://stackoverflow.com/questions/7666509/hash-function-for-string>

    

完整代码:

https://gist.github.com/doorOfChoice/0a7edd2198f9ff0ea6a5538576d712e8