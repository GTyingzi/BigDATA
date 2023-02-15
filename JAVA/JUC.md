&emsp;<a href="#0">JUC</a>  
&emsp;&emsp;<a href="#1">线程创建方式</a>  
&emsp;&emsp;&emsp;<a href="#2">继承Thread类</a>  
&emsp;&emsp;&emsp;<a href="#3">实现Runnable接口</a>  
&emsp;&emsp;&emsp;<a href="#4">ExecutorService、Callable、Future 有返回值线程</a>  
&emsp;&emsp;&emsp;<a href="#5">基于线程池的方式</a>  
&emsp;&emsp;<a href="#6">四种线程池</a>  
&emsp;&emsp;<a href="#7">线程生命周期（状态）</a>  
&emsp;&emsp;<a href="#8">终止线程4种方式</a>  
&emsp;&emsp;<a href="#9">sleep与wait区别</a>  
&emsp;&emsp;<a href="#10">start与run区别</a>  
&emsp;&emsp;<a href="#11">JAVA后台线程</a>  
&emsp;&emsp;<a href="#12">JAVA锁</a>  
&emsp;&emsp;&emsp;<a href="#13">乐观锁</a>  
&emsp;&emsp;&emsp;<a href="#14">悲观锁</a>  
&emsp;&emsp;&emsp;<a href="#15">自旋锁</a>  
&emsp;&emsp;&emsp;<a href="#16">Synchronized同步锁</a>  
&emsp;&emsp;&emsp;<a href="#17">ReentrantLock</a>  
&emsp;&emsp;&emsp;<a href="#18">Semaphore信号量</a>  
&emsp;&emsp;&emsp;<a href="#19">AtomicInteger</a>  
&emsp;&emsp;&emsp;<a href="#20">可重入锁</a>  
&emsp;&emsp;&emsp;<a href="#21">公平锁与非公平锁</a>  
&emsp;&emsp;&emsp;<a href="#22">读写锁</a>  
&emsp;&emsp;&emsp;<a href="#23">独占锁与共享锁</a>  
&emsp;&emsp;&emsp;<a href="#24">同步锁与死锁</a>  
&emsp;&emsp;&emsp;<a href="#25">锁的状态</a>  
&emsp;&emsp;&emsp;<a href="#26">分段锁</a>  
&emsp;&emsp;&emsp;<a href="#27">锁优化</a>  
&emsp;&emsp;<a href="#28">线程基本方法</a>  
&emsp;&emsp;<a href="#29">线程池原理</a>  
&emsp;&emsp;&emsp;<a href="#30">线程池的组成</a>  
&emsp;&emsp;&emsp;<a href="#31">决绝策略</a>  
&emsp;&emsp;&emsp;<a href="#32">Java线程池工作过程</a>  
&emsp;&emsp;<a href="#33">JAVA阻塞队列原理</a>  
&emsp;&emsp;&emsp;<a href="#34">阻塞队列的主要方法</a>  
&emsp;&emsp;&emsp;<a href="#35">Java中的阻塞队列</a>  
&emsp;&emsp;<a href="#36">CountDownLatch、CyclicBarrier、Semaphore 的用法</a>  
&emsp;&emsp;<a href="#37">volatile关键字的作用</a>  
&emsp;&emsp;<a href="#38">如何在两个线程间共享数据</a>  
&emsp;&emsp;<a href="#39">ThreadLocal</a>  
&emsp;&emsp;<a href="#40">Synchronized与ReentrantLock的区别</a>  
&emsp;&emsp;<a href="#41">ConcurrentHashMap并发</a>  
&emsp;&emsp;<a href="#42">Java中用到的线程调度</a>  
&emsp;&emsp;&emsp;<a href="#43">抢占式调度</a>  
&emsp;&emsp;&emsp;<a href="#44">协同式调度</a>  
&emsp;&emsp;<a href="#45">CAS</a>  
&emsp;&emsp;<a href="#46">AQS</a>  
## <a name="0">JUC


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220523203029556.png)

### <a name="1">线程创建方式


#### <a name="2">继承Thread类


Thread 类本质上是实现了 Runnable 接口的一个实例，代表一个线程的实例。启动线程的唯一方法就是通过 Thread 类的 start()实例方法

**start()方法是一个 native 方法**，它将启动一个新线程，并执行 run()方法

```java
public class MyThread extends Thread { 
 public void run() { 
 System.out.println("MyThread.run()"); 
 } 
} 
MyThread myThread1 = new MyThread(); 
myThread1.start();
```

#### <a name="3">实现Runnable接口


当自己的类已经extends 另一个类，就无法直接 extends Thread，此时，可以实现一个 Runnable 接口

```java
public class MyThread extends OtherClass implements Runnable { 
     public void run() { 
     	System.out.println("MyThread.run()"); 
     } 
}
//启动 MyThread，需要首先实例化一个 Thread，并传入自己的 MyThread 实例：
MyThread target = new MyThread(); 
Thread thread = new Thread(target); 
thread.start(); 
//事实上，当传入一个 Runnable target 参数给 Thread 后，Thread 的 run()方法就会调用target.run()
public void run() { 
     if (target != null) { 
     	target.run(); 
     } 
}
```

#### <a name="4">ExecutorService、Callable、Future 有返回值线程


有返回值的任务必须实现Callable接口。执行Callable任务后，可获取一个Future对象，在该对象上调用get就可以获取到Callable任务返回的Object了，再结合线程池ExecutorService就可以实现由返回结果的多线程

```java
//创建一个线程池
ExecutorService pool = Executors.newFixedThreadPool(taskSize);
// 创建多个有返回值的任务
List<Future> list = new ArrayList<Future>(); 
for (int i = 0; i < taskSize; i++) { 
    Callable c = new MyCallable(i + " "); 
    // 执行任务并获取 Future 对象
    Future f = pool.submit(c); 
    list.add(f); 
} 
// 关闭线程池
pool.shutdown(); 
// 获取所有并发任务的运行结果
for (Future f : list) { 
    // 从 Future 对象上获取任务的返回值，并输出到控制台
    System.out.println("res：" + f.get().toString()); 
} 
```

#### <a name="5">基于线程池的方式


线程和数据库连接都是非常宝贵的资源，每次需要的时候创建，不需要的时候销毁是非常浪费资源，故可以使用缓存策略，也就是线程池

```java
        // 创建线程池
        ExecutorService threadPool = Executors.newFixedThreadPool(10);
        while (true) {
            threadPool.execute(new Runnable() { // 提交多个线程任务，并执行
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + " is running ..");
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
```

### <a name="6">四种线程池


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220523204822151.png)

Java 里面线程池的顶级接口是 Executor，但是严格意义上讲 Executor 并不是一个线程池，而只是一个执行线程的工具，真正的线程池接口是 ExecutorService

- <u>**newCachedThreadPool**</u>：创建一个可**根据需要创建新线程**的线程池，在以前构造的线程可用时将重用它们。对于执行很多短期异步任务的程序而言，这些线程池通常可提高程序性能。**调用 execute 将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程**。因此，长时间保持空闲的线程池不会使用任何资源

