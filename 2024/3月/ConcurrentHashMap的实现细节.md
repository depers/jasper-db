| title                       | tags                   | background                                                   | auther | isSlow |
| --------------------------- | ---------------------- | ------------------------------------------------------------ | ------ | ------ |
| ConcurrentHashMap的实现细节 | Java/集合框架/线程安全 | 之前在开发中只知道为了保证HashMap的线程安全，直接使用ConcurrentHashMap就行了，对这个类的认识和了解不够深入，所以这里单独开一篇文章来探讨这个类的实现细节和使用场景。 | depers | true   |

# 结论

* ConcurrentHashMap是线程安全的。
* ConcurrentHashMap在Java7以及之前的版本，数据存储是通过**数组+链表**的方式存储的，采用了**分段锁技术**保证操作的线程安全。
* ConcurrentHashMap在Java8版本进行了升级优化，数据存储采用了**数组+链表+红黑树**的结构，采用了**CAS+synchronized**机制保证数据的一致性的。