---
logout: post
title: HashTable、HashMap、ConcurrentHashMap异同
tags: [java,all]
---

### HashMap、HashTable、ConcurrentHashMap三者区别

|              | HashMap               | HashTable             | ConcurrentHashMap                          |
| ------------ | --------------------- | --------------------- | ------------------------------------------ |
| 是否线程安全 | 不安全                | 安全                  | 安全                                       |
| 结构         | 数组+链表+红黑树(1.8) | 数组+链表             | 数组+链表+红黑树(1.8)                      |
| 同步机制     |                       | 锁住整个对象          | CAS+同步锁(1.8)，segment(1.7)+synchronized |
| 允许为null   | key和value可以为null  | key\value都不能为null | key\value都不能为null                      |

### HashTable

底层是数组+链表实现，无论key还是value都不能为null，线程安全，实现线程安全的方式是在修改数据时锁住整个HashTable，效率低，ConcurrentHashMap做了相关优化。初始的size为11，扩容：`newsize=oldsize*2+1`。计算index的方法：`index=(hash&0x7FFFFF)%tab.length`

### HashMap

HashMap底层是`数组+链表`组成的，不过jdk1.7和jdk1.8实现略有不同。HashMap可以存储null键和null值，但key只能插入一次null，value可以插入多个。当Map中元素总数超过Entry数组的75%，会触发扩容操作，为了减少链表长度，会时元素分配更均匀。

#### jdk1.7和jdk1.8的区别

|                        | jdk1.7                             | jdk1.8                          |
| ---------------------- | ---------------------------------- | ------------------------------- |
| 底层结构               | 数组+链表                          | 数组+链表/红黑树(链表个数大于8) |
| 插值方式               | 先扩容再插值                       | 先插值再扩容                    |
| 插入方式(put方法)      | 表头插入法                         | 表尾插入法                      |
| 扩容后是否重新计算索引 | 扩容时需要重新计算哈希值和索引位置 | 不需要重新计算                  |
|                        | 改变原有顺序，并发时引起链表闭环   | 保持原有顺序，不会出现闭环      |

#### HashMap -- jdk1.7

初始容量是16，数组中每一个元素其实就是`Entry<K,V>[] table`，Map中的key和value都是以Entry的形式存储的。Entry包含四个属性：key、value、hash值和用于单向链表的next。

在HashMap中，null可以作为键，这样的键只有一个，但可以有一个或多个键所对应的值为null。**当get()方法返回null值时，即可以表示HashMap中没有该key，也可以表示该key所对应的value为null**。HashMap中不能由get()方法来判断HashMap中是否存在某个key，应该用**containsKey()**方法来判断。

每次插入值时，会先计算出hash值，然后查找该链，如果没有查到，就把新值插入到该数组所在位置的链表表头。但由于该方式是非线程安全的，在高并发模式下，在put操作的时候，如果`size>initialCapacity*loadFactor`时，hash表会进行扩容，那么这时候就会进行rehash操作，随之HashMap结构就会发生很大的变化。就有可能导致在执行HashMap.get的时候进入死循环，将CPU消耗到100%。