- **<u>newFixedThreadPool</u>**：**创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程**。在任意点，大多数Threads线程会处于处理任务的活动状态。在所有线程处于活动状态时提交附加任务，在有可用线程之前，附加任务将在队列中等待。如果在关闭前的执行期间由于失败而导致任何线程终止，那么一个新线程将代替它执行后续的任务（如果需要）。在某个线程被显式地关闭之前，池中的线程将一直存在

- **<u>newSingleThreadExecutor</u>**：返回一个只有一个线程的线程池，这个线程池在线程死后（或发生异常时）重新启动一个线程来替代原来的线程继续执行下去

- **<u>newScheduledThreadPool</u>**：创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行

  ```java
  ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(3);
  scheduledThreadPool.schedule(newRunnable() {
              @Override
              public void run () {
                  System.out.println("延迟三秒");
              }
          },3, TimeUnit.SECONDS);
  scheduledThreadPool.scheduleAtFixedRate(newRunnable() {
              @Override
              public void run () {
                  System.out.println("延迟 1 秒后每三秒执行一次");
              }
          },1, 3, TimeUnit.SECONDS);
  ```

### <a name="7">线程生命周期（状态）


五种状态：新建（New）、就绪（Runnable）、运行（Running）、阻塞（Blocked）、死亡（Dead）

当线程启动以后，CPU会在多条线程之间切换，线程状态也会多次在运行、阻塞之间切换

- **<u>新建</u>**：**new 关键字创建了一个线程之后**，该线程就处于新建状态，此时仅由 JVM 为其分配 内存，并初始化其成员变量的值
- **<u>就绪状态</u>**：线程对象**调用start()方法之后**，该线程就处于就绪状态。JVM会为其创建方法调用栈和程序计数器
- **<u>运行状态</u>**：处于**就绪状态的线程获得了CPU**，就开始**执行run()方法**的线程执行体
- **<u>阻塞状态</u>**：指线程因为某种原因放弃了 cpu 使用权，也即让出了 cpu timeslice，暂时停止运行
  - 等待阻塞：运行的线程执行wait()方法，JVM会把该线程放入waitting queue中
  - 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，JVM会把该线程放入锁池中
  - 其他阻塞：运行的线程执行 Thread.sleep(long ms)或 t.join()方法，或者发出了 I/O 请求时，JVM 会把该线程置为阻塞状态
- **<u>线程死亡</u>**：
  - 正常结束：run()、call()方法执行完成，线程正常结束
  - 异常结束：线程抛出一个未捕获的 Exception 或 Error
  - 调用stop：直接调用该线程的 stop()方法来结束该线程，该方法通常容易导致死锁，不推进使用

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220523211932064.png)

### <a name="8">终止线程4种方式


- **<u>正常运行结束</u>**

- **<u>使用退出标志退出线程</u>**：一般run()方法执行完，线程就会正常结束，然而，常常有线程是伺服线程，它们**需要长时间的运行，只有在外部某些条件满足的情况下，才会关闭这些线程**。使用一个变量来控制循环，定义一个退出标志exit，当exit为true时退出循环。此处使用了一个java关键字 volatile，这个关键字的目的是使 exit 同步，同一时刻只能有一个线程来修改exit值

  ```java
  public class ThreadSafe extends Thread {
       public volatile boolean exit = false; 
           public void run() { 
               while (!exit){
                   //do something
               }
       } 
  }
  ```

- **<u>Interrupt()方法结束线程</u>**：使用interrupt()方法来中断线程有两种情况

  - **线程处于阻塞状态**：使用了sleep，同步锁的wait，socket中的receiver，accept等方法时，线程将处于阻塞状态。当调用线程的 interrupt()方法时，会抛出 InterruptException 异常。阻塞中的那个方法抛出这个异常，通过代码捕获该异常，然后 break 跳出循环状态，从而让我们有机会结束这个线程的执行

  - **线程未处于阻塞状态**：使用 isInterrupted()判断线程的中断标志来退出循环，当使用nterrupt()方法时，中断标志就会置 true，和使用自定义的标志来控制循环是一样的道理

    ```java
    public class ThreadSafe extends Thread {
         public void run() { 
             while (!isInterrupted()){ //非阻塞过程中通过判断中断标志来退出
                 try{
                     Thread.sleep(5*1000);//阻塞过程捕获中断异常来退出
                 }catch(InterruptedException e){
                     e.printStackTrace();
                     break;//捕获到异常之后，执行 break 跳出循环
                 }
             }
         } 
    }
    ```

- **<u>stop方法</u>**：线程不安全，调用stop()之后，创建子线程的线程会抛出ThreadDeatherror 的错误，并且会释放子线程所持有的所有锁，那么被保护的数据将可能呈现不一致性，其他线程在使用这些被破坏的数据时，会导致一些很奇怪的应用程序错误。

### <a name="9">sleep与wait区别


- sleep()方法属于Thread类，会导致程序暂停执行指定时间，让出CPU给其他线程，监控状态依然保持，线程不会释放对象锁
- wait()方法属于Object类，线程放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象调用notify()方法后本线程才进入对象锁定池准备获取对象锁进入运行状态

### <a name="10">start与run区别


- start()方法来启动线程，真正实现了多线程运行，这时无需等待run方法体代码执行完毕，可以继续执行下面的代码；通过调用Thread类的start()方法来启动一个线程，这时此线程处于**就绪状态**，没有运行
- run()方法称为线程体，包含了要执行的这个线程的内容，线程就进入了**运行状态**，开始运行run函数中的代码。run方法运行结束，此线程终止，CPU再调度其他线程

### <a name="11">JAVA后台线程


也称守护（服务）线程，为用户线程提供公共服务，在没有用户线程可服务时会自动离开

- **优先级**：守护线程优先级低，用于为系统中的其他对象和线程提供服务
- **设置**：setDaemon(true)来设置线程为“守护线程”；将一个用户线程设置为守护线程 的方式是在 线程对象创建 之前 用线程对象的 setDaemon 方法
- 在 Daemon 线程中产生的新线程也是 Daemon 的
- 线程则是 JVM 级别的，以 Tomcat 为例，如果你在 Web 应用中启动一个线程，这个线程的 生命周期并不会和 Web 应用程序保持同步。也就是说，即使你停止了 Web 应用，这个线程 依旧是活跃的
- **example**：垃圾回收线程就是一个经典的守护线程，当我们的程序中不再有任何运行的Thread，程序就不会再产生垃圾，垃圾回收器也就无事可做，所以当垃圾回收线程是 JVM 上仅剩的线程时，垃圾回收线程会自动离开。它始终在低级别的状态中运行，用于实时监控和管理系统中的可回收资源
- **生命周期**：守护（Daemon）进程是运行在后台的一种特殊进程，它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。守护线程不依赖于终端，但依赖于系统，于系统同生共死。当JVM 中所有的线程都是守护线程的时候，JVM 就可以退出了，如果还有一个或以上的非守护线程则 JVM 不会退出

### <a name="12">JAVA锁


#### <a name="13">乐观锁


```
乐观思想，认为读多写少，遇到并发写的可能性低，每次去拿数据时都认为别人不会修改，故不会上锁

在更新时会判断在此期间别人有没有去更新这个数据，采取在写时先读出当前版本号，然后加锁操作（比较上一次版本，一样则更新），若失败则要重复 读-比较-写 的操作

JAVA中的乐观锁基本是通过 **CAS** 操作实现，CAS 为一种更新的原子操作，比较当前值跟传入值是否一样，一样则更新，否则失败
```

