## java基础

### java基础知识

#### 多态

1. 所谓多态就是指程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定，
   即一个引用变量到底会指向哪个类的实例对象，该引用变量发出的方法调用到底是哪个类中实现的方法，必须在由程序运行期间才能决定。
   因为在程序运行时才确定具体的类，这样，不用修改源程序代码，就可以让引用变量绑定到各种不同的类实现上，从而导致该引用变量调用的具体方法随之改变，
   即不修改程序代码就可以改变程序运行时所绑定的具体代码，让程序可以选择多个运行状态，这就是多态性。
2. java实现多态的三个必要条件：
   1. 继承:在多态中必须存在有继承关系的子类和父类。
   2. 重写:子类对父类中某些方法进行重新定义，在调用这些方法时就会调用子类的方法。
   3. 向上转型:父类引用指向子类对象

#### 抽象类和接口

1. 抽象类要被子类继承，接口要被类实现
2. 接口只能做方法声明，抽象类中可以作方法声明，也可以做方法实现。（java 1.8接口可以包含默认实现）
3. 接口里定义的变量只能是公共的静态的常量，抽象类中的变量是普通变量。
4. 抽象类可以有具体的方法和属性，接口只能有抽象方法和不可变常量。
5. 使用场景
   1. 在既需要统一的接口，又需要实例变量或缺省的方法的情况下，就可以使用它
   2. 大多数情况下组合大于继承 因为java的是单继承模式
6. 父类的静态方法不能被子类重写 可以被子类继承使用

*静态内部类不持有外部内的引用 不会造成内存泄漏的风险 在Android常用于handler对象使用

#### final，finally，finalize的区别

1. final final修饰类表示类不能被继承 修饰方法方法不能被子类重写 修饰变量类似于const 只能被赋值一次 （直接声明时赋值或者在类构造方法里赋值）
2. finally 配合try catch表示一定要执行的语句
3. finalize object类定义的 一般不需要重写 在GC时会被调用 (Android中体现在bitmap回收native层像素数据时会用到)

#### 序列化

1. Serializable 继承接口即可  子类可继承 会自动序列化非static transient变量
2. Parcelable 由于Serializable IO操作频繁 Parcelable性能更好 主要用于内存间传递数据，不适合本地持久化或者网络传输，需要实现其抽象方法(Android特有，binder传输中的数据就是采用该形式)

#### 代理模式

##### 静态代理

