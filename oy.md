## rocketMQ 2
事务消息
延时消息
commitLog
推拉
消息返回

## JVM

## java core

> final 类型的类变量，如果在编译时（转成.class文件）就可以确定，那么这个类变量就相当于“宏变量”，编译时，直接替换成值。所以，即使使用这个类变量，程序也不会导致该类的初始化！！----相当于直接使用 常量

* hashcode 根据内存地址hash

#### Object

* public final native Class<?> getClass();
* public native int hashCode();
* protected native Object clone() throws CloneNotSupportedException;
* public final native void notify();
* public final native void notifyAll();
* public final native void wait(long timeout) throws InterruptedException;
* public boolean equals(Object obj)
* public String toString() { return getClass().getName() + "@" + Integer.toHexString(hashCode());}
* protected void finalize() throws Throwable { }

#### System

* exit() 唯一一个能够退出程序并不执行finally的情况

#### String
* 内部维护了char[] 

* final
    * 首先String被许多Java类用来当参数，如果字符串可变，那么会引起各种严重错误和安全漏洞。
    * 再者String作为核心类，很多的内部方法的实现都是本地调用的，即调用操作系统本地API，其和操作系统交流频繁，假如这个类被继承重写的话，难免会是操作系统造成巨大的隐患。
    * 最后字符串的不可变性使得同一字符串实例被多个线程共享，所以保障了多线程的安全性。而且类加载器要用到字符串，不可变性提供了安全性，以便正确的类被加载。

* String.intern()
    * java8常量池移到堆中
    
### 多线程

#### Thread 


* 状态：就绪，运行，阻塞，死亡
* yield()
    * 停止当前线程，让同等优先权的线程运行。如果没有同等优先权的线程，那么该方法将不会起作用。
* sleep()
* wait()
    * 进入到一个和该对象相关的等待池中，同时失去了对象的机锁
    * wait的方法实现，lock.wait()方法最终通过ObjectMonitor的void wait(jlong millis, bool interruptable, TRAPS);实现
        1. 将当前线程封装成ObjectWaiter对象node
        2. 通过ObjectMonitor::AddWaiter方法将node添加到_WaitSet列表中
        3. 通过ObjectMonitor::exit方法释放当前的ObjectMonitor对象，这样其它竞争线程就可以获取该ObjectMonitor对象
        4. 最终底层的park方法会挂起线程
        
* notify() notifyAll() 
    * 等待池中的线程被放到锁池中，从锁池中获得机锁，然后回到wait()前的中断现场
* run()
* start()
* interrupt()
    * 打断wait、sleep线程的暂停状态，从而使线程立刻抛出InterruptedException。需要注意的是，InterruptedException是线程自己从内部抛出的，并不是interrupt()方法抛出的。对某一线程调用interrupt()时，如果该线程正在执行普通的代码，那么该线程根本就不会抛出InterruptedException。但是，一旦该线程进入到wait()/sleep()/join()后，就会立刻抛出InterruptedException。
* join()
    * 使当前线程停下来等待，直至另一个调用join方法的线程终止。
* ~~stop()~~ 终止线程。 会破坏synchronized原子性
* ~~suspend()~~ 挂起线程 挂起不会释放锁
* resume() 重启线程
* setPriority() 优先级
* activeCount() 活动线程的当前线程的线程组中的数量
* enumerate() 复制当前线程组线程到指定数组
* LockSupport.park() 为了线程调度，禁用当前线程，除非许可可用。
* LockSupport.unpark() 如果给定线程的许可尚不可用，则使其可用。

> Synchronized代码块通过javap生成的字节码中包含monitorenter和monitorexit两个指令。在进如锁的时候会执行monitorenter，执行monitorenter指令可以获取对象的monitor。同时在执行Lock.wait()的时候也必须持有monitor对象。

> 首先在HotSpot虚拟机中，monitor采用ObjectMonitor实现，每个线程都具有两个队列，分别为free和used，用来存放ObjectMonitor。如果当前free列表为空，线程将向全局global list请求分配ObjectMonitor。
  
