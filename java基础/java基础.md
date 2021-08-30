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



# 线程池
1. 线程池有哪些核心参数
    1. 核心线程数：corePoolSize ,线程池中活跃的线程数  
       allowCoreThreadTimeOut的值是控制核心线程数是否在没有任务时是否停止活跃的线程，
       当它的值为true时，在线程池没有任务时，所有的工作线程都会停止
    2. 最大线程数：maximumPoolSize
    3. 多余线程存活时长：keepAliveTime, 多余线程数 = 最大线程数 - 核心线程数
       如果在这个时间范围内，多余线程没有任务需要执行，则多余线程就会停止
    4. 多余线程存活时间的单位：TimeUnit
    5. 任务队列：workQueue
       线程池的任务队列，使用线程池执行任务时，任务会先提交到这个队列中，然后工作线程取出任务进行执行，当这个队列满了，线程池就会执行拒绝策略。
    6. 线程工厂：threadFactory
       创建线程池的工厂，线程池将使用这个工厂来创建线程池，自定义线程工厂需要实现ThreadFactory接口。
    7. 拒绝执行处理器（也称拒绝策略）：handler
       当线程池无空闲线程，并且任务队列已满，此时将线程池将使用这个处理器来处理新提交的任务。
2. 线程池有哪些拒绝策略(0到9)
    - AbortPolicy         --(0,1) 当任务添加到线程池中被拒绝时，它将抛出 RejectedExecutionException 异常。
    - CallerRunsPolicy    --(全部执行完) 当任务添加到线程池中被拒绝时，会在线程池当前正在运行的Thread线程池中处理被拒绝的任务。
    - DiscardOldestPolicy --(0,9) 当任务添加到线程池中被拒绝时，丢弃队列最后面的任务，然后将被拒绝的任务添加到等待队列后面
    - DiscardPolicy       --(0,1) 当任务添加到线程池中被拒绝时，线程池将丢弃被拒绝的任务
3. 线程池工作流程/原理
    - 如果workerCount < corePoolSize，则创建并启动一个线程来执行新提交的任务。
    - 如果workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添加到该阻塞队列中。
    - 如果workerCount >= corePoolSize && workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则创建并启动一个线程来执行新提交的任务。
    - 如果workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的处理方式是直接抛异常。
    - 如果线程池中线程数超过corePoolSize，且线程空闲下来时，超过空闲时间 就会被销毁，直到线程数==corePoolSize, 如果设置allowCoreThreadTimeOut=true,那么超过keepAliveTime时，低于corePoolSize数量的线程空闲时间达到keepAliveTime也会销毁
4. 什么情况下使用线程池  
   T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间