#### <a name="14">悲观锁


```
悲观思想，认为写多，遇到并发写的可能性高，每次拿数据时都会认为别人会修改，故每次在读写数据时会加上锁

JAVA中的悲观锁就是 Synchronized，AQS框架下的锁是先尝试 CAS 乐观锁去获取锁，获取不到才会转为 悲观锁，如RetreenLock
```

#### <a name="15">自旋锁


```
若持有锁的线程在很短时间内释放锁资源，那些等待竞争锁的线程就不需要做内核态和用户态之间的切换进入阻塞挂起的状态，它们只需要等一等（自旋），等持有锁的线程释放锁后即可立即获取锁，这样就避免用户线程和内核的切换消耗

线程自旋需要消耗CPU，若一直获取不到锁，线程也不能一直占用CPU自旋做无用功，需设定一个自旋等待的最大时间。若持有锁的先出执行的时间超过最大时间仍没有释放锁，那么争用线程会停止自旋进入阻塞状态

自旋锁的优缺点
- 优：尽可能减少线程的阻塞，对于锁的竞争不激烈，且占用锁时间非常短的代码来说性能优异。自旋的消耗会小于线程阻塞挂起再唤醒的消耗，这些操作会导致线程发生两次上下文切换
- 缺：若锁的竞争激烈，或者持有锁的线程需要长时间占用锁执行同步代码块，会导致自旋锁一直占用 CPU 做无用功，导致线程自旋的消耗大于线程阻塞挂起的消耗，造成 CPU 浪费，此时我们需要关闭自旋锁

自旋锁时间阈值（1.6引入适应性自旋锁）
JVM对于自旋周期的选择，jdk1.5之前是写死的，在1.6引入了适应性自旋锁
由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定，基本认为一个线程上下文切换的时间为最佳时间
- JVM还针对 CPU 的负荷情况做了较多的优化，若平均负载小于 CPUs 则一直自旋，若有超过 CPUs/2 个线程正在自旋，则后来线程直接阻塞
- 若正在自旋的线程发现 Owner 发生了变化则延迟自旋时间(自旋计数)或进入阻塞
- 若CPU处于节电模式则停止自旋
- 自旋时间的最坏情况是 CPU 的存储延迟（CPU A存储一个数据，CPU B得知这个数据的时间差）
- 自旋会适当放弃线程优先级之间的差异

自旋锁的开启
JDK1.6 中-XX:+UseSpinning 开启；
-XX:PreBlockSpin=10 为自旋次数；
JDK1.7 后，去掉此参数，由 jvm 控制
```

#### <a name="16">Synchronized同步锁


```
可以把任意一个非NULL的对象当做锁，属于独占式的悲观锁，同时属于可重入锁

Synchronized作用范围
	-作用于方法时，锁住的是对象的实例(this)
	-作用于静态方法，锁住的是Class实例，又因为Class的相关数据存储在永久代(jdk1.8后是metaspace)，永久代全局共享，静态方法锁相当于类的一个全局锁，会锁住所有调用该方法的线程
	-synchronized作用于一个对象实例时，锁住的是所有以该对象为锁的代码块。它有多个队列，当多个线程一起访问某个对象监视器的时候，对象监视器会将这些线程存储在不同的容器中
	
Synchronized核心组件
Wait Set：那些调用wait方法被阻塞的线程将被放置在这里
Contention List：竞争队列，所有请求锁的线程首先被放在这个竞争队列中
Entry List：Contention List中那些有资格成功候选资源的线程被移动到 Entry List 中
OnDeck：任意时刻，最多只有一个线程正在竞争锁资源，该线程被称为 OnDeck
Owner：当前已经获取到所有资源的线程被称为 Owner
!Owner：当前释放释放锁的线程
```

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220529195415984.png" style="zoom: 67%;" />

1. JVM每次从队列的尾部取出一个数据用于锁竞争候选者（OnDeck），但是并发情况下，ContentionList会被大量的并发线程进行CAS访问，为了降低尾部元素的竞争，JVM会将部分线程移动到 EntryList 中作为候选竞争线程
2. Owner线程会在 unlock时，将ContentionList 中的部分线程迁移到 EntryList 中，并指定EntryList 中的某个线程为 OnDeck线程
3. Owner线程并不直接把锁传递给 OnDeck 线程，而是把锁竞争的权利交给 OnDeck，OnDeck需要重新竞争锁。这样虽牺牲了一些公平性，但能极大的提升系统的吞吐量，在JVM中，把这种选择行为称之为“竞争切换”
4. OnDeck 线程获取到锁资源后会变为 Owner 线程，而没有得到锁资源的仍然停留在 EntryList 中。如果 Owner 线程被 wait 方法阻塞，则转移到 WaitSet 队列中，直到某个时刻通过 notify 或者 notifyAll 唤醒，会重新进去 EntryList 中
5. 处于 ContentionList、EntryList、WaitSet 中的线程都处于阻塞状态，该阻塞是由操作系统 来完成的（Linux 内核下采用 pthread_mutex_lock 内核函数实现的）
6. Synchronized 是非公平锁。 Synchronized 在线程进入 ContentionList 时，等待的线程会先尝试自旋获取锁，如果获取不到就进入 ContentionList，这明显对于已经进入队列的线程是不公平的，还有一个不公平的事情就是自旋获取锁的线程还可能直接抢占 OnDeck 线程的锁资源
7. 每个对象都有个 monitor 对象，加锁就是在竞争 monitor 对象，代码块加锁是在前后分别加 上 monitorenter 和 monitorexit 指令来实现的，方法加锁是通过一个标记位来判断的
8. synchronized 是一个重量级操作，需要调用操作系统相关接口，性能是低效的，有可能给线程加锁消耗的时间比有用操作消耗的时间更多
9. Java1.6，synchronized 进行了很多的优化，有适应自旋、锁消除、锁粗化、轻量级锁及偏向锁等，效率有了本质上的提高。在之后推出的 Java1.7 与 1.8 中，均对该关键字的实现机理做了优化。引入了偏向锁和轻量级锁。都是在对象头中有标记位，不需要经过操作系统加锁
10. 锁可以从偏向锁升级到轻量级锁，再升级到重量级锁。这种升级过程叫做锁膨胀
11. JDK 1.6 中默认是开启偏向锁和轻量级锁，可以通过-XX:-UseBiasedLocking 来禁用偏向锁

#### <a name="17">ReentrantLock


ReentrantLock 继承接口 Lock 并实现了接口中定义的方法，他是一种可重入锁，除了能完成 synchronized 所能完成的所有工作外，还提供了诸如可响应中断锁、可轮询锁请求、定时锁等避免多线程死锁的方法

**Lock接口的主要方法**