> ObjectMonitor对象中有两个队列，都用来保存ObjectWaiter对象，分别是_WaitSet 和 _EntrySet。_owner用来指向获得ObjectMonitor对象的线程

> ObjectWaiter对象是双向链表结构，保存了_thread（当前线程）以及当前的状态TState等数据， 每个等待锁的线程都会被封装成ObjectWaiter对象。

> 中断线程最好的，最受推荐的方式是，使用共享变量（shared variable）发出信号，告诉线程必须停止正在运行的任务。线程必须周期性的核查这一变量（尤其在冗余操作期间），然后有秩序地中止任务。

> stop() 停止一个线程会导致它解锁它所锁定的所有monitor（当一个ThreadDeath Exception沿着栈向上传播时会解锁monitor），如果这些被释放的锁所保护的objects有任何一个进入一个不一致的状态，其他将要访问该objects的线程也会以一种不一致的状态来访问这些objects。这种objects称为“被损坏了”。

* 如何避免死锁
    * 锁顺序
    * 非阻塞锁超时
    * 不使用锁
    * 乐观锁
    * 数据不共享 ThreadLocal


#### 线程池

添加任务时，先达到核心线程数，然后放到队列中，然后创建至最大线程数处理新任务

阻塞队列
cache->keepAliveTime 60s
如果允许为核心池中的线程设置存活时间，那么核心池中的线程空闲时间超过keepAliveTime，线程也会被终止。

* 使用woke封装任务。当前任务不为空或getTask()队列有任务则继续执行。getTask()里会判断，如果当前线程池状态>=STOP 或者线程池状态为shutdown并且工作队列为空则，减少工作线程个数。
* 调用shutdown后，线程池就不会在接受新的任务了，但是工作队列里面的任务还是要执行的，但是该方法立刻返回的，并不等待队列任务完成在返回。仅仅设置下线程的中断标志，如果线程内没有使用中断标志做一些事情，那么这个对线程没有影响。
* 调用shutdown后，线程池就不会在接受新的任务了，并且丢弃工作队列里面里面的任务，正在执行的任务会被中断，但是该方法立刻返回的，并不等待激活的任务执行完成在返回。返回队列里面的任务列表。
  


### 并发

* Semaphore： acquire()获得， release()释放

* CyclicBarrier屏障：阻塞线程都需要达到屏障数，才会释放

* CountDownLatch：阻塞，直到计数减到0

* Condition

### 集合

#### HashMap *
* 数组长度是2的倍数

    * hash与数组长度-1做二进制与运算，结果一定是在原来数组大小的范围内，不会会造成空间浪费
    
    * 扩容后相当与后面新的节点，（经过rehash之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置）索引变成“原索引+oldCap”，省去了重新计算hash值的时间
    * 索引计算公式为i = (n - 1) & hash，如果n为2次幂，那么n-1的低位就全是1，哈希值进行与操作时可以保证低位的值不变，从而保证分布均匀，效果等同于hash%n，但是位运算比取余运算要高效的多

* 扩容 0.75
    * 元素在哈希表中分布的桶频率服从参数为0.5的泊松分布

* 1.7和1.8的区别 红黑树

* 如何实现线程安全

* 获取hash值时：为何在hash方法中加上异或无符号右移16位的操作？
    * 此方式是采用"扰乱函数"的解决方案，将key的哈希值，进行高16位和低16位异或操作，增加低16位的随机性，降低哈希冲突的可能性。
    
    
* 为何单链表转化为红黑树，要求节点个数大于8？
    * 一个bin中链表长度达到8个元素的概率为0.00000006，几乎是不可能事件。概率统计决定的
 
* 为何转化为红黑树前，要求数组总长度要大于64？
 
* 为何红黑树转化为单链表，要求节点个数小于等于6？
### ThreadLocal

