> Java/集合

> 这两天在学习Java集合框架的时候，发现HashMap是一个线程不安全的类，这篇文章主要讨论了他线程不安全的原因。

# 什么是线程不安全的类

如果一个类的对象同时被多个线程访问，如果不做特殊的同步或并发处理，很容易表现出线程不安全的现象，比如抛出异常、逻辑处理错误等，这种类我们就称为线程不安全的类。

# jdk7中HashMap的线程不安全

先来揭秘原因：JDK7 中，由于多线程对HashMap进行扩容，调用了HashMap#transfer()，具体原因：某个线程执行过程中，被挂起，其他线程已经完成数据迁移，等CPU资源释放后被挂起的线程重新执行之前的逻辑，数据已经被改变，造成死循环、数据丢失

具体分析

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