- **lock**()：执行此方法时，若锁处于空闲状态，当前线程将获取到锁；若锁已经被其他线程持有，则禁用当前线程，直至当前线程获取锁
- **tryLock**()：如果锁可用，则获取锁，并立即返回true，否则返回false。该方法和lock()的区别在于，tryLock()只是试图获取锁，若当前锁不可用，不会导致当前线程禁用，当前线程仍然继续往下执行代码
- **tryLock**(long timeout，TimeUnit unit)：若锁在给定等待时间内没有被另一个线程持有，则获取该锁
- **unlock**()：执行此方法时，当前线程将释放锁持有的锁。锁只能由持有者释放，若线程并不持有锁，执行此方法可能导致异常
- **Condition newConditition**()：条件对象，获取等待通知组件。该组件和当前锁绑定，当前线程只有获取了锁，才能调用该组件的await()方法，调用后当前线程将释放锁
- **getHoldCount**()：查询当前线程保持此锁的次数，也就是此线程执行lock()的次数
- **getQueueLength**()：返回正等待获取此锁的线程估计数。比如启动10个线程，1个线程获得锁，此时返回的是9
- **getWaitQueueLength**(Condition condition)：返回等待与此锁相关的给定调节的线程估计数。比如10个线程，同用一个 condition 对象，并且此时这10个线程都执行了 condition 对象的await 方法，那么此时执行此方法返回10
- **hasWaiters**(Condition condition)：查询是否有线程等待与此锁有关的给定条件，指定 condition 对象，有多少线程执行了condition.await方法
- **hasQueuedThread**(Thread thread)：查询给定线程释放等待获取此锁
- **hasQueuedThreads**()：是否有线程等待此锁
- **isFair**()：该锁释放为公平锁
- **isHeldByCurrentThread**()：当前线程是否保持锁定，线程执行 lock() 的前后分别是 false 和true
- **isLock**()：此锁释是否有任意线程占用
- **lockInterruptibly**()：若当前线程未被中断，获取锁

**ReentrantLock与Synchronized的区别**

- ReentrantLock通过lock()与unlock()来进行加锁、解锁操作，与Synchronized会被JVM自动解锁机制不同，ReentrantLock加锁后需要手动解锁。为了避免程序出现异常而无法正常解锁的情况，使用ReentrantLock必须在finally控制块中进行解锁操作
- ReentrantLock相比synchronized的优势是可中断、公平锁、多个锁

**Condition类和Object类锁方法区别**

- Condition类的 awiat 方法与Object类的 wait 方法等效
- Condition类的 signal方法与Object类的 notify方法等效
- Condition类的 signalAll方法与Object类的 notifyAll方法等效
- ReentrantLock类可以唤醒指定条件的线程，而Object的唤醒是随机的

#### <a name="18">Semaphore信号量


一种基于计数的信号量。可以设定一个阈值，多个线程竞争获取许可信号，做完自己的申请后归还，超过阈值后，线程申请许可信号将会被阻塞。Semaphore可用来构建一些对象池、资源池之类的

**Semaphore与ReentrantLock的区别**

- Semaphore基本能完成ReentrantLock的所有工作，使用方法也与之类似，通过acquire()与release()方法来获得和释放临界资源。经实测，Semphore.acquire()方法默认为可响应中断锁，与ReentrantLock.lockInterruptibly()作用效果一致，也就是说在等待临界资源的过程中可以被Thread.interrupt()方法中断
- Semaphore也实现了**可轮询锁的请求与定时锁的功能**，除了方法名 tryAcquire 与 tryLock 不同，其使用方法与 ReentrantLock 几乎一致。Semaphore 也提供了公平与非公平锁的机制，也可在构造函数中进行设定
- Semaphore的锁释放操作也由手动进行，因此与ReentrantLock一样，为避免线程因抛出异常而无法正常释放锁的情况发生，释放锁的操作必须在finnally代码块中完成

#### <a name="19">AtomicInteger


一个提供原子操作的 Integer 的类，常见的还有：AtomicBoolean、AtomicInteger、AtomicLong、AtomicReference等，它们的实现原理相同，可通过 AtomicReference<V>将一个对象的所有操作转化成原子操作

在多线程程序中，诸如++i或i++等运算不具有原子性，是不安全的线程操作之一。通常我们会使用 Synchronized将该操作变成一个原子操作，但JVM为此类操作特意提供了一些同步类，使得使用更方便，且使程序运行效率更高

#### <a name="20">可重入锁


也叫做递归锁，指的是同一线程外层函数获得锁之后，内存递归函数仍然有获取该锁的代码，但不受影响

在JAVA环境下：ReentrantLock、Synchronized 都是可重入锁

#### <a name="21">公平锁与非公平锁


**公平锁**：加锁前检测是否有排队等待的线程，优先排队等待的线程，先来先得

**非公平锁**：加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待

- 非公平锁性能比公平锁高5-10倍，因为公平锁需要在多核情况下维护一个队列
- JAVA中的 Synchronized 是非公平锁，ReentrantLock 默认的lock()方法采用非公平锁

#### <a name="22">读写锁


**读锁**：代码只读数据，可以很多人同时读，但不能同时写

**写锁**：代码修改数据，只能一个人在写，且不能同时读写

Java中读写锁有个接口 java.util.concurrent.locks.ReadWriteLock，具体的实现例子：ReentrantReadWriteLock

#### <a name="23">独占锁与共享锁


Java并发包提供的加锁模式分为 独占锁 与 共享锁

**独占锁**：独占锁是一种悲观保守的加锁策略，避免了读读冲突，若某个只读线程获取锁，则其他读线程只能等待，这种情况下就限制了不必要的并发性，读操作不会影响数据一致性。独占锁模式下，每次只能有一个线程持有锁，ReentrantLock就是以独占方式实现的互斥锁

**共享锁**：共享锁是一种乐观锁，放宽了加锁策略，允许多个执行读操作的线程同时访问共享资源。允许多个线程同时获取锁，并发访问共享资源，如：ReadWriteLock

- AQS的内部类 Node 定义了两个常量 SHARED 和 EXCLUSIVE，它们分别标识 AQS 队列中等待线程的锁获取模式
- JAVA的并发包提供了 ReadWriteLock，读-写锁。允许一个资源可以被多个读操作访问，或者被一个写操作访问，两者不能同时进行

#### <a name="24">同步锁与死锁


**同步锁**：多个线程访问同一个数据时，很容易出现问题，为了避免这种出现，要保证线程同步互斥，并发执行多个线程，在同一个时间内只允许一个线程访问共享数据，Java中可以使用synchronized关键字来取得一个对象的同步锁

**死锁**：多个线程同时被阻塞，它们中的一个或全部都在等待某个资源被释放

#### <a name="25">锁的状态


锁的状态总共有四种：无锁状态、重量级锁、轻量级锁、偏向锁

锁升级：随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级到重量级锁（锁的升级是单向）

**重量级锁**（Mutex Lock）

```
Synchronized 是通过对象内部的一个叫做监视器锁（monitor）来实现的。但是监视器锁本质是依赖于底层操作系统的Mutex Lock实现。操作系统实现线程之间的切换这就需要从用户态转换到核心态，成本很高，切换需要较长时间，这就是Synchronized效率低的原因。
我们称依赖于操作系统 Mutex Lock所实现的锁为“重量级锁”，JDK1.6以后，为了减少获得锁和释放锁所带来的性能消耗，提高性能，引入了 轻量级锁、偏向锁
```

**轻量级锁**

```
轻量级锁是相对于操作系统互斥量来实现的传统锁而言。轻量锁并不是用来代替重量级锁，它的本意是在没有多线程竞争的前提下，减少传统的重量级锁使用产生的性能消耗。
轻量级锁所适应的场景为线程交替执行同步块的情况，如果存在同一时间访问同一锁的情况，就会导致轻量级锁膨胀为重量级锁
```