* ThreadLocal底层是通过threadLocalMap进行存储键值 每个ThreadLocal类创建一个Map，然后用线程的ID作为Map的key，实例对象作为Map的value，这样就能达到各个线程的值隔离的效果。
* ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。
谁设置谁负责移除
* static remove的时候就可以正确进行定位到并且删除

* ThreadLocalMap
    * nextHashCode() & x 冲突替换或向后移继续查找
    * ThreadLocalMap中出现哈希冲突时，它是线性探测，直到找到空的位置。这种效率是非常低的，那为什么Java大神们写代码时还要这么做呢？笔者认为取决于它采用的哈希算法，正因为nextHashCode()，保证了冲突出现的可能性很低。而且ThreadLocalMap在处理过程中非常注意清理"stale entry"，及时释放出空余位置，从而降低了线性探测带来的低效。

* FastThreadLocal

### NIO 
#### NIO 1

* 事件驱动模型
* 避免多线程
* 单线程处理多任务
* 非阻塞I/O，I/O读写不再阻塞，而是返回0
* 基于block的传输，通常比基于流的传输更高效
* 更高级的IO函数，zero-copy
* IO多路复用大大提高了Java网络应用的可伸缩性和实用性


* 多路复用 
* 空转 1
* 缓冲区
* 通道

### 反射

* Java的反射就是利用加载到jvm中的.class文件来进行操作的。.class文件中包含java类的所有信息，当你不知道某个类具体信息时，可以使用反射获取class，然后进行各种操作。

* Java反射就是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；并且能改变它的属性。总结说：反射就是把java类中的各种成分映射成一个个的Java对象，并且可以进行操作。

* Class.forName()会连接，ClassLoader.loadClass(name,false)不会,所以也不会initialize。jdbc使用Class.forName()，spring 使用loadClass

* jdk2 后自定义加载器必须重写findClass() 

### 代理模式

* 静态代理
    * 基于继承的静态代理
    * 基于接口的静态代理
    * 优点：业务类只需要关注业务逻辑本身，保证了业务类的重用性。这是代理模式的共有优点。
    * 缺点：
        1. 代理对象的一个接口只服务于一种类型的对象，如果要代理的类型很多，势必要为每一种类型的方法都进行代理，静态代理在程序规模稍大时就无法胜任了。
        2. 如果接口增加一个方法，除了所有实现类需要实现这个方法外，所有代理类也需要实现此方法。显而易见，增加了代码维护的复杂度。 
           

* 动态代理

    > 动态代理是在实现阶段不用关心代理类，而在运行阶段才指定哪一个对象。
    >
    > 动态代理类的源码是在程序运行期间由JVM根据反射等机制动态的生成 。
    
    * 原理：
        * Java动态代理通过创建接口的实现类来完成对目标对象的代理，使用Java原生的反射API进行操作，在生成类上比较高效；但不能代理未实现接口的类；
        * CGLIB 在运行期间生成的是目标类扩展的子类，直接使用ASM框架直接对字节码进行操作，在类的执行过程中比较高效；不能扩展final修饰的类或方法；
        
    * 使用：
        * Java动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为Proxy，Java类继承机制不允许多重继承）；
        * Java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理；
        
        * CGLIB能够代理普通类，但是不能代理final修改时的类；
        * CGLIB动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。
        
    * 区别：
        1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP 
        2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP ,spring中配置proxy-target-class='true'
        3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换
        

## eureka 2
* 多级缓存
* 心跳
* 注册

## redis 2
* 分布式锁 1
* 缓存
    * 写缓存失败
        * 设置过期时间，删除缓存
        * （先缓存再DB，主从同步问题）订阅binlog更新缓存
    * 先写缓存还是先写数据库
    * 缓存雪崩
        * 缓存时间随机
    * 缓存击穿
        * 加锁
    * 缓存穿透
        * 缓存0、null
    * 懒加载式
    * 定时刷新
    * 大Value 缓存
        * 拆分value
        * 多线程缓存Memcached
    * 热点缓存
        * 增加从缓存
