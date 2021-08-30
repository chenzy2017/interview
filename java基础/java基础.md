# hashmap

- 数据结构  
  数据+链表,1.8之后新增红黑树,查询性能从o(n)提升到o(lgn)
- 允许key和value为null
- hashmap初始容量默认为16, 负载因子是0.75, 通过位运算每次扩容是2的n次方
- 1.8中优化点:新增了红黑树,从头插法改为尾插法,1.7先扩容再插入, 1.8先插入再扩容
- 线程安全:  
  1.7会造成环形链或者数据丢失;  
  1.8会发生数据覆盖的情况  
  put的时候可能会造成数据不一致  
  resize而引起死循环
- 为什么hashmap1.8使用红黑树不用二叉平衡树
    - CurrentHashMap中是加锁了的，实际上是读写锁，如果写冲突就会等待，如果插入时间过长必然等待时间更长，而红黑树相对AVL树他的插入更快
    - avl树更加平衡, 适合使用查询多的;
    - 插入数据, avl的旋转次数比红黑树更多;
- 为什么redis的zset使用跳表不用红黑树
    1. 跳表实现更简单
    2. zset有区间查询, 跳表的效率更高
- 说说你对红黑树的见解？
    - 每个节点非红即黑
        - 根节点总是黑色的
    - 如果节点是红色的，则它的子节点必须是黑色的（反之不一定）
        - 每个叶子节点都是黑色的空节点（NIL节点）
        - 从根节点到叶节点或空子节点的每条路径，必须包含相同数目的黑色节点（即相同的黑色高度）


# concurrenthashmap
1. currentHashMap的key和value都不允许为null(因为不知道是key为null还是value是null)
1. 大量的利用了volatile,CAS等乐观锁技术来减少锁竞争对于性能的影响
2. ConcurrentHashMap保证线程安全的方案是：  
   JDK1.8：synchronized+CAS+HashEntry+红黑树  
   JDK1.7：ReentrantLock+Segment+HashEntry
3. Segment继承ReentrantLock, 是一种可重入锁，是一种数组和链表的结构，一个Segment中包含一个HashEntry数组(用volatile修饰, 保证了多线程下的可见性), 每个HashEntry又是一个链表结构，因此在ConcurrentHashMap查询一个元素的过程需要进行两次Hash操作，如下所示：
    - 第一次Hash定位到Segment
    - 第二次Hash定位到元素所在的链表的头部
4. 但在JDK1.8中摒弃了Segment分段锁的数据结构，基于CAS(乐观锁)操作保证数据的获取以及使用synchronized关键字对Node（首结点）（实现 Map.Entry) 加锁来实现线程安全，这进一步提高了并发性
5. 为什么使用synchronized替换可重入锁ReentrantLock?
    - jdk1.6中, 对 synchronized 锁的实现引入了大量的优化，并且 synchronized 有多种锁状态，会从无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁一步步转换, 所以性能差不多
    - 减少内存开销 。假设使用可重入锁来获得同步支持，那么每个节点都需要通过继承 AQS来获得同步支持。但并不是每个节点都需要获得同步支持的,只有链表的头节点（红黑树的根节点）需要同步，这无疑带来了巨大内存浪费
5. 描述ConcurrentHashMap的put操作步骤
    1. 如果没有初始化，就调用 initTable() 方法来进行初始化；
    2. 如果没有 hash 冲突就直接 CAS 无锁插入；
    3. 如果需要扩容，就先进行扩容；
    4. 如果存在 hash 冲突，就加锁来保证线程安全，两种情况：一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入；
    5. 如果该链表的数量大于阀值8，就要先转换成红黑树的结构，break 再一次进入循环
    6. 如果添加成功就调用 addCount() 方法统计 size，并且检查是否需要扩容。
6. synchronized和ReentrantLock区别?
    1. ReenTrantLock可以指定是公平锁还是非公平锁(默认)。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。
    2. ReenTrantLock提供了一个Condition（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像synchronized要么随机唤醒一个线程要么唤醒全部线程。
    3. ReenTrantLock提供了一种能够中断等待锁的线程的机制，通过lock.lockInterruptibly()来实现这个机制。