**偏向锁**

```
偏向锁的目的是在某个线程获得锁之后，消除这个线程锁重入（CAS）的开销，看起来让这个线程得到偏护。
引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁的执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令
偏向锁只需要在置换ThreadID时依赖一次CAS原子指令（一旦出现多线程竞争的情况就必须撤销偏向锁，所以偏向锁的撤销操作性能的损耗必须小于节省下来的 CAS原子指令的性能消耗）
轻量级锁是为了在线程交替执行同步块时提高性能；偏向锁是在只有一个线程执行同步块时进一步提供性能
```

#### <a name="26">分段锁


并非一种实际的锁，而是一种思想，ConcurrentHashMap是学习分段锁的最好实践

#### <a name="27">锁优化


- **减少锁持有时间**：只用在有线程安全要求的程序上加锁
- **减小锁粒度**：将大对象拆成小对象，大大增加并行度，降低锁竞争。最典型的减小锁粒度的案例是ConcurrentHashMap
- **锁分离**：最常见的锁分离是读写锁ReadWriteLock，根据功能进行分离成读锁、写锁，读读不互斥、读写互斥、写写互斥，保证了线程安全，提高了性能。读写分离思想可延伸，只要操作互不影响，锁就可以分离，比如LinkedBlockingQueue从头部取出，从尾部放数据
- **锁粗化**：通常情况下，为了保证多线程间的有效并发，会要求每个线程持有锁的时间尽量短，在使用完公共资源后，应该立即释放锁。但是凡事都有一个度。若对同一个锁不停的进行请求、同步、释放，其本身也会消耗系统宝贵的资源，反而不利于性能优化
- **锁消除**：锁消除是在编译器级别的事情，在即使编译器时，若发现不可能被共享的对象，则可以消除这些对象的锁操作，多数是因为程序员编码不规范引起

### <a name="28">线程基本方法


线程相关的基本方法有：wait、notify、notifyAll、sleep、join、yield等

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530101457837.png)

- **线程等待**（wait）：调用该方法的线程进入 WAITING 状态，只有等待另外线程的通知或中断才会返回，需要注意的是调用 wait()方法后会释放对象的锁。因此，wait方法一般用在同步方法或同步代码块中
- **线程睡眠**(sleep)：sleep导致当前线程休眠，与wait方法不同的是 sleep不会释放当前占有的锁，sleep(long)会导致线程进入TIMED-WATING 状态
- **线程让步**(yield)：会使当前线程让出CPU执行时间片，与其他线程一起重新竞争PCU时间片。一般情况下，优先级高的线程有更大的可能性成功竞争到CPU时间片，但这又不是绝对的，有的操作系统对线程优先级并不敏感
- **线程中断**(interrupt)：中断一个线程，其本意是给这个线程一个通知信号，会影响这个线程内部的一个中断标识位。这个线程本身并不会因此改变状态（如阻塞、终止等）
  - 调用 interrupt 方法并不会中断一个正在运行的线程。也就是说处于 Running 状态的线程并不会因为被中断而被终止，仅仅改变了内部维护的中断标识位
  - 若调用 sleep() 而使线程处于 TIMED-WATING 状态，这时调用 interrupt()方法，会抛出 InterruptedException，从而使线程提前结束 TIMED-WATING 状态
  - 许多声明抛出 InterruptedException 的方法（如Thread.sleep(long mills方法），抛出异常前，会清除中断标识位，抛出异常后，调用 isInterrupted()方法将会返回 false
  - 中断状态是线程固有的一个标识位，可以通过此标识位安全的终止线程
- **Join**：等待其他线程终止，在当前线程中调用一个线程的join()方法，则当前线程转为阻塞状态，回到另一个线程结束，当前线程再由阻塞状态变为就绪状态，等待CPU的宠幸
- **线程唤醒**(notify)：**唤醒在此对象监视器上等待的单个线程**，若所有线程都在此对象上等待，则会唤醒其中一个线程，选择是任意的，并在对实现做出决定时发生，线程通过调用其中一个wait()方法，在对象监视器上等待，**直到当前的线程放弃此对象上的锁定，才能继续执行被唤醒的线程**，被唤醒的线程将以常规方式与该对象上主动同步的其他所有线程进行竞争，类似的方法还有notifyAll()，唤醒在此监视器上等待的所有线程
- 其他方法
  - sleep()：强迫一个线程睡眠N毫秒
  - isAlive()：判断一个线程是否存活
  - activeCount()：程序中活跃的线程数
  - enumerate()：枚举程序中的线程
  - currentTread()：得到党性线程
  - isDaemon()：是否为守护线程
  - setDaemon()：设置一个线程为守护线程
  - setName()：为线程设置一个名词
  - wait()：强迫一个线程等待
  - notify()：通知一个线程继续运行
  - setPriority()：设置一个线程的优先级
  - getPriority()：得到一个线程的优先级

### <a name="29">线程池原理


线程池做的工作主要是控制运行的线程数量，处理过程中将任务放入队列，然后在线程创建后启动这些任务，如果线程数量超过了最大数理（超出数量的线程排队等候），等其他线程执行完毕，再从队列中取出任务来执行

**原理**：每一个Thread的类都有一个start方法，当调用start启动线程时Java虚拟机会调用该类的run方法，那么该类的run方法中继续调用Runnable对象的run方法。我们可以继承重写Thread类，在其start方法中添加不断循环调用传递过来的Runnable对象。循环方法中不断获取Runnable是用Queue实现，在获取下一个Runnable之前可以是阻塞的

主要特点为：**线程复用**、**控制最大并发数**、**管理线程**

#### <a name="30">线程池的组成


一般的线程池主要分为以下4个组成部分：

- **线程池管理器**：用于创建并管理线程池
- **工作线程**：线程池中的先出
- **任务接口**：每个任务必须实现的接口，用于工作线程调度其运行
- **任务队列**：用于存放待处理的任务，提供一种缓冲机制

Java中的线程池是通过Executor框架实现，该框架中用到了Executor、Executors、ExecutorService、ThreadPoolExecutor、Callable、Future、FutureTask这几个类

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530110203408.png)

```java
public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize, long keepAliveTime,TimeUnit unit, BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
}
```

- corePoolSize：指定了线程池中的线程数量
- maximumPoolSize：指定了线程池中的最大线程数量
- keepAliveTime：当前线程池数量超过corePoolSize时，多余的空闲线程的存活时间，即多少时间内会被销毁
- unit：keepAliveTime的单位
- workQueue：任务队列，被提交但尚未被执行的任务
- threadFactory：线程工厂，用于创建线程，一般用默认的即可
- handler：拒绝策略，当前任务太多来不及处理，如何拒绝任务

#### <a name="31">决绝策略


线程池中的线程已经用完，无法继续为新任务服务。同时，等待队列也已经排满了，再也塞不下新任务了，这时我们就需要拒绝策略机制合理的处理这个问题，JDK内置的拒绝策略如下：