* 集群实现
    * 主从同步
    
        * 从服务器连接主服务器，发送SYNC命令； 
        * 主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
        * 主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
        * 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
        * 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
        * 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；（从服务器初始化完成）
        * 主服务器每执行一个写命令就会向从服务器发送相同的写命令，从服务器接收并执行收到的写命令（从服务器初始化完成后的操作）
        
        优点：读写分离，非阻塞同步数据
        
        缺点：无容错、恢复，不易扩容
    * 哨兵
        * 哨兵向其他机器发射PING
        * 检查到一台实例断开或超时则主观下线，多台则客观下线，不提供服务
        * 重新PING通则恢复状态
        
        优点：主从可以自动切换
        
        缺点：不易扩容
        
    * Redis-Cluster集群
        
        * 0-16383 hash槽，对key运算crc16 得到结果，然后对16383取余，决定在哪个节点上。
        
    * codis

        * 连接代理，访问实例。部分命令不可用
        
        * 依赖 ZooKeeper 来存放数据路由表和 codis-proxy 节点的元信息, codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy
* 底层数据结构
    * long 类型的整数
    * embstr 编码的简单动态字符串
    * 简单动态字符串
    * 字典
    * 双端链表
    * 压缩列表
    * 整数集合
    * 跳跃表和字典
* Lua脚本实现

## hystrix
* rx支持
* 漏桶算法／令牌桶算法 2
* 线程池／信号量　





## netty 

* 内存池化
* FastThreadLocal
* NIO 3

## 设计模式 2

* 装饰者模式
* 工程模式
* 策略模式
* 代理模式：为其他对象提供一种代理以控制对这个对象的访问，可以隔离客户端和委托类的中介。我们还可以借助代理来在增加一些功能，而不需要修改原有代码。
* 模板模式 spring


* 策略模式
* 简单工厂模式
* 工厂模式
* 抽象工厂模式
* 装饰者模式
* 代理模式
* 模板方法模式
* 外观模式
* 适配器模式
* 桥接模式
* 建造者模式
* 观察者模式
* 单例模式
* 单例模式
* 命令模式


## zookeeper
* 选举 1
* CAP
    * 一致性
    * 分区容错性
    * 可用性
* 分布式锁实现
    * 锁对应目录，在该目录下创建节点（zk自己保证大小顺序），该节点有时间限制
    * 最小的节点获取锁
    * 释放锁删除该节点
    * 锁获取失败，则订阅上一个较小的节点，删除时通知到
* watch机制

## 算法 4
* 归并算法
* 快速排序
* 二分法

## mysql 2
* B+树索引
    * 如果where查不到索引则全表搜索，update where 有索引行锁，没索引表锁，数据少表锁
* sql优化 1
* 集群实现、读写分离
* 事务
    ACID
    * 原子性 Atomicity
    * 一致性 Consistency
    * 隔离性 Isolation
        * 读未提交
        * 读已提交，防止脏读  脏读：未提交的数据被其他事务读取了  解决：commit
        * 可重复读，防止不可重复 不可重复：事务多次读取其它事务的数据，结果不一致 解决：行锁
        * 串行化，防止幻读 幻读：根据范围读取数据，第二次再次根据范围读取数据，发现多了一行  解决:锁表
    * 持久化 Durability