一种设计模式 [代理](https://blog.csdn.net/asd051377305/article/details/80490432)

##### 动态代理

1. 创建一个类继承InvocationHandler类 同时重写其构造方法传入具体实现对象类
2. 调用Proxy.newProxyInstance 获取具体实现类，参数一传入classloader 参数二为对象类的接口class 参数三InvocationHandler （在android中retrofit是典型应用场景）

#### ThreadLocal

1. ThreadLocal是通过每个线程单独一份存储空间，牺牲空间来解决冲突，并且相比于Synchronized，ThreadLocal具有线程隔离的效果，只有在线程内才能获取到对应的值，线程外则不能访问到想要的值。（在android中的典型应用场景即为Looper  同时在perpare前先get是否存在来保证线程唯一）

2. 每个线程Thread类中又一个ThreadLocalMap用于存放当前线程的ThreadLocal变量  其中每个元素Entry包含key value 属性 其set get代码如下 很好理解

   ```java
   //set 方法
   public void set(T value) {
         //获取当前线程
         Thread t = Thread.currentThread();
         //实际存储的数据结构类型
         ThreadLocalMap map = getMap(t);
         //如果存在map就直接set，没有则创建map并set
         if (map != null)
             map.set(this, value);
         else
             createMap(t, value);
     }
   
   //ThreadLocal中get方法
   public T get() {
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null) {
           ThreadLocalMap.Entry e = map.getEntry(this);
           if (e != null) {
               @SuppressWarnings("unchecked")
               T result = (T)e.value;
               return result;
           }
       }
       return setInitialValue();
   }
   ```

#### 注解

##### java元注解

​	注解的注解

1. @Target 用于描述注解作用域

   - CONSTRUCTOR:用于描述构造器
   - FIELD:用于描述域即类成员变量
   - LOCAL_VARIABLE:用于描述局部变量
   - METHOD:用于描述方法
   - PACKAGE:用于描述包
   - PARAMETER:用于描述参数
   - TYPE:用于描述类、接口(包括注解类型) 或enum声明
   - TYPE_PARAMETER:1.8版本开始，描述类、接口或enum参数的声明
   - TYPE_USE:1.8版本开始，描述一种类、接口或enum的使用声明

2. @Retention 用于描述注解生命周期

   - SOURCE:在源文件中有效（即源文件保留）
   - CLASS:在class文件中有效（即class保留）
   - RUNTIME:在运行时有效（即运行时保留）

3. @Documented

   用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化。它是一个标记注解，没有成员

4. @Inherited

   用于表示某个被标注的类型是可以被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

##### 运行时注解的基本Api

1. class.isAnnotationPresent() 用于获取class是否被指定注解所修饰

2. class.getAnnotation()根据注解class获取注解的实例对象

3. class.getAnnotations()获取当前类对象的所有修饰注解数组

   *field method同理 根据反射获取即可

   getDeclaredXXX代表的含义是不包含继承的方法 、注释

#### 反射

##### [反射的含义](https://www.jianshu.com/p/9be58ee20dee)

JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意方法和属性；在日常开发中应用广泛 多用于hook系统类 方法 属性 （gson 序列化、retrofit等等均有实际应用）性能开销较大 

#### 泛型

泛型的用处和优点

1. 类型安全 

   泛型的主要目标是实现java的类型安全。 泛型可以使编译器知道一个对象的限定类型是什么，这样编译器就可以在一个高的程度上验证这个类型 在我们写代码的时候就会及时提示错误

2. 消除了强制类型转换 

   使得代码可读性好，减少了很多出错的机会

泛型的实现原理

主要靠的是类型擦除技术 编译器在编译的时候会擦除掉所有类型相关的信息，因此在运行期是无法获取到泛型对象的类信息的（kotlin inline方法可以使用reified关键字来获取）



### 集合（collection）

#### List接口

1. Arraylist

   动态数组实现 

   增删元素可动态改变数组长度

   线程不安全

   操作效率高 多用于查询修改

   查询修改效率高是因为基于数组实现 直接通过角标即可实现

   增删效率低是因为需要动态进行数组扩容和拷贝

2. Vector

   特殊的动态数组 

   实现了同步 线程安全

   其他同ArrayList

3. Linkedlist

   基于双向链表实现

   增删效率高（修改指针即可）

   改查效率低（需遍历链表）

   线程不安全

   同时实现了栈和队列接口

#### Set接口

1. HashSet

   实现了Set接口

   没有重复元素 无序

   底层通过HashMap的key的集合作为容器存储数据对象

   线程不安全

2. TreeSet

   实现SortedSet接口

   没有重复元素，有序

   不支持快速随机遍历，只能通过迭代器进行遍历，

   底层使用TreeMap作为存储数据对象，

   TreeSet支持两种排序方式，自然排序 和定制排序，其中自然排序为默认的排序方式。

#### Map接口

1. HashMap

   使用hash散列表思想

   本身基于数组+链表实现

   无序

   键值不重复 value可重复

   运行null键和null值

   线程不安全

   多线程访问可能导致死锁 同时put过多时get会引发环形指针导致死循环

   通过链表法解决hash冲突问题 java1.8当链表长度>8时会将链表转换为红黑树提高遍历效率 

2. LinkedHashMap

   有序的Map集合类 在hashMap的基础上加入了双向链表

   默认链表顺序是插入顺序 可通过构造方法传值改为访问顺序

   在Android端多用于LruCache实现

3. Hashtable

   Hashtable继承自Dictionary类，而HashMap继承自AbstractMap类。但二者都实现了Map接口。

   和HashMap基本相似 区别之处在于实现了同步 线程安全

   不允许null键

4. TreeMap

   有序的集合

   通过红黑树实现

   线程不安全

5. concurrentHashMap

   java1.8支持并发的HashMap

   采用reentranklock+分段锁实现

   写操作同步 读操作不同步 效率较高

#### *集合框架用两个实用类collections（排序，复制、查找）和Arrays对数组进行（排序，复制、查找）

#### Android端的特殊集合容器

1. ArrayMap

   基于两个动态数组实现

   相对于hashmap牺牲性能换时间

   长度大小没有空余位置

   增删效率慢 因为牵扯到整个数组的复制移位

   遍历通过第一个mArray数组获取键 然后二分查找第二个arrays的所有值（数组为有序数组）在数量较大的情况下效率低于hashmap 

2. SparseArray

   在ArrayMap的基础上对key的类型进行优化 针对基本类型避免了拆装箱操作 进一步优化性能

### java内存模型

#### jvm内存分区

1. 方法区

   线程共享

   生命周期与进程相同

   内存地址可以不连续

   存储已被虚拟机加载的类信息、常量、静态变量、方法等

   *方法区还包括了字符常量池 用来存放字符串和符号

2. 堆

   线程共享

   生命周期与进程相同

   内存地址可以不连续

   用来保存对象实例

   GC回收的区域

3. jvm栈

   线程私有

   使用连续内存地址

   Java 方法执行的内存模型，存储局部变量表、操作栈、动态链接、方法出口等信息

4. native栈

   和jvm栈类似

   区别在于存放的是native内存模型

5. 程序计数器

   用来记录字节码行号执行位置

   占用内存很小

   线程私有

   生命周期和线程一致 不需要关注gc问题

6. JMM

   每个线程都有一份自己的本地内存，它包含了进程堆上主内存的共享变量副本，由JMM控制 多线程不安全的原因也在这里，volatile关键字的作用是当共享变量副本发生改变时会立刻刷新到主内存去，保证了可见性

#### GC

[GC原理]: https://www.jianshu.com/p/2c997ec4505a

GcRoot的种类

1.虚拟机栈：栈帧中的本地变量表引用的对象

2.native方法引用的对象

3.方法区中的静态变量和常量引用的对象

堆内存区域划分为两块 新生代/老年代

1. 新生代

   使用复制算法 保证了内存的连续性

   包含有1个eden区和两个survivor区

   每次GC时都是将eden区和其中一个survivor区

   的对象复制到另一个空survivor区 然后清除掉本身

2. 老年代

   使用标记整理算法 保证内存连续性

   对象进入老年代的条件

   1.在新生代中活过15次GC的对象

   2.大对象（数组 集合等）直接进入老年代

3. android内存抖动

   参数：
   
   [heapgrowthlimit]() -  是一般状况下 vm 最大内存限制，多了就 OOM 了
   
   [heapmaxfree]() - 是在配置文件中设置 largeHeap="true" 后 vm 能获得的最大内存，超了一样 OOM
   
   [heapmaxfree]() - 最大内存空闲值，超了会触发 GC，同时调整内存上限
   
   [heapminfree]() - 最小内存空闲值，不够同样会触发 GC，GC 后再不够会调整内存上限
   
   [heapstartsize]() - vm 启动时申请的内存大小
   
   [heaptargetutilization]() - 内存期望利用率
   
   理论上堆的大小应该 - LiveSize 实际使用量 / heaptargetutilization，但是这个值是有限制的，必须 >= LiveSize + MinFree，<=  LiveSize + MaxFree，否则就要进行调整，调整的其实是 vm 内存上限 softLimit
   
   - 要申请的内存空间 > 当前空余内存，会先 GC,要是不够会动态调整内存上限，一次增加 heapmaxfree 的量
   - 当前空余内存 - 要申请的内存空间 < heapminfree，会先 GC,要是不够会动态调整内存上限
   - 当前空余内存 - 要申请的内存空间在 heapmaxfree - heapminfree 之间，不会触发 GC
   
   这频繁的触发 GC  就称为内存抖动
   
   可能引发内存抖动的地方
   
   1.循环体内new对象
   
   2.在自定义viewondraw方法中创建对象
   
   
   
### 多线程

#### 线程创建方法

1. 继承Thread类，并复写run方法，创建该类对象，调用start方法开启线程。
2. 实现Runnable接口，复写run方法，创建Thread类对象，将Runnable子类对象传递给Thread类对象。调用start方法开启线程。
3. 1.创建FutureTask对象，创建Callable子类对象，复写call(相当于run)方法，将其传递给FutureTask对象（相当于一个Runnable）。
    2.创建Thread类对象，将FutureTask对象传递给Thread对象。调用start方法开启线程。这种方式可以获得线程执行完之后的返回值。
4. tart方法和run方法的区别
   调用start方法方可启动线程后由java自身通过native启动线程后去在该线程里调用run方法执行，而run方法只是thread的一个普通方法调用，还是在当前工作线程里执行。

#### 进程和线程的区别

1. 进程是程序资源分配的最小单位，线程是程序执行的最小单位。
2. 进程有自己的内存地址空间，线程包含在进程的地址空间中。
3. 相对于进程与进程之间线程之间通信方式比较方便，线程能共享进程分配到的逻辑内存的资源。也就是说，同一进程下的线程共享全局变量、静态变量等数据，
4. 进程的分配开销比线程大，但是进程的健壮性比线程高，因为进程间不会互相影响，线程一个挂掉了可能会造成进程崩溃。

#### 如何控制某个方法允许并发访问线程的个数

​	使用Semaphore类 

​	sSemaphore = new Semaphore(6) 表示该方法允许几个线程访	问 sSemaphore.acquire():调用一次，允许进入方法的线程数量减	一
​	sSemaphore.release()：调用一次，允许进入方法的线程数量加1

​	sSemaphore.availablePermits()：可以获取当前允许进入方法的	线程数量

​	当允许进入方法的线程数量减为0的时候其他线程就等待，直到允	许的线程数量大于0 

#### wait和sleep的区别

​	wait是object类的方法 阻塞线程后需要主动调用notify去唤醒 作	用于对象 会释放资源 只能在同步代码块里调用

​	sleep是Thread的方法 阻塞后等待时间到了自动继续执行 可以尝	试调用interrupt来直接唤醒 不会释放锁资源 作用于线程

#### Java中实现线程阻塞的方法

1. 线程睡眠：Thread.sleep?(long millis)方法，使线程转到阻塞状态。millis参数设定睡眠的时间，以毫秒为单位。当睡眠结束后，就转为就绪（Runnable）状态。sleep()平台移植性好。
2. 线程等待：Object类中的wait()方法，导致当前的线程等待，直到其他线程调用此对象的 notify() 唤醒方法。这个两个唤醒方法也是Object类中的方法，行为等价于调用 wait() 一样。wait() 和 notify() 方法：两个方法配套使用，wait() 使得线程进入阻塞状态，它有两种形式，一种允许 指定以毫秒为单位的一段时间作为参数，另一种没有参数，前者当对应的 notify() 被调用或者超出指定时间时线程重新进入可执行状态，后者则必须对应的 notify() 被调用.
3. 线程礼让，Thread.yield()?方法，暂停当前正在执行的线程对象，把执行机会让给相同或者更高优先级的线程。yield() 使得线程放弃当前分得的 CPU 时间，但是不使线程阻塞，即线程仍处于可执行状态，随时可能再次分得 CPU 时间。调用 yield() 的效果等价于调度程序认为该线程已执行了足够的时间从而转到另一个线程.
4. 线程自闭，join()方法，等待其他线程终止。在当前线程中调用另一个线程的join()方法，则当前线程转入阻塞状态，直到另一个进程运行结束，当前线程再由阻塞转为就绪状态。

#### 停止线程的方法

1. 使用volatile类型的boolean变量控制跳出线程run方法
2. 调用Thread的interrupt方法尝试停止 如果线程是阻塞则直接退出 如果是非阻塞则还是要在run方法的适当位置加入isinterrupt判断来控制结束run方法
3. 调用线程池的shutdown和shutdownNow 区别在于shutdown会将当前线程队列里的任务执行完毕后才进行停止 shutdownNow会忽略当前队列里尚未执行的任务 同时调用interrupt去尝试结束当前线程，之后返回队列里未执行的任务列表

#### 线程安全在三个方面体现

1. 原子性：提供互斥访问，同一时刻只能有一个线程对数据进行操作，（atomic,synchronized）

2. 可见性：一个线程对主内存的修改可以及时地被其他线程看到，（synchronized,volatile）

3. 有序性：一个线程观察其他线程中的指令执行顺序，由于指令重排序，该观察结果一般杂乱无序，（happens-before原则）

   (volatile 禁止进行指令重排序)

*多线程安全的操作list可以用CopyOnWriteArrayList 缺点是内存占用高 不能保证实时一致性 只能保证最终一致性 读不同步 写是同步的（使用重入锁ReentrantLock实现） 如果写操作多则不适合 性能较差

#### synchronized

1. 修饰普通方法 一个对象中的加锁方法只允许一个线程访问。但要注意这种情况下锁的是访问该方法的实例对象， 如果多个线程不同对象访问该方法，则无法保证同步。（对象锁 针对具体的某一对象有效）
2. 修饰静态方法 由于静态方法是类方法， 所以这种情况下锁的是包含这个方法的类，也就是类对象；这样如果多个线程不同对象访问该静态方法，也是可以保证同步的。（类锁 对该类的所有对象都可以保证同步）
3. 修饰代码块 其中普通代码块 如Synchronized（obj） 这里的obj 可以为类中的一个属性、也可以是当前的对象，它的同步效果和修饰普通方法一样；Synchronized方法 （obj.class）静态代码块它的同步效果和修饰静态方法类似。

#### volatile关键字

1. 保持内存可见性 每次读取前必须先从主内存刷新最新的值。每次写入后必须立即同步回主内存当中。（多线程 线程中止控制变量）
2. 防止指令重排 （双重锁单例）

#### synchronized和volatile的区别

1. volatile只能作用于变量，使用范围较小。synchronized可以用在变量、方法、类、同步代码块等，使用范围比较广。
2. volatile只能保证可见性和有序性，不能保证原子性。而可见性、有序性、原子性synchronized都可以包证。 
3. volatile不会造成线程阻塞。synchronized可能会造成线程阻塞。

#### synchronized和ReentrantLock的区别

1. 首先synchronized是java内置关键字，在jvm层面，Lock是个java类；
2. synchronized无法判断是否获取锁的状态，Lock可以判断是否获取到锁；
3. synchronized会自动释放锁(a 线程执行完同步代码会释放锁 ；b 线程执行过程中发生异常会释放锁)，Lock需在finally中手工释放锁（unlock()方法释放锁），否则容易造成线程死锁；
4. 用synchronized关键字的两个线程1和线程2，如果当前线程1获得锁，线程2线程等待。如果线程1阻塞，线程2则会一直等待下去，而Lock锁就不一定会等待下去，如果尝试获取不到锁，线程可以不用一直等待就结束了；
5. synchronized的锁可重入、不可中断、非公平，而Lock锁可重入、可判断、可公平（两者皆可）

#### CAS类

1. Compare and Swap，比较并交换。CAS有3个操作数：内存值V、预期值A、要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

2. java中有一系列原始类型变量的CAS包装类 如AtomicInteger等

   常用在多线程并发中

#### 死锁的必要条件

1. 互斥 一个资源每次只能被一个进程使用，即在一段时间内某 资源仅为一个进程所占有。此时若有其他进程请求该资源，则请求进程只能等待。
2. 请求与保持 进程已经保持了至少一个资源，但又提出了新的资源请求，而该资源 已被其他进程占有，此时请求进程被阻塞，但对自己已获得的资源保持不放。（1.要么一次性请求到所有资源后再执行程序 2.在程序执行过程中及时释放资源后再申请新的资源）
3. 不可剥夺 进程所获得的资源在未使用完毕之前，不能被其他进程强行夺走，即只能 由获得该资源的进程自己来释放（只能是主动释放)。（同上 ）
4. 循环等待 若干进程间形成首尾相接循环等待资源的关系（尝试终止其中一个进程以打破循环等待状态）