- AbortPolicy：直接抛出异常，组织系统正常运行
- CallerRunsPolicy：只要线程池未关闭，该策略直接在调用者线程中，运行当前被丢弃的任务。显然这样做不会真的丢弃任务，但会导致任务提交线程的性能急剧下降
- DiscardOldsetPolicy：丢弃最老的一个请求，也就是即将被执行的一个任务，并尝试再次提交当前任务
- DiscardPolicy：该策略默默地丢弃无法处理的任务，不予任何处理，如果允许任务丢弃，这是最好的一种方案

以上内置拒绝策略均实现了 RejectedExecutionHadler 接口，若以上策略仍无法满足实际需求，可以自己扩展 RejectedExecutionHandler 接口

#### <a name="32">Java线程池工作过程


- 当线程池刚创建时，里面没有一个线程。任务队列是作为参数传进来的，不过就算队列里面有任务，线程池也不会马上执行它们
- 当调用 execute() 方法添加一个任务时，线程池会做如下判断：
  - 如果正在运行的线程数量 < corePoolSize，那么马上创建线程运行这个任务
  - 如果正在运行的线程数量 >= corePoolSize，那么将这个任务放入队列
  - 如果这时候队列满了，而且正在运行的线程数量 < maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务
  - 如果队列满了，而且正在运行的线程数量 >= maximumPoolSize，那么线程池会抛出异常 RejectExecutionException
- 当一个线程完成任务时，它会从队列中取下一个任务来执行
- 当一个线程无事可做，超过一定的时间（keepAliveTime）时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530111933429.png)

### <a name="33">JAVA阻塞队列原理


在阻塞队列中，线程阻塞有这样的两种情况：

- 当队列中没有数据的情况下，消费者端的所有线程都会别自动阻塞(挂起)，直到有数据放入队列
- 当队列中填满数据的情况下，生产者端的所有线程都会自动阻塞(挂起)，直到队列中有空的位置，线程将自动唤醒

#### <a name="34">阻塞队列的主要方法


![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530112339228.png)

- 抛出异常：抛出一个异常
- 特殊值：返回一个特殊值（null或false，视情况而定）
- 阻塞：在成功操作之前，一直阻塞线程
- 超时：放弃前只在最大的时间内阻塞

**插入操作**

```
public abstract boolean add(E paramE)
将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回true，如果当前没有可用，则抛出 IllegalStateException，如果该元素是NULL，则会抛出NullPointException异常

public abstract boolean offer(E paramE)
将指定元素插入此队列中（如果立即可行且不会违反容量限制），成功时返回true，如果没有可用空间则返回flase

public abstract void put(E paramE) throws InterruptedException
将指定元素插入此队列中，将等待可用的空间（如果有必要）

offer(E o, long timeout, TimeUnit unit)
可以设定等待的时间，如果在指定的时间内，还不能往队列中加入BlockingQueue，则返回失败
```

**获取数据操作**

```
poll(time)
取走 BlockingQueue 里排在首位的对象，若不能立即取出，则可以等 time 参数规定的时间，取不到时返回null

poll(long timeout, TimeUnit unit)
从 BlockinQueue 取出一个队首的对象，如果在指定时间内，队列一旦有数据可取，则立即返回队列中的数据。否则直到时间超时还没有数据可取，返回失败

take()
取走 BlockingQueue 里排在首位的对象，若BlockingQueue 为空，阻断进入等待状态直到 BlockingQueue 有新的数据被加入

drainTo()
一次性从 BlockingQueue 获取所有可用的数据对象（还可以指定获取数据的个数），通过该方法，可以提升获取数据效率；不需要多次分批加锁或释放锁
```

#### <a name="35">Java中的阻塞队列


- ArrayBlockingQueue：由数据结构组成的有界阻塞队列
- LinkedBlockingQueue：由链表结构组成的有界阻塞队列
- PriorityBlockingQueue：支持优先级排序的无界阻塞队列
- DelayQueue：使用优先级队列实现的无界阻塞队列
- SynchronousQueue：不存储元素的阻塞队列
- LinkedTransferQueue：由链表结构组成的无界阻塞队列
- LinkedBlockingDeque：由链表结构组成的双向阻塞队列

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530113845488.png)

**ArrayBlockingQueue**

```
用数组实现的有界阻塞队列，此队列按照先进先出（FIFO）的原则对元素进行排序。默认情况下不保证访问者公平的访问队列
```

**LinkedBlockingQueue**（两个独立锁提高并发）

```
基于链表的阻塞队列，此队列按照先进先出的原则对元素进行排序。
LinkedBlockingQueue之所以能高效的处理并发数据，还因为其对于生产者端、消费者端分别采用了独立的锁来控制数据同步，这意味着在高并发的情况下生产者和消费者可以并行地操作队列中的数据，以此来提高整个队列的并发性能
LinkedBlockingQueue 会默认一个类似无限大小的容量（Integer.MAX_VALUE）
```

**PriorityBlockingQueue**（compareTo 排序实现优先）

```
一个支持优先级的无界队列，默认情况下元素采取自然顺序升序排列，可以自定义实现compareTo()方法来指定元素进行排序，或者初始化PriorityBlockingQueue时，指定构造参数Comparator来对元素进行排序，不能保证同优先级元素的顺序
```

**DelayQueue**（缓存失效、定时任务）

```
一个支持延时获取元素的无界阻塞队列，队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素，只有才延迟期满时才能从队列中提取元素
应用场景如下：
	-缓存系统的设计：可以用 DelayQueue 保存缓存元素的有效期，使用一个线程循环查询 DelayQueue，一旦能从 DelayQueue 中获取元素时，表示缓存有效期到了
	-定时任务调度：使用 DelayQueue 保存当天将会执行的任务和执行时间，一旦从DelayQueue 中获取到任务就开始执行，从比如 TimerQueue 就是使用 DelayQueue 实现的
```

**SynchronousQueue**（不存储数据、可用于传递数据）

```
一个不存储元素的阻塞队列。每一个put操作必须等待一个take操作，否则不能继续添加元素。
SynchronousQueue 可以看成是一个传球手，负责把生产者线程处理的数据直接传递给消费者线程。队列本身并不存储任何元素，非常适合于传递性场景,比如在一个线程中使用的数据，传递给另外一个线程使用， SynchronousQueue 的吞吐量高于 LinkedBlockingQueue 和ArrayBlockingQueue
```

**LinkedTransferQueue**

```
由一个链表结构组成的无界阻塞 TransferQueue 队列。相对于其他阻塞队列，多了transfer、tryTransfer方法
	-transfer：如果当前有消费者正在等待接收元素，transfer方法可以把生产者传入的元素立刻transfer给消费者，若没有消费者在等待接收元素，transfer方法会将元素存放在队列的 tail 节点，并等到该元素被消费者消费了才返回
	-tryTransfer：用来试探生产者传入的元素是否能直接传给消费者，若没有消费者等待接收元素，则返回false
```

**LinkedBlockingDeque**

```
一个由链表结构组成的双向阻塞队列。
相比其他的阻塞队列，多了addFirst，addLast，offerFirst，offerLast，peekFirst，peekLast 等方法。
在初始化 LinkedBlockingDeque 时可以设置容量防止其过渡膨胀。另外双向阻塞队列可以运用在“工作窃取”模式中
```

### <a name="36">CountDownLatch、CyclicBarrier、Semaphore 的用法


**CountDownLatch**

```
CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能
```

**CyclicBarrier**