* 锁
    
    * 行锁
        * 共享锁(S)：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。ps: SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE。
        * 排他锁(X)：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。 ps: SELECT * FROM table_name WHERE ... FOR UPDATE。
    * 表锁
        * 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
        * 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。
    * 间隙锁
        * 
    * 注意
        * InnoDB行锁是通过给索引上的索引项加锁来实现的，这一点MySQL与Oracle不同，后者是通过在数据块中对相应数据行加锁来实现的。InnoDB这种行锁实现特点意味着：只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！
        * 由于MySQL的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是如果是使用相同的索引键，是会出现锁冲突的。
        * 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引
    * 乐观锁
    * MVCC 多版本并发控制
        * 行级锁
        * InnoDB在每行数据都增加两个隐藏字段，一个记录创建的版本号，一个记录删除的版本号。（DB_TRX_ID，DB_ROLL_PTR，DB_ROW_ID）
        * InnoDB只查找版本早于当前事务版本的数据行(也就是数据行的版本必须小于等于事务的版本)，这确保当前事务读取的行都是事务之前已经存在的，或者是由当前事务创建或修改的行
        * 读不加锁，读写不冲突
        * innodb MVCC主要是为Repeatable-Read事务隔离级别做的
        * 不是真正的删除数据，而是标志出来的删除。真正意义的删除是在commit的时候
        * MVCC是通过保存数据在某个时间点的快照来实现的. 不同存储引擎的MVCC. 不同存储引擎的MVCC实现是不同的,典型的有乐观并发控制和悲观并发控制.
* 索引
    * 聚集索引
        * 自增主键
    * 非聚集索引
        * 通过普通索引查到key，再通过聚集索引查到数据
    * 覆盖索引
        * 联合索引

## 分布式事务
* 二段提交
    * 阻塞占有资源，协调者通知参与者是否提交
    
* 三段提交
    * 非阻塞，增加预提交流程，避免协调者出问题
* tcc
    * 业务预留资源状态位，try执行验证数据库服务等没有问题，confirm确认执行，cancel回滚状态。若超时重试调用，实现幂等性。

## spring 2
* springMVC 1
* bean生命周期
* IOC／AOP

## 网络通信 4
* TCP／ HTTP
* 三次握手
    * 第一次握手：建立连接时,客户端发送syn包(syn=j)到服务器,并进入SYN_SEND状态,等待服务器确认； SYN：同步序列编号(Synchronize Sequence Numbers)
    * 第二次握手：服务器收到syn包,必须确认客户的SYN（ack=j+1）,同时自己也发送一个SYN包（syn=k）,即SYN+ACK包,此时服务器进入SYN_RECV状态； 
    * 第三次握手：客户端收到服务器的SYN＋ACK包,向服务器发送确认包ACK(ack=k+1),此包发送完毕,客户端和服务器进入ESTABLISHED状态,完成三次握手.
    * 完成三次握手,客户端与服务器开始传送数据
* 四次挥手
    * 第一次挥手：Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
    * 第二次挥手：Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
    * 第三次挥手：Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
    * 第四次挥手：Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。
* 5／7 层协议
    * 应用层
    * 表示层
    * 绘画层
    * 运输层
    * 网络层
    * 数据链路层
    * 物理层

## 运维 4
### Linux
### nginx
* 一致性Hash:计算服务器ip地址对应的hash值，落在0~最大正整数，请求的ip hash也落在上面，顺时针访问最近的一个服务器IP。

### jmap/jstack 2
### arthx

## 架构能力 3

## 问题解决能力 2

## 项目能力 1

Round 1
object 有哪些方法
hashcode 应用
环型链表检测
thread 方法
wait实现原理
native 方法好处


http／tcp
缓存击穿

为什么

Round 2
秒杀 库存
库存回流
redis 
分布式锁
砍砍购
防刷

Round 3
下单 库存 分布式事务 事务消息
dubbo
NIO BIO区别
jvm
内存模型，类型引用
垃圾回收算法 cms

Round 4
缓存 eacache
dubbo调用流程、负载均衡
redis zset 结构
分布式锁
数据库事务，spring事务实现
hasmap curMap
线程池
jvm
线上问题排查方式 arthas

Round 5

threadLocal 为什么线性查找
hashmap 1.7 1.8 区别
线程池丢弃策略
spring core里的类 生命周期 设计模式 aop 单例
dubbo 调用过程
算法 100万找1个数
redis内存满了
不依靠第三方实现分布式事务
类加载 卸载的条件
roekctMQ 队列 发消息 实现订阅
i++ 为什么不是原子性的

必考题
hashmap
redis
dubbo
事务