为什么会形成闭环呢？可以参考[HashMap闭环(死循环)的详细原因](https://www.cnblogs.com/jing99/p/11319175.html)

- 正常单线程扩容

执行代码细节：

```java
do { 
    Entry<K,V> next = e.next; // <--假设线程一执行到这里就被调度挂起了 
    int i = indexFor(e.hash, newCapacity); 
    e.next = newTable[i]; 
    newTable[i] = e; 
    e = next; 
} while (e != null);
```

最开始Hash表的大小是2，但是需要插入key=5,7,3，一次你需要扩容，执行`resize`后大小是4，然后进行重新rehash操作。

![img](http://s9.51cto.com/wyfs01/M00/0E/D4/wKioJlGwIqaBMCL9AACSa9G7jXM118.jpg)

- 高并发模式下扩容

假设有两个线程，假设线程二已经扩容完成，扩容的步骤和上述单线程一致，结果如下，从结果看出，在重新hash之后，由于jdk1.7版本是表头插入的方式，且因为Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。我们可以看到链表的顺序被反转。

![img](http://s2.51cto.com/wyfs01/M01/0E/D6/wKioOVGwIqbT2Lm9AABa7hopONo955.jpg)

接下来线程一开始调度执行重分区操作，先是执行`newTalbe[i] = e`，然后执行`e = next`,导致e指向key(7)。而下一次循环的`next=e.next`导致了next指向了key(3)。执行如下

![img](http://s2.51cto.com/wyfs01/M01/0E/D6/wKioOVGwIqaSLr4_AABRK-2isvQ893.jpg)

线程一接着工作。把key(7)摘下来，放到newTable[i]的第一个，然后把e和next往下移。

![img](http://s9.51cto.com/wyfs01/M01/0E/D4/wKioJlGwIqaBFzwAAABjy2yXnj0185.jpg)

最后，`e.next = newTable[i] `导致key(3).next指向了key(7)。此时出现了闭环

![img](http://s5.51cto.com/wyfs01/M01/0E/D4/wKioJlGwIqajFEx0AABbKoX02SU647.jpg)

#### HashMap -- jdk1.8

> 1.7有个明显的缺陷：当当 Hash 冲突严重时，在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为 `O(N)`，

在jdk1.8中HashMap的内部结构可以看作是数组`(Node<K,V>[] table)`和链表的复合结构，数组被分为一个个桶（bucket），通过哈希值决定了键值对在这个数组中的寻址（哈希值相同的键值对，则以链表形式存储。有一点需要注意，如果链表大小超过阈值`(TREEIFY_THRESHOLD,8)`，会被调整成一颗红黑树，此时红黑树的查找效率一直是`O(logn)`，可以参见红黑树相关内容。

![img](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/QCu849YTaIPf1sDCN5zcDdGsibZwyzy9rc81tfAsDb0FdjzHkBRu4jXRgLco0aDPXQXOqTiamFL9eOtC0g5RuwYw/640?wx_fmt=png)

### ConcurrentHashMap

HashTable和HashMap都实现了Map接口，但是HashTable的实现基于Dictionary抽象类的，Java5提供了ConcurrentHashMap，它是HashTable的替代，比HashTable的扩展性更好。ConcurrentHashMap是线程安全且高效的HashMap实现。

#### jdk1.7和jdk1.8的区别

jdk1.8时主要的改进：

- 不采用**segment而采用node，锁住node来实现减小锁粒度**。
- 设计了MOVED状态 当resize的中过程中 线程2还在put数据，线程2会帮助resize。
- 使用3个**CAS操作来确保node的一些操作的原子性**，这种方式代替了锁。
- sizeCtl的不同值来代表不同含义，起到了控制的作用。
- **采用synchronized而不是ReentrantLock**

#### ConcurrentHashMap -- jdk1.7

![640?wx_fmt=png](https://ss.csdn.net/p?https://mmbiz.qpic.cn/mmbiz_png/QCu849YTaIPf1sDCN5zcDdGsibZwyzy9rmnTSzibQ6VEBXUhicBWHFae47ShkNzCRB7SZibuUN6gDmGkfeB5saAMQQ/640?wx_fmt=png)

1.7版本是由Segment数组、HashEntry组成，和HashMap一样，仍然是数组加链表的结构，但区别是在数组的上层加了Segment段，用来代表多个数组。在ConcurrentHashMap中核心数据如value以及链表都是volatile修饰的，保证了获取时的可见性。

原理上来说，ConcurrentHashMap 采用了分段锁技术，其中 Segment 继承于 ReentrantLock。不会像 HashTable 那样不管是 put 还是 get 操作都需要做同步处理，理论上 ConcurrentHashMap 支持 CurrencyLevel (Segment 数组数量)的线程并发。每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。**(相当于对每个segment加锁)**

#### ConcurrentHashMap -- jdk1.8

1.7 已经解决了并发问题，并且能支持 N 个 Segment 这么多次数的并发，但依然存在 HashMap 在 1.7 版本中的问题(**链表查询效率太低**)。因此存储结构1.8时也采用了数组+链表/红黑树的结构。

1.8也抛弃了原有的Segment分段锁，采用了`CAS + synchronized`来保证并发安全性。

##### Node类

Node是最核心的内部类，它包装了key-value键值对，所有插入ConcurrentHashMap的数据都包装在这里面。它与HashMap中的定义很相似，但是但是有一些差别它对value和next属性设置了volatile同步锁(与JDK7的Segment相同)，它不允许调用setValue方法直接改变Node的value域，它增加了find方法辅助map.get()方法。

##### TreeNode类

树节点类，另外一个核心的数据结构。当链表长度过长的时候，会转换为TreeNode。但是与HashMap不相同的是，它并不是直接转换为红黑树，而是把这些结点包装成TreeNode放在TreeBin对象中，由TreeBin完成对红黑树的包装。而且TreeNode在ConcurrentHashMap集成自Node类，而并非HashMap中的集成自LinkedHashMap.Entry<K,V>类，也就是说TreeNode带有next指针，这样做的目的是方便基于TreeBin的访问。

##### TreeBin类

这个类并不负责包装用户的key、value信息，而是包装的很多TreeNode节点。它代替了TreeNode的根节点，也就是说在实际的ConcurrentHashMap“数组”中，存放的是TreeBin对象，而不是TreeNode对象，这是与HashMap的区别。另外这个类还带有了读写锁。

##### CAS关键操作

> tabAt()该方法用来**获取table数组中索引为i的Node元素**。
>
> casTabAt()利用**CAS操作设置table数组中索引为i的元素**
>
> setTabAt()该方法用来设置table数组中索引为i的元素

##### 扩容

当ConcurrentHashMap容量不足的时候，需要对table进行扩容。这个方法的基本思想跟HashMap是很像的，但是由于它是支持并发扩容的，所以要复杂的多。原因是它支持多线程进行扩容操作，而并没有加锁。我想这样做的目的不仅仅是为了满足concurrent的要求，而是希望利用并发处理去减少扩容带来的时间影响。因为在扩容的时候，总是会涉及到从一个“数组”到另一个“数组”拷贝的操作，如果这个操作能够并发进行，那真真是极好的了。

整个扩容操作分为两个部分：

- 构建一个nextTable,它的容量是原来的两倍，这个操作是单线程完成的。这个单线程的保证是通过RESIZE_STAMP_SHIFT这个常量经过一次运算来保证的
- 将原来table中的元素复制到nextTable中，这里允许多线程进行操作。

它的大体思想就是遍历、复制的过程。首先根据运算得到需要遍历的次数i，然后利用tabAt方法获得i位置的元素：

> 如果这个位置为空，就在原table中的i位置放入forwardNode节点，这个也是触发并发扩容的关键点；
>
> 如果这个位置是Node节点（fh>=0），如果它是一个链表的头节点，就构造一个反序链表，把他们分别放在nextTable的i和i+n的位置上
>
> 如果这个位置是TreeBin节点（fh<0），也做一个反序处理，并且判断是否需要untreefi，把处理的结果分别放在nextTable的i和i+n的位置上
>
> 遍历过所有的节点以后就完成了复制工作，这时让nextTable作为新的table，并且更新sizeCtl为新容量的0.75倍 ，完成扩容。

最精彩的地方是多线程下保证安全的操作：**在代码的69行有一个判断，如果遍历到的节点是forward节点，就向后继续遍历，再加上给节点上锁的机制，就完成了多线程的控制。多线程遍历节点，处理了一个节点，就把对应点的值set为forward，另一个线程看到forward，就向后遍历。这样交叉就完成了复制工作。而且还很好的解决了线程安全的问题。**

![img](http://static.oschina.net/uploads/space/2016/0516/173132_DQMG_2243330.jpg) 

##### put过程

1. 判断Node[]数组是否初始化，没有则**进行初始化操作**
2. 通过**hash定位数组的索引坐标**，是否有Node节点，如果没有则使用CAS进行添加（链表的头节点），添加失败则进入下次循环。
3. 检查到内部正在扩容，就帮助它一块扩容。
4. 如果f！=null，则**使用synchronized锁住**f元素（链表/红黑树的头元素）。如果是Node（链表结构）则执行链表的添加操作；如果是TreeNode（树型结构）则执行树添加操作。
5. 判断链表长度已经达到临界值8（默认值），当节点超过这个值就需要**把链表转换为树结构**。
6. 如果添加成功就调用addCount（）方法统计size，并且检查是否需要扩容

##### get过程

1. 计算hash值，定位到该table索引位置，如果是首节点符合就返回。
2. 如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回。
3. 以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null

### 参考

[https://blog.csdn.net/weixin_44460333/article/details/86770169](https://blog.csdn.net/weixin_44460333/article/details/86770169)

[https://yuanrengu.com/2020/ba184259.html](https://yuanrengu.com/2020/ba184259.html)

[https://www.cnblogs.com/jing99/p/11319175.html](https://www.cnblogs.com/jing99/p/11319175.html)

[https://blog.csdn.net/a123demi/article/details/79100534](https://blog.csdn.net/a123demi/article/details/79100534)