```
字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行
当所有等待线程都被释放后，CyclicBarrier可以别重用，我们暂且把这个状态叫做barrier，当调用await方法，线程就处于barrier
	-public int await()：用来挂起当前线程，直至所有线程到达barrier状态再同时执行后续任务
	-public int await(long timeout, TimeUnit unit)：让这些线程等待至一定的时间，若还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务
```

**Semaphore**（信号量-控制同时访问的线程个数）

```
通过acquire()获取一个许可，若没有就等待，而release()释放一个许可
Semaphore类中比较重要的几个方法：
	-public void acquire()：用来获取一个许可，若无许可能够获得，会一直等待，直至获得许可
	-public void acquire(int permits)：获取permits个许可
	-public void release() { }：释放许可
	-public void release(int permits) { }：释放permits个许可
	
以上4个方法都会被阻塞，若想立即得到执行结构，可以使用下面几个方法：
	-public boolean tryAcquire():尝试获取一个许可，若获取成功，则立即返回 true，若获取失败，则立即返回 false
	-public boolean tryAcquire(long timeout, TimeUnit unit):尝试获取一个许可，若在指定的时间内获取成功，则立即返回 true，否则则立即返回 false
	-public boolean tryAcquire(int permits):尝试获取 permits 个许可，若获取成功，则立即返回 true，若获取失败，则立即返回 false
	-public boolean tryAcquire(int permits, long timeout, TimeUnit unit): 尝试获取 permits个许可，若在指定的时间内获取成功，则立即返回 true，否则则立即返回 false
	-还可以通过 availablePermits()方法得到可用的许可数目
```

- CountDownLatch 一般用于某个线程 A 等待若干个其他线程执行完任务之后，它才执行；而 CyclicBarrier 一般用于一组线程互相等待至某个状态，然后这一组线程再同时 执行
- CountDownLatch 是不能够重用的，而 CyclicBarrier 是可以重用的
- Semaphore 其实和锁有点类似，它一般用于控制对某组资源的访问权限

### <a name="37">volatile关键字的作用


Java 语言提供了一种稍弱的同步机制，即 volatile 变量，用来确保将变量的更新操作通知到其他线程

volatile 变量具备两种特性：变量可见性、禁止重排序

- **变量可见性**：保证该变量对所有线程可见，这里的可见性指当一个线程修改了变量的值，那么新的值对于其他线程是可以立即获取的
- **禁止重排序**：volatile禁止了指令重排

**比synchronized更轻量级的同步锁**：在访问volatile变量时不会执行加锁的操作，因此也就不会使执行线程阻塞

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530123208429.png)

当对非volatile变量进行读写时，每个线程先从内存拷贝变量到CPU缓存中，若计算机有多个CPU，每个线程可能在不同CPU上被处理，这意味着每个线程拷贝到不同的 CPU chace中。声明变量是 volatile的，JVM保证了每次读变量都从内存中读，跳过CPU cache这一步

**适用场景**：一个变量被多个线程共享，线程直接给这个变量赋值。volatile 变量的单次读/写操作可以保证原子性的，如 long 和 double 类型变量， 但是并不能保证 i++这种操作的原子性，因为本质上 i++是读、写两次操作。在某些场景下可以代替 Synchronized。但是,volatile 的不能完全取代 Synchronized 的位置，只有在一些特殊的场景下，才能适用 volatile。总的来说，必须同时满足下面两个条件才能保证在并发环境的线程安全

- 对变量的写操作不依赖于当前值（如i++），或者说是单纯的变量赋值（boolean flag = true）
- 该变量没有包含在具有其他变量的不变式中，也就是说，不同的 volatile 变量之间，不能相互依赖。只有在状态真正独立于程序内其他内容时才能使用 volatile

### <a name="38">如何在两个线程间共享数据


Java里面进行多线程通信的主要方式：**共享内存**

共享内存主要关照点两个：可见性、有序原子性。JVM解决了可见性、有序性，而锁解决了原子性。

理想情况下，我们希望做到 同步 和互斥，有以下常规实现方法：

- **将数据抽象成一个类，并将数据的操作作为这个类的方法**：在方法上加 synchronized
- **Runnable对象作为一个类的内部类**：共享数据作为这个类的成员变量，每个线程对共享数据的操作方法也封装在外部类，以便实现对数据的各个操作的同步和互斥，作为内部类的各个 Runnable 对象调用外部类的这些方法

### <a name="39">ThreadLocal


线程本地变量。提供线程内的局部变量，这种变量在线程的**生命周期内起作用**，减少同一个线程内多个函数或者组件之间一些公共变量传递的复杂度

**ThreadLocalMap（线程的一个属性）**

- 每个线程中都有一个自己的 ThreadLocalMap 类对象，可将线程自己的对象保持到其中， 各管各的，线程可正确访问到自己的对象
- 将一个共用的 ThreadLocal 静态实例作为 key，将不同对象的引用保存到不同线程的 ThreadLocalMap 中，然后在线程执行的各处通过这个静态 ThreadLocal 实例的 get()方法取得自己线程保存的那个对象，避免了将这个对象作为参数传递的麻烦
- 它在 Thread 类中定义：ThreadLocal.ThreadLocalMap threadLocals = null

**使用场景**：最常见的 ThreadLocal 使用场景为 用来解决 数据库连接、Session管理等

### <a name="40">Synchronized与ReentrantLock的区别


**相同点**：

- 都用来协调多线程共享对象、变量的访问
- 都是可重入锁，同一线程可多次获得同一个锁
- 都保证了可见性、互斥性

**不同点**：

- ReentrantLock显示获得锁、释放锁；Synchronized隐式获得、释放锁
- ReentrantLock可响应中断、轮回；Synchronized不可以响应中断，为处理锁的不可用性提供了更高的灵活性
- ReentrantLock是API级别；Synchroized是JVM级别
- ReentrantLock可以实现公平锁、通过Condition可以绑定多个条件
- lock是同步非阻塞，乐观并发策略；Synchronized是同步阻塞，悲观并发策略
- Lock是一个接口；Synchronized是Java中的关键字，synchronized是内置的语言实现
- Lock发生异常时，若没有主动通过unLock()去释放锁，可能造成死锁现象；Synchronized在发生异常时，会自动释放线程占用的锁
- Lock可以让等待锁的线程响应中断；Synchronized会让等待的线程一直等待下去，不能响应中断
- Lock可以知道有没有成功获取锁；Synchronized无法办到

### <a name="41">ConcurrentHashMap并发


**减小锁粒度**

```
减小锁粒度是指缩小锁定对象的范围，从而减小锁冲突的可能性，提高系统的并发能力。减小锁粒度是一种削弱多线程锁竞争的有效手段，这种技术典型的应用是 ConcurrentHashMap(高性能的HashMap)类的实现。对于HashMap而言，最重要的两个方法是 get 与 set 方法，如果我们对整个 HashMap加锁，可以得到线程安全的对象，但是加锁粒度太大。Segment 的大小也被称为 ConcurrentHashMap的并发度
```

**ConcurrentHashMap 分段锁**

```
ConcurrentHashMap，它的内部细分了若干个小的 HashMap，称之为段（Segment）。默认情况下一个 ConcurrentHashMap 被进一步细分为16个段，即为锁的并发度
若需要在 CouncurrentHashMap 中添加一个新的表项，并不是将整个 HashMap 加锁，而是首先根据 hashcode 得到该表项应该存放在哪个段中，然后对该段加锁，完成put操作。在多线程环境中，若多个线程同时进行 put 操作，只要被加入的表项不存放在同一个段中，则线程间可以做到真正的并行
```