#### 线程池

##### java四种标准线程池

1. [FixedThreadPool ](https://www.jianshu.com/p/4366011830f1)- 定长线程池，可控制线程最大并发数，超出的线程会在队列中等待

   ```java
   public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
           return new ThreadPoolExecutor(nThreads, nThreads,
                                         0L, TimeUnit.MILLISECONDS,
                                         new LinkedBlockingQueue<Runnable>(),
                                         threadFactory);
       }
   ```

   核心线程数和最大线程数相等 没有空闲等待时间（即任务执行完成后立刻销毁）采用无界有序阻塞队列

2. [SingleThreadExecutor ](https://www.jianshu.com/p/4366011830f1)- 单线程线程池，只用一个工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行

   ```java
   public static ExecutorService newSingleThreadExecutor() {
           return new FinalizableDelegatedExecutorService
               (new ThreadPoolExecutor(1, 1,
                                       0L, TimeUnit.MILLISECONDS,
                                       new LinkedBlockingQueue<Runnable>()));
       }
   ```

   核心线程数和最大线程数均为1 没有空闲等待时间 采用无界有序阻塞队列

3. [CachedThreadPool](https://www.jianshu.com/p/4366011830f1) - 可缓存线程池，线程数量没有限制，如果线程池数量超过任务，可灵活回收空闲线程，若无可回收，则新建线程

   ```java
   public static ExecutorService newCachedThreadPool() {
           return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                         60L, TimeUnit.SECONDS,
                                         new SynchronousQueue<Runnable>());
       }
   ```

   核心线程数为0 最大线程数可以理解为无限 超时时间60s 使用SynchronousQueue 阻塞队列（可以理解为每新增一个任务时如果当前没有空闲线程则马上为该任务开辟一个新线程进行处理，该队列的最大长度为1）

4. [ScheduledThreadPool](https://www.jianshu.com/p/4366011830f1) - 定时线程池，支持定时及周期性任务执行

   ```java
   public ScheduledThreadPoolExecutor(int corePoolSize) {
           super(corePoolSize, Integer.MAX_VALUE,
                 DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
                 new DelayedWorkQueue());
       }
   ```

   核心线程数为指定线程数 最大线程数无限，超时时间10ms 使用DelayedWorkQueue延迟阻塞队列 用于执行定时任务或者周期任务

##### 线程池状态

1. [RUNNING](https://www.jianshu.com/p/4366011830f1) - 线程池初始化后就是 RUNNING 状态，此时线程池中的任务为0
2. [SHUTDOWN](https://www.jianshu.com/p/4366011830f1) - 调了 shutdown 结束线程池，线程池就是 SHUTDOWN 状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕
3. [STOP](https://www.jianshu.com/p/4366011830f1) - 调了 shutdownNow 结束线程池，线程池就是 STOP 状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务
4. [TIDYING](https://www.jianshu.com/p/4366011830f1) - 所有的任务已终止，ctl 记录的”任务数量”为 0，线程池会变为 TIDYING 状态，接着会执行 terminated 函数

​    *关闭线程池的两种方法

1. [shutdown](https://www.jianshu.com/p/4366011830f1) - 只标记状态 SHUTDOWN，正在执行的任务会继续执行下去，没有被执行的则中断返回。
2. [shutdownNow ](https://www.jianshu.com/p/4366011830f1)- 将线程池的状态设置为 STOP，正在执行的任务则被尝试停止，没被执行任务的则返，注意这样线程池此时若是 sheep 的话会抛异常

##### 线程池大小配置规则

1. CPU密集型 需要尽量压榨 CPU，参考值可以设为 CPU核数 + 1
2. IO密集型 参考值可以设置为 2* CPU核数（android中一般为IO密集型）

##### 线程池小知识点

1. 线程工厂[ThreadFactory]一般用于自定义线程创建规则（例如线程命名 是否是守护线程 是否和非核心线程一样使用超时规则等等）

   *守护线程thread.setdeamon（true）表示和父线程同生共死 当父线程结束时守护线程也会终止 否则会一直执行到结束

2. 拒绝策略[RejectedExecutionHandler] 

   AbortPolicy - 默认饱和策略，超出队列长度后直接抛出异常

   CallerRunsPolicy - 只用调用者所在线程来运行任务

   DiscardPolicy - 不处理，丢弃掉，也不会日志什么的

   DiscardOldestPolicy - 丢弃队列里最近的一个任务，并执行当前任务。

   自定义Police - 需要实现RejectedExecutionHandler

#### 生产者消费者模型（java实现）

```java
public class Test1 {
    private static Integer count = 0;
    private static final Integer FULL = 10;
    private static String LOCK = "lock";
    
    public static void main(String[] args) {
        Test1 test1 = new Test1();
        new Thread(test1.new Producer()).start();
        new Thread(test1.new Consumer()).start();
        new Thread(test1.new Producer()).start();
        new Thread(test1.new Consumer()).start();
        new Thread(test1.new Producer()).start();
        new Thread(test1.new Consumer()).start();
        new Thread(test1.new Producer()).start();
        new Thread(test1.new Consumer()).start();
    }
    //生产者
    class Producer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(3000);
                } catch (Exception e) {
                    e.printStackTrace();
                }
                synchronized (LOCK) {
                    while (count == FULL) {
                        try {
                            LOCK.wait();
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                    count++;
                    System.out.println(Thread.currentThread().getName() + "生产者生产，目前总共有" + count);
                    LOCK.notifyAll();
                }
            }
        }
    }
    //消费者
    class Consumer implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (LOCK) {
                    while (count == 0) {
                        try {
                            LOCK.wait();
                        } catch (Exception e) {
                        }
                    }
                    count--;
                    System.out.println(Thread.currentThread().getName() + "消费者消费，目前总共有" + count);
                    LOCK.notifyAll();
                }
            }
        }
    }
}
```

### 常用设计模式

#### 1.观察者模式

- Android中使用最广泛的一种设计模式，点击事件监听 handler okhttp网络回调等等均有实际应用

  ```java
  //实体
  public class Weather {  
   
      private String description;  
    
      public Weather(String description) {  
          this.description = description;  
      }  
    
      public String getDescription() {  
          return description;  
      }  
    
      public void setDescription(String description) {  
          this.description = description;  
      }  
  } 
  //观察者管理器
  public class Observable<T> {  
      List<Observer<T>> mObservers = new ArrayList<Observer<T>>();  
    	//注册观察者
      public void register(Observer<T> observer) {  
          if (observer == null) {  
              throw new NullPointerException("observer == null");  
          }  
          synchronized (this) {  
              if (!mObservers.contains(observer))  
                  mObservers.add(observer);  
          }  
      }  
    	//解除注册观察者
      public synchronized void unregister(Observer<T> observer) {  
          mObservers.remove(observer);  
      }  
    	//通知观察者变化
      public void notifyObservers(T data) {  
          for (Observer<T> observer : mObservers) {  
              observer.onUpdate(this, data);  
          }  
      }  
  } 
  //观察者接口
  public interface Observer<T> {  
      void onUpdate(Observable<T> observable,T data);  
  } 
  
  //使用
  public static void main(String [] args){  
          Observable<Weather> observable=new Observable<Weather>();  
          Observer<Weather> observer1=new Observer<Weather>() {  
              @Override  
              public void onUpdate(Observable<Weather> observable, Weather data) {  
                  System.out.println("观察者1："+data.toString());  
              }  
          };  
          Observer<Weather> observer2=new Observer<Weather>() {  
              @Override  
              public void onUpdate(Observable<Weather> observable, Weather data) {  
                  System.out.println("观察者2："+data.toString());  
              }  
          };  
    
          observable.register(observer1);  
          observable.register(observer2);  
  
          Weather weather=new Weather("晴转多云");  
          observable.notifyObservers(weather);  
    
          Weather weather1=new Weather("多云转阴");  
          observable.notifyObservers(weather1);  
    
          observable.unregister(observer1);  
    
          Weather weather2=new Weather("台风");  
          observable.notifyObservers(weather2);  
    
      }  
  ```

#### 2.代理模式

- 静态代理

  ```java
  //接口
  public interface Skill {
   
      /**
       * 唱歌
       * @param name
       */
      void sing(String name);
   
      /**
       * 演出
       * @param name
       */
      void perform(String name);
   
      /**
       * 综艺节目
       * @param name
       */
      void variety(String name);
   
  }
  
  //具体实现类
  public class YingBao implements Skill {
   
      @Override
      public void sing(String name) {
          System.out.println("颖宝唱了一首[" + name + "]");
      }
   
      @Override
      public void perform(String name) {
          System.out.println("颖宝出演了[" + name + "]");
      }
   
      @Override
      public void variety(String name) {
          System.out.println("颖宝上[" + name + "]综艺节目");
      }
   
  }
  
  //代理实现类
  public class YingBaoProxy implements Skill {
   
      //保存被代理人的实例
      private Skill yb;
   
      public YingBaoProxy(Skill skill) {
          this.yb = skill;
      }
   
      //代理人实际是让颖宝去做事情
   
      @Override
      public void sing(String name) {
          yb.sing(name);
      }
   
      @Override
      public void perform(String name) {
          yb.perform(name);
      }
   
      @Override
      public void variety(String name) {
          yb.variety(name);
      }
   
  }
  
  //使用
  public static void main(String[] args) {
          YingBaoProxy ybp = new YingBaoProxy(new YingBao());
          ybp.sing("北京北京");
          ybp.perform("陆贞传奇");
          ybp.variety("天天向上");
      }
  ```

- 动态代理

  动态代理本质是利用反射去调用真正的实现类 在Android中典型场景是retrofit的核心实现 根据定义的API接口方法去包装成实体请求类传递给方法的返回对象

  - InvocationHandler

    该接口中仅定义了一个方法Object：invoke(Object obj,Method method,Object[] args）。在实际使用时，第一个参数obj一般是指代理类，method是被代理的方法，args为该方法的参数数组。这个抽象方法在代理类中动态实现。

  - Proxy

    Static Object newProxyInstance(ClassLoader loader,Class[] interfaces,InvocationHandler h）：返回代理类的一个实例，返回后的代理类可以当作被代理类使用（可使用被代理类的在Subject接口中声明过的方法）。（第一个参数为实体类的classloader，第二个参数是实体抽象接口 第三个参数是实现了InvocationHandler的实现类）

  ```java
  //接口
  public interface Skill {
   
      /**
       * 唱歌
       * @param name
       */
      void sing(String name);
   
      /**
       * 演出
       * @param name
       */
      void perform(String name);
   
      /**
       * 综艺节目
       * @param name
       */
      void variety(String name);
   
  }
  
  //具体实现类
  public class YingBao implements Skill {
   
      @Override
      public void sing(String name) {
          System.out.println("颖宝唱了一首[" + name + "]");
      }
   
      @Override
      public void perform(String name) {
          System.out.println("颖宝出演了[" + name + "]");
      }
   
      @Override
      public void variety(String name) {
          System.out.println("颖宝上[" + name + "]综艺节目");
      }
   
  }
  
  //InvocationHandler
  public class MyInvocationHandler implements InvocationHandler {
   
      private Object obj;//被代理类实例
   
      public MyInvocationHandler(Object obj) {
          this.obj = obj;
      }
   
      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          System.out.println("代理对象要处理的公共逻辑可以统一写到这里");
          Object invoke = method.invoke(obj, args);
          return invoke;
      }
   
  }
  
  //使用
  public static void main(String[] args) throws Exception {
          Skill yb = new YingBao();
          //创建调度对象
          MyInvocationHandler mih = new MyInvocationHandler(yb);
          //动态代理的方式创建对象
          Skill s = (Skill)Proxy.newProxyInstance(yb.getClass().getClassLoader(), new Class[]{Skill.class}, mih);
          s.sing("小幸运");
          s.perform("陆贞传奇");
          s.variety("天天向上");
      }
  ```

#### 3.策略模式

```java
//策略接口
public interface Strategy {
  	void travel();
}

//具体策略执行者
public class WalkStrategy implements Strategy{
    @Override
    public void travel() {
        System.out.println("walk");
    }
}
public class PlaneStrategy implements Strategy{
    @Override
    public void travel() {
        System.out.println("plane");
    }
}
public class SubwayStrategy implements Strategy{
    @Override
    public void travel() {
        System.out.println("subway");
    }
}

//策略包装类
public class TravelContext implements Strategy{
    Strategy strategy;

    public Strategy getStrategy() {
        return strategy;
    }

    public void setStrategy(Strategy strategy) {
        this.strategy = strategy;
    }
    @Override
    public void travel() {
        if (strategy != null) {
            strategy.travel();
        }
    }
}

//使用
TravelContext travelContext=new TravelContext();
travelContext.setStrategy(new PlaneStrategy());
travelContext.travel();
travelContext.setStrategy(new WalkStrategy());
travelContext.travel();
travelContext.setStrategy(new SubwayStrategy());
travelContext.travel();
```

Android中典型应用比如动画的各种插值器 估值器 各种图片库的缓存策略具体实现者 和代理模式非常类似

#### 4.建造者模式

```java
public class UserInfo {
 
    private String name;
    private int age;
    private double height;
    private double weight;
 	//以builder为参数的构造方法
    private UserInfo(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.height = builder.height;
        this.weight = builder.weight;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public int getAge() {
        return age;
    }
 
    public void setAge(int age) {
        this.age = age;
    }
 
    public double getHeight() {
        return height;
    }
 
    public void setHeight(double height) {
        this.height = height;
    }
 
    public double getWeight() {
        return weight;
    }
 
    public void setWeight(double weight) {
        this.weight = weight;
    }
 	//静态内部类可以被外界直接实例化
    static class Builder {
        private String name;
        private int age;
        private double height;
        private double weight;
 
        public Builder name(String name) {
            this.name = name;
            //支持链式调用
            return this;
        }
 
        public Builder age(int age) {
            this.age = age;
            return this;
        }
 
        public Builder height(double height) {
            this.height = height;
            return this;
        }
 
        public Builder weight(double weight) {
            this.weight = weight;
            return this;
        }
 		//关键方法用于创建真正的实体对象
        public UserInfo build() {
            //静态内部类可以访问外部类的私有构造方法
            return new UserInfo(this);
        }
    }
}

//使用
UserInfo.Builder builder=new UserInfo.Builder();
UserInfo person=builder
    .name("张三")
    .age(18)
    .height(178.5)
    .weight(67.4)
    .build();
//获取UserInfo的属性
Log.e("userinfo","name = "+person.getName());
```

- 建造者模式在Android 中的应用比如说alterdialog okhttp 等等

#### 5.单例模式

1. 饿汉式

   ```java
   //优点：写法简单，线程安全。
   //缺点：没有懒加载的效果，如果没有使用过的话会造成内存浪费。
   public class EHanSingleton {
       //在类初始化时，已经自行实例化,所以是线程安全的。
       private static final EHanSingleton single =new EHanSingleton();
    
       public static EHanSingleton getInstance(){
           return single;
       }
   }
   ```

2. 懒汉式

   ```java
   //优点：实现了懒加载的效果，线程安全。
   //缺点：使用synchronized会造成不必要的同步开销，而且大部分时候我们是用不到同步的。
   public class LanHanSingleton {
       private static LanHanSingleton singleton;
    
       public static synchronized LanHanSingleton getSingleton(){
           if (singleton == null){
               singleton =new LanHanSingleton();
           }
           return singleton;
       }
   }
   ```

3. 双重校验锁懒汉

   ```java
   //优点：懒加载，线程安全，效率较高
   public class DoubleCheckSingleton {
       //volatile关键字保证有序性
       private volatile static DoubleCheckSingleton singleton;
    
       public static DoubleCheckSingleton getInstance(){
           if (singleton == null){//第一重空判断保证效率，排除掉不必要的加锁操作
               synchronized (DoubleCheckSingleton.class){
                   if (singleton == null){//第二重空判断保证假设两个线程同时走进了第一重判断后A线程优先拿到锁进入 完毕后B线程进入时不会重新new对象
                       singleton =new DoubleCheckSingleton();
                   }
               }
           }
           return singleton;
       }
   }
   ```

4. 静态内部类

   ```java
   //优点：懒加载，线程安全，推荐使用
   public class A {
    
       public static StaticInnerSigleton getInstance(){
           return Holder.instance;
       }
       //构造方法私有
       private A() {
           
       }
    
       //静态内部类
       private static class Holder{
           private static final A instance =new A();
       }
   }
   ```

#### 6.适配器模式

- 意义

  将一个类的接口变换成客户端所期待的另一种接口，从而使原本因接口不匹配而无法在一起工作的两个类能够在一起工作。Android中典型场景是listview viewpager 的adapter retrofit的calladapter等

  ```java
  //目标角色接口
  public interface Target {
      /**
       * 目标角色的方法
       */
      void request();
  }
  
  //目标角色实现类
  public class ConcreteTarget implements Target {
      @Override
      public void request() {
          log.info("目标角色的具体业务实现。{}", this.getClass().getSimpleName());
      }
  }
  
  //源角色
  public class Adaptee {
      /**
       * 原有的业务逻辑
       */
      public void provider() {
          log.info("原有的业务逻辑，{}", this.getClass().getSimpleName());
      }
  }
  
  //适配器
  public class Adapter extends Adaptee implements Target {
      @Override
      public void request() {
          log.info("我是装饰器：{}", this.getClass().getSimpleName());
          super.provider();
      }
  }
  
  //使用
  public static void main(String[] args) {
          //原有业务逻辑
          Target target = new ConcreteTarget();
          target.request();
          //增加了适配器角色后的业务逻辑
          Target adapter = new Adapter();
          adapter.request();
  }
  ```

  

#### 7.装饰者模式

- 和代理模式很像 但是使用场景不同 区别如下

  1. 扩展一个类的功能而不想使用继承
  2. 动态添加或者隐藏类的功能。

  ```java
  //抽象类
  public abstract class MilkyTea {
   
      public abstract void make();//制作奶茶
   
  }
  
  //实现类
  public class Pearl extends MilkyTea {
   
      @Override
      public void make() {
          System.out.println("加入珍珠......");
      }
   
  }
  
  //抽象装饰者类
  public abstract class MilkyTeaDecorator extends MilkyTea {
   
      private MilkyTea mt;//保存要装饰的对象，使用抽象而不是具体，这样具体的类只有在执行的时候才找到
   
      public MilkyTeaDecorator(MilkyTea mt) {
          this.mt = mt;
      }
   
      @Override
      public void make() {
          if (mt != null) {
              mt.make();
          }
      }
  }
  
  //具体装饰者
  public class JellyDecorator extends MilkyTeaDecorator {
   
      public JellyDecorator(MilkyTea mt) {
          super(mt);
      }
   
      @Override
      public void make() {
          super.make();
          //此处即为加强的功能
          System.out.println("加入果冻......");
      }
  }
  
  //使用
  public static void main(String[] args) {
          System.out.println("制作珍珠奶茶做法一开始......");
          Pearl p = new Pearl();
          JellyDecorator jd = new JellyDecorator(p);      
          jd.make();
  }
  
  ```

  Android中典型应用如ContextWrapper

#### 8.工厂模式

- 简单工厂模式

  ```java
  //抽象产品类
  public abstract class Video {
      public abstract void produce();
  }
  
  //具体产品类
  public class PythonVideo extends Video {
      @Override
      public void produce() {
          System.out.println("Python课程视频");
      }
  }
  ublic class JavaVideo extends Video {
      @Override
      public void produce() {
          System.out.println("Java课程视频");
      }
  }
  
  //工厂类
  public class VideoFactory {
      public Video getVideo(String type){
          if("java".equalsIgnoreCase(type)){
              return new JavaVideo();
          }else if("python".equalsIgnoreCase(type)){
              return new PythonVideo();
          }
          return null;
      }
  }
  
  //使用
  public static void main(String[] args) {
     VideoFactory videoFactory = new VideoFactory();
     Video video = videoFactory.getVideo("java");
     if(video == null){
        return;
     }
     video.produce();
  }
  ```

  *Android中典型使用是创建fragment时的工厂方法

- 工厂方法模式

  ```java
  //在简单工厂的基础上加入抽象工厂类
  public abstract class VideoFactory {
      public abstract Video getVideo();
  }
  //具体工厂类
  public class PythonVideoFactory extends VideoFactory {
      @Override
      public Video getVideo() {
          return new PythonVideo();
      }
  }
  public class JavaVideoFactory extends VideoFactory {
      @Override
      public Video getVideo() {
          return new JavaVideo();
      }
  }
  
  //使用
  public static void main(String[] args) {
        VideoFactory videoFactory = new PythonVideoFactory();
        Video video = videoFactory.getVideo();
        video.produce();
    }
  ```

  应用场景如retrofit的calladapter 就是加入了抽象产品calladapter和抽象工厂calladapter.factory 

  如RxJava2CallAdapterFactory.create()

   

   













