> Java/集合框架

> 这两天在学习Java集合框架的时候，发现HashMap是一个线程不安全的类，这篇文章主要讨论了他线程不安全的原因。

# 什么是线程不安全的类

如果一个类的对象同时被多个线程访问，如果不做特殊的同步或并发处理，很容易表现出线程不安全的现象，比如抛出异常、逻辑处理错误等，这种类我们就称为线程不安全的类。

# jdk7中HashMap的线程不安全

先来揭秘原因：JDK7 中，由于多线程对HashMap进行扩容，调用了`HashMap#transfer()`，某个线程执行过程中，被挂起，其他线程已经完成数据迁移，等CPU资源释放后被挂起的线程重新执行之前的逻辑，数据已经被改变，造成**死循环、数据丢失**。

## 具体分析

JDK7中HashMap扩容的代码如下：

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity); // 获取到元素新的桶下标
            e.next = newTable[i]; // 采用头插法，将元素的next指向链表的第一个元素（也就是桶里面的元素）
            newTable[i] = e; // 将桶中的元素置为当前的元素
            e = next; // 然后开始处理当前元素的下一个元素
        }
    }
}
```

首先我们来模拟下扩容的流程：

1. 假设有两个线程A、B，同时对HashMap进行扩容操作。当前HashMap中key值存储状态如下，蓝色的是桶（存储结构是数组）：

   ![hashmap-1](/assert/hashmap-1.png)

2. 正常情况下，上面这个结构扩容之后的结构如下图：

   ![hashmap-2](/assert/hashmap-2.png)

3. 此时线程A开始执行`transfer()`方法：e是key=3的元素，我们来执行这段代码：

   ```java
   Entry<K,V> next = e.next; // next是key=7的元素
   if (rehash) {
       e.hash = null == e.key ? 0 : hash(e.key);
   }
   int i = indexFor(e.hash, newCapacity); // 重新hash后e的桶的下标为3，也就是i=3
   e.next = newTable[i]; // 此时newTable[3]=null, 所以e.next=null
   newTable[i] = e; // 此时由于CPU时间片耗尽，线程A被挂起，这行代码还没执行
   e = next;
   ```

   此时内存中上述数据的结构如下，next是key=7的元素，next的next是key=5的元素，e是key=3的元素，e的next是null：

4. 由于方法局部变量是堆栈封闭的，所以线程A的执行情况并不会影响线程B的执行。如果线程B顺利的完成了扩容操作，线程B中该hashmap的结构如下图：

   ![hashmap-2](/assert/hashmap-2.png)
   
5. 按照Java内存模型可知，线程B顺利完成扩容之后，Java会将工作内存的数据同步到主内存，此时在主内存中table（原来的桶数组）和newTable（扩容后的桶数组）都是第4步中的那个结构。也就是说key=7的元素next是key=3的元素，key=3的元素next是null。
   
6. 此时线程A重新获得CPU时间片，继续执行`newTable[i] = e;`，也就是`newTable[3] = e`，代码执行的步骤如下：

   ```java
   newTable[i] = e; // 此时线程A获得时间片，继续执行，newTable[3]=key为3的元素
   e = next; // 此时e=key为7的元素
   ```

   这个循环执行之后的hashmap的结构如下图：

   ![hashmap-3](/assert/hashmap-3.png)

7. 此时接着执行下一个循环，e是key=7的元素，代码执行的步骤如下：

   ```java
   Entry<K,V> next = e.next; // e是key=7的元素，从主内存中读取e.next时发现主内存中7.next=3，所以next是key=3的元素
   if (rehash) {
    e.hash = null == e.key ? 0 : hash(e.key);
   }
   int i = indexFor(e.hash, newCapacity); // 重新hash后e的桶的下标为3，也就是i=3
   e.next = newTable[i]; // 采用头插法，此时newTable[3]=3, 所以e.next=key为3的元素
   newTable[i] = e; // 此时newTable[3]=key为7的元素
   e = next; // 此时e为key=3的元素
   ```
   
   执行完上述的代码，此时HashMap的结构如下图：
   
   ![hashmap-4](/assert/hashmap-4.png)
   
8. 接下来，我们继续执行循环中的代码，此时e是key=3的元素：
   
   ```java
   Entry<K,V> next = e.next; // e.next是null，所以这是最后一次循环
   if (rehash) {
       e.hash = null == e.key ? 0 : hash(e.key);
   }
   int i = indexFor(e.hash, newCapacity); // 获取到元素新的桶下标
   e.next = newTable[i]; // 采用头插法，此时元素3的下一个元素指向了元素7
   newTable[i] = e; // 将桶中的元素置为了元素3
   e = next; // 此时e是null，循环结束
   ```
   
   执行完上述代码，此时HashMap的结构如下图：
   
   ![hashmap-5](/assert/hashmap-5.png)
   
   因为`while(null != e)`，测试e=null，所以循环结束，线程A完成了扩容操作，会将工作内存的结构同步到主内存中，此时主内存中的newTable和table都会变成上面的结构。
   
   **当线程A执行完后，HashMap中出现了环形结构，当在以后对该HashMap进行操作时会出现死循环。**
   
   **当线程A执行完后，元素5在扩容期间被丢失，这就发生了数据丢失的问题。**
   
   所以，HashMap是线程不安全的。

# jdk8中HashMap的线程不安全

JDK1.8 HashMap线程不安全体现在：**数据覆盖**。

## 具体分析

下面这段代码是HashMap的put方法调用的逻辑：

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null) # 如果桶数组当前位置没有发送hash碰撞或者说这里没有元素
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold) # 判断hashmap元素的数量是否超过阈值
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

1. **数据覆盖情况一**

    首先我们来看这一句代码：`if ((p = tab[i = (n - 1) & hash]) == null)`

    * p：桶数组的元素，也就是指桶元素指向链表的第一个链表元素或是链表的头元素。
    * tab：桶数组。
    * n：桶的大小，也就是常说的HashMap的容量。
    * hash：根据key计算出的hash值。
    * `(n - 1) & hash`：这句代码计算出的值永远小于等于**n-1**。

    所以这段代码就是在判断这个元素是否能够放到桶数组中，或者说是否能做链表的表头元素，也就是上面说的这里是否发送了hash碰撞，如果发生了hash碰撞，则这个元素就要使用拉链法存储到链表中去。

    这句代码如果有线程A、B两个线程同时执行到这里。

    1. 线程A执行完这句代码之后发现没有发生hash碰撞，准备要执行下一行，将元素放到桶数组下标为2的位置，但是由于CPU时间片耗尽，被挂起。
    2. 线程B也执行到这句代码，发送也没有发生hash碰撞且计算的桶数组下标也是下标为2的位置，执行下一行后，将元素放到了下标为2的位置。
    3. 此时线程A获得时间片，继续执行，将元素放到了下标为2的位置。因为线程A之前已经判断过是否发生Hash碰撞，所以这次是直接插入的。这就导致原来线程B放置到该下标2的数据被线程A给覆盖了，从而导致了线程不安全。


2. **数据覆盖情况二**

   接着我们来看下一句代码：`if (++size > threshold)`

   * size：键值对数量。
   * threshold：size 的临界值，当 size 大于等于 threshold 就必须进行扩容操作。

   这句代码如果有线程A、B两个线程同时执行到这里。

   1. 线程A、B，这两个线程同时进行put操作时，假设当前HashMap的zise大小为10。

   2. 当线程A执行到这行代码时，从主内存中获得size的值为10后准备进行+1操作，但是由于时间片耗尽只好让出CPU。

   3. 线程B快乐的拿到CPU还是从主内存中拿到size的值10进行+1操作，完成了put操作并将size=11写回主内存。

   4. 然后线程A再次拿到CPU并继续执行(此时size的值仍为10)，当执行完put操作后，还是将size=11写回内存。

   5. 此时，线程A、B都执行了一次put操作，但是size的值只增加了1，所以说还是由于数据覆盖又导致了线程不安全。

# 参考文章

* [HashMap为什么线程不安全](https://juejin.cn/post/6917526751199526920)