**ConcurrentHashMap 是由 Segment 数组结构和 HashEntry 数组结构组成**

- Segment是一种可重入锁 ReentrantLock，HashEntry则用于存储键值对数据
- 一个 ConcurrentHashMap 里包含一个 Segment 数组，Segment 的结构和 HashMap 类似，是一种数组和链表结构，一个 Segment 里包含一个 HashEntry 数组，每个HashEntry 是一个链表结构的元素
- 每个 Segment 守护一个 HashEntry 数组里的元素，当对HashEntry 数组的数据进行修改时，必须首先获得它对应的 Segment 锁

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530160700049.png)

### <a name="42">Java中用到的线程调度


#### <a name="43">抢占式调度


抢占式调度指的是每条线程执行的时间、线程的切换都由系统控制

系统控制指的是在系统某种运行机制下，可能每条线程都分同样的执行时间片，也可能是某些线程执行的时间片教长，甚至某些线程得不到执行的时间时间片。在这种机制下，一个线程的堵塞不会导致整个进程堵塞

#### <a name="44">协同式调度


协同式调度指某一线程执行完后主动通知系统切换到另一线程上执行，这种模式就想接力赛意义，一个人跑完自己的路程就把接力棒交接给下一个人，下个人继续往下跑

线程的的执行时间由线程本身控制，线程切换可以预知，不存在多线程同步问题，但它有一个致命弱点：若一个线程编写有问题，运行到一半就一直堵塞，可能导致整个系统崩溃

<img src="https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530161340515.png" style="zoom:67%;" />

**JVM的线程调度实现**：java使用抢占式调度，线程会按优先级分配CPU时间片运行，**且优先级越高越优先执行，但优先级高并不代表能独自占用执行时间片**，

**线程让出CPU的情况**：

- 当前运行线程主动放弃CPU，JVM暂时放弃CPU操作（基于时间片轮转调度的 JVM 操作系统 不会让线程永久放弃 CPU，或者说放弃本次时间片的执行权），例如调用yield()方法
- 当前运行线程因为某些原因进入阻塞状态，例如阻塞在 I/O上
- 当前运行线程结束，即运行完 run()方法里面的任务

### <a name="45">CAS


CAS（Compare And Swep/Set），CAS算法过程如下：它包含3个参数CAS(V,E,N)

- V：表示要更新的变量（内存值）
- E：表示预期值（旧的）
- N：表示新值

当前仅当 V = E 时，才会将 V 值设为 N；若V ≠ N，则说明已经有其他线程做了更新，则当前线程什么都不做。最好CAS返回当前V的真实值

CAS操作是抱着乐观的态度进行（乐观锁），总认为自己可以成功完成操作。当多个线程同时使用 CAS 操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会被挂起，仅是被告知失败，并且允许再次尝试，也允许失败的线程放弃操作。基于这样的原理，CAS操作即使没有锁，也可以发现其他线程对于当前线程的干扰，并进行恰当的处理

**ABA问题**

```
CAS会导致“ABA”问题，CAS算法实现一个重要前提需要取出内存中某时刻的数据，而在下时刻比较并替换，那么在这个时间差类会导致数据变化
例如：一个线程 one 从内存位置 V 取出 A，这时候另一个线程 two 也从内存中取出 A，并且 two 进行了一些操作变成了 B，然后 two 又将 V 位置的数据变成了A，这时候线程 one 进行 CAS操作发现内存中仍然是 A，然后 onw 操作成。尽管线程 one 的CAS操作成功，但不代表这个过程没有问题

部分乐观锁的实现是通过版本号的方式来解决 ABA 问题，乐观锁每次在执行数据的修改操作时，都会带上一个版本号，一旦版本号和数据的版本号一致就可以执行修改操作并对版本号执行 +1 操作，否则就执行失败，因为每次操作的版本号都会随之增加，不会出现 ABA 问题，因为版本号只会增加不会减少
```

### <a name="46">AQS


AbstractQueuedSynchronizer类如其名，抽象的队列式的同步器，AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如常用的：ReentrantLock、Semaphore、CountDownLatch

![](https://yingziimage.oss-cn-beijing.aliyuncs.com/img/image-20220530171233303.png)

y以上维护了一个 volatile int state（代表共享资源）和一个 FIFO 线程等待队列（多线程争用资源被阻塞时就会进入此队列）。这里 volatile 是核心关键词

state的访问方式有三种：getState()、steState()、compareAndSetState()

**AQS定义两种资源共享方式**

- Exclusive 独占资源，独占，只有一个线程能执行，如ReentrantLock
- Share 共享资源，共享，多个线程可同时执行，如Semaphore、CountDownLoatch

**AQS只是一个框架，具体资源的获取/释放方式交由自定义同步器去实现**

通过state的get/set/CAS，之所以没有定义成abstract，是因为独占模式下只用实现tryAcquire-tryRelease，而共享模式下只用实现

tryAcquireShared-tryReleaseShared。若都定义成abstractt，那么每个模式也要去实现另一模式下的接口。不同的自定义同步器争用共享资源的方式也不同，自定义同步器在实现时只需要实现共享资源 state 的获取与释放方式即可，至于具体线程等待对垒的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了，自定义同步器实现时主要实现以下几种方法：

- isHeldExclusively()：该线程是否正在独占资源，只有用到 condition 才需要去实现它
- tryAcquire(int)：独占方式，尝试获取资源，成功则返回 true，失败则返回false
- tryRelease(int)：独占方式，尝试释放资源，成功则返回 true，失败则返回false
- tryAcquireShared(int)：共享方式，尝试获取资源。负数代表失败；0代表成功，但没有剩余可用资源；正数表示成功，且有剩余资源
- tryReleaseShared(int)：共享方式，尝试释放资源。若释放后允许唤醒后续等待节点返回 ture，否则返回 false

**同步器的实现是 ABS 核心**

```
同步器的实现时 ABS 核心，以 ReentrantLock 为例，state 初始化为0，表示未锁定状态。A 线程lock()时，会调用 tryAcquire()独占锁并将 state + 1。此后，其他线程再tryAcquire()时就会失败，直到A线程，unlock()到state = 0(释放锁)为止，其他线程才有机会获取该锁。当然，释放锁之前，A线程自己值可以重复获取此锁（state累加），这就是重入的概念。但要注意，获取多少次就要释放多少次，这样才能保证state是能回零态的

例子：以 CountDownLatch 以例，任务分为 N 个子线程去执行，state 也初始化为 N (注意N要与 线程个数一直)，这 N 个子线程是并行执行，每个子线程执行完成后 countDown()一次，state会 CAS 减1，等到所有子线程都执行完后(state = 0)，会unpark()调用线程，然后主调用线程就会从 await()函数返回，继续后余动作
```

**ReentrantReadWriteLock实现独占锁和共享两种方式**

```
一般来说，自定义同步器要么是独占方法，要么是共享方式，它们也只需要实现 tryAcquire-tryRelease、ryAcquireShared-tryReleaseShared中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如 ReentrantReadWriteLock
```

