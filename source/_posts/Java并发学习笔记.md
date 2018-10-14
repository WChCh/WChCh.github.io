---
title: Java并发学习笔记
date: 2017-12-26 17:17:46
tags: [Java,并发]
categories: [技术,学习,Java,并发]
---

#### 并发基础

##### synchronized关键字

1. 对某个对象加锁
2. 当修饰静态方法或是代码块时，表示对Class对象加锁。
3. 同步和非同步方法可以同时调用。
4. 一般情况下要对读方法和写方法同时加锁，要不很可能会出现脏读问题。
5. 一个同步方法可以调用另外一个同步方法，一个线程已经拥有某个对象的锁，再次申请的时候仍然会得到该对象的锁，synchronized获得的锁是可重入的。
6. 程序在执行过程中，如果出现异常，默认情况锁会被释放。
7. 同步代码块中的语句越少越好。
8. 锁定某对象o，如果o的属性发生改变，不影响锁的使用，但是如果o变成另外一个对象，则锁定的对象发生改变，应该避免将锁定对象的引用变成另外的对象。
9. 不要以字符串常量作为锁定对象，因为你的程序和你用到的类库不经意间使用了同一把锁，比如相同的字符串常量。

##### volatile关键字

volatile只能保证可见性，但是不保证原子性。

1. volatile并不能保证多个线程共同修改同一变量时所带来的不一致问题，也就是说volatile不能替代synchronized。
2. nchronized可以保证可见性和原子性，volatile只能保证可见性。

<!--more-->

##### 原子操作类

使用原子方式更新基本类型，共包括3个类：

- AtomicBoolean：原子更新布尔变量
- AtomicInteger：原子更新整型变量
- AtomicLong：原子更新长整型变量

解决同样的问题的更高效的方法，使用AtomXXX类，AtomXXX类本身方法都是原子性的，但不能保证多个方法连续调用是原子性的。

```java
/*volatile*/ //int count = 0;

AtomicInteger count = new AtomicInteger(0); 

/*synchronized*/ void m() { 
for (int i = 0; i < 10000; i++)
	//if count.get() < 1000
	count.incrementAndGet(); //count++
}

public static void main(String[] args) {
	T t = new T();

	List<Thread> threads = new ArrayList<Thread>();

for (int i = 0; i < 10; i++) {
	threads.add(new Thread(t::m, "thread-" + i));
}

threads.forEach((o) -> o.start());

threads.forEach((o) -> {
	try {
		o.join();
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
});

System.out.println(t.count);

}
```







#### ThreadLocal

ThreadLocal线程局部变量，是使用空间换时间，synchronized是使用时间换空间。

```java
//volatile static Person p = new Person();
static ThreadLocal<Person> tl = new ThreadLocal<>();

public static void main(String[] args) {
			
new Thread(()->{
	try {
		TimeUnit.SECONDS.sleep(2);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
	tl.set(new Person());
	System.out.println(tl.get().name);
}).start();

new Thread(()->{
	try {
		TimeUnit.SECONDS.sleep(1);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
	tl.set(new Person());
	System.out.println(tl.get());
}).start(); 
}

static class Person {
	String name = "zhangsan";
}
```



#### Java中的锁

##### Lock接口

锁能够防止多个线程同时访问共享资源。Lock接口在使用时需要显示地获取和释放锁（Synchronized是隐式的），并且是可中断、可超时获取等。

```java
ReentrantLock1 rl = new ReentrantLock1();
Lock1 r2 = new ReentrantLock1();
```



##### 队列同步器AbstractQueuedSynchronizer(AQS)

队列同步器AbstractQueuedSynchronizer(AQS)，似乎我们不经常用，但是它是用来构建锁或者其他同步组件的基础框架，它使用了一个int成员变量表示同步状态，通过内置的FIFO队列来完成资源获取线程的排队工作。

##### 重入锁

推荐使用reentrantlock用于替代synchronized。

1. 需要注意的是，必须要必须要必须要手动释放锁。使用syn锁定的话如果遇到异常，jvm会自动释放锁，但是lock必须手动释放锁，因此经常在finally中进行锁的释放。示例：

   ```java
   try {
   	lock.lock(); //synchronized(this)
   	for (int i = 0; i < 10; i++) {
   		TimeUnit.SECONDS.sleep(1);
   
   		System.out.println(i);
   		}
   	} catch (InterruptedException e) {
   			e.printStackTrace();
   	} finally {
   	   lock.unlock();
   }
   ```

2. 使用reentrantlock可以进行“尝试锁定”tryLock，这样无法锁定，或者在指定时间内无法锁定，线程可以决定是否继续等待。可以根据tryLock的返回值来判定是否锁定，也可以指定tryLock的时间，由于tryLock(time)抛出异常，所以要注意unclock的处理，必须放到finally中

   ```java
    boolean locked = false;
   		
   try {
   	locked = lock.tryLock(5, TimeUnit.SECONDS);
   	System.out.println("m2 ..." + locked);
   } catch (InterruptedException e) {
   	e.printStackTrace();
   } finally {
   	if(locked) lock.unlock();
   }
   ```

1. 使用ReentrantLock还可以调用lockInterruptibly方法，可以对线程interrupt方法做出响应，实现一个线程在等待锁的过程中，可以被打断。

   ```java
   Thread t2 = new Thread(()->{
   try {
   	//lock.lock();
   	lock.lockInterruptibly(); //可以对interrupt()方法做出响应
   	System.out.println("t2 start");
   	TimeUnit.SECONDS.sleep(5);
   	System.out.println("t2 end");
   } catch (InterruptedException e) {
   	System.out.println("interrupted!");
   } finally {
   	lock.unlock();
   }
   });
   t2.start();
   
   try {
   	TimeUnit.SECONDS.sleep(1);
   } catch (InterruptedException e) {
   	e.printStackTrace();
   }
   t2.interrupt(); //打断线程2的等待
   ```

2. ReentrantLock还可以指定为公平锁

   ```java
   private static ReentrantLock lock=new ReentrantLock(true); //参数为true表示为公平锁，请对比输出结果
   public void run() {
       for(int i=0; i<100; i++) {
           lock.lock();
           try{
               System.out.println(Thread.currentThread().getName()+"获得锁");
           }finally{
               lock.unlock();
           }
       }
   }
   public static void main(String[] args) {
       ReentrantLock5 rl=new ReentrantLock5();
       Thread th1=new Thread(rl);
       Thread th2=new Thread(rl);
       th1.start();
       th2.start();
   }
   ```

##### 读写锁

读写锁在同一时刻可以允许多个读线程访问，但是在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护了一对锁，一个读锁和一个写锁，通过读写分离，使得并发性相比一般的排他性锁有了很大提升。读写锁的接口为`ReadWriteLock`，其实现为`ReentrantReadWriteLock`。

```java
ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
static Lock r = rw.readLock();
static Lock w = rw.writeLock();
```



##### Condition接口

使用Lock和Condition来实现，对比使用wait和notify/notifyAll，Condition的方式可以更加精确的指定哪些线程被唤醒。并且要像wait()使用一个while循环来做限制。

```java
	final private LinkedList<T> lists = new LinkedList<>();
final private int MAX = 10; //最多10个元素
private int count = 0;

private Lock lock = new ReentrantLock();
private Condition producer = lock.newCondition();
private Condition consumer = lock.newCondition();

public void put(T t) {
    try {
    	lock.lock();
    	while(lists.size() == MAX) { //想想为什么用while而不是用if？
    		producer.await();
    	}
    	
    	lists.add(t);
    	++count;
    	consumer.signalAll(); //通知消费者线程进行消费
    } catch (InterruptedException e) {
    	e.printStackTrace();
    } finally {
    	lock.unlock();
    }
}

public T get() {
	T t = null;
    try {
    	lock.lock();
    	while(lists.size() == 0) {
    		consumer.await();
    	}
    	t = lists.removeFirst();
    	count --;
    	producer.signalAll(); //通知生产者进行生产
    } catch (InterruptedException e) {
    	e.printStackTrace();
    } finally {
    	lock.unlock();
    }
	return t;
}

public static void main(String[] args) {
	MyContainer2<String> c = new MyContainer2<>();
	//启动消费者线程
    for(int i=0; i<10; i++) {
    	new Thread(()->{
    		for(int j=0; j<5; j++) System.out.println(c.get());
    	}, "c" + i).start();
    }
    
    try {
    	TimeUnit.SECONDS.sleep(2);
    } catch (InterruptedException e) {
    	e.printStackTrace();
    }
    
    //启动生产者线程
    for(int i=0; i<2; i++) {
    	new Thread(()->{
    		for(int j=0; j<25; j++) c.put(Thread.currentThread().getName() + " " + j);
    	}, "p" + i).start();
    }
}
```



#### 并发容器

使用并发容器代替加锁的非并发容器。

##### 几种key-value容器的使用场景：

1. 快速存取键值对时可以使用HashMap。
2. 多线程并发中存取键值对时，可以选择ConcurrentHashMap。ConcurrentHashMa使用分段锁的技术。效率比HashTable效率高。
3. 当需要存取的键值对有序时可以使用TreeMap。TreeMap保证数据是按照Key的自然顺序或者compareTo方法指定的排序规则进行排序。底层是红黑树实现。
4. 当需要多线程并发存取数据并且希望保证数据有序时，可以添加lock来实现ConcurrentTreeMap，但是随着并发量的提升，lock带来的性能开销也随之增大。此时可以选择ConcurrentSkipListMap提供了一种线程安全的并发访问的排序映射表。内部是SkipList（跳表）结构实现，在理论上能够O(log(n))时间内完成查找、插入、删除操作。

##### 写时复制容器CopyOnWriteArrayList

多线程环境下，写时效率低，读时效率高，适合写少读多的环境。

##### Collections.synchronizedList()方法

可以对某一个对象的方法加锁，但肯定效率也不高。如果Synchronized加锁。

```java
List<String> strs = new ArrayList<>();
List<String> strsSync = Collections.synchronizedList(strs);
```



##### 并发单向队列ConcurrentLinkedQueue

1. 提供有返回值的offer()方法，可判断是否添加成功。如果使用add()，如果添加未成功则抛出异常。
2. poll()方法表示拿出第一个。peek()也是拿出第一个，但是不删除。
3. 底层是单向链表实现的无界队列。

##### 并发双端队列ConcurrentLinkedDeque

1. 底层是双向列表实现，属于无界队列。

##### LinkedBlockingQueue

1. 阻塞式队列，链表实现。使用方法

   ```java
   static BlockingQueue<String> strs = new LinkedBlockingQueue<>();
   
   strs.put("a" + i); //如果满了，就会等待
   
   strs.take()); //如果空了，就会等待
   ```

2. 属于无界队列

##### ArrayBlockingQueue

1. 属于有界队列，代码如下：

   ```java
   static BlockingQueue<String> strs = new ArrayBlockingQueue<>(10);
   
   strs.put("aaa"); //满了就会等待，程序阻塞
   //strs.add("aaa");
   //strs.offer("aaa");
   //strs.offer("aaa", 1,TimeUnit.SECONDS);//1秒钟之内加不进去就不加了
   ```

##### DelayQueue

1. 属于无界队列。是有序的，等待时间最长的排在前面。
2. 是一个无界的BlockingQueue，用于放置实现了Delayed接口的对象，其中的对象只能在其到期时才能从队列中取走。这种队列是有序的，即队头对象的延迟到期时间最长。注意：不能将null元素放置到这种队列中。

##### LinkedTransferQueue

1. 如果有消费者线程，那么直接将消息送给消费者线程，而不是放在队列里面。

2. 如果没有消费者，使用的是`transfer()`方法，那么将阻塞在如下代码：

   ```java
   strs.transfer("aaa");
   ```

   如果使用的是`add()，put()`方法则将加入到队列中。

3. 队列为无界队列。

##### 没有容量的队列SynchronousQueue

SynchronousQueue是容量为0的队列，消费者必须马上消费掉，否则出现问题。使用`add()`方法直接抛出异常，可以使用`put（）`将会阻塞。

##### 总结

```java
1：对于map/set的选择使用

不需要并发：
    HashMap
    TreeMap
    LinkedHashMap

并发量小：
    Hashtable
    Collections.sychronizedXXX

并发量大：
    ConcurrentHashMap 
并发量大且排序：
ConcurrentSkipListMap 

2：队列

无需并发
    ArrayList
    LinkedList
并发量小：
    Collections.synchronizedXXX
    Vector

并发量大
    写少，读多：
        CopyOnWriteList
    Queue
    	CocurrentLinkedQueue //concurrentArrayQueue
    	ConcurrentLinkedDeque
    	BlockingQueue
    		LinkedBlockingQueue
    		ArrayBlockingQueue //有界
    		TransferQueue //可以直接传递给消费者，但会阻塞
    		SynchronusQueue //容量为0
    	DelayQueue //执行定时任务，是有序的
```

#### 并发工具类

常用的并发工具类有`CountDownLatch`，`CyclicBarrier`，`Semaphore`。用于并发流程控制。

使用Latch（门闩）替代wait notify来进行通知（不需要加锁时，如果是加锁，可能得需要Condition）:

- 好处是通信方式简单，同时也可以指定等待时间
- 使用await和countdown方法替代wait和notify
- CountDownLatch不涉及锁定，当count的值为零时当前线程继续运行
- 当不涉及同步，只是涉及线程通信的时候，用synchronized + wait/notify就显得太重了
- 这时应该考虑countdownlatch/cyclicbarrier/semaphore

区别如下([转载地址](http://blog.csdn.net/jackyechina/article/details/52931453))：

1. CountDownLatch 使一个线程A或是组线程A等待其它线程执行完毕后，一个线程A或是组线程A才继续执行。CyclicBarrier：一组线程使用await()指定barrier，所有线程都到达各自的barrier后，再同时执行各自barrier下面的代码。Semaphore：是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。
2. CountDownLatch是减计数方式，计数==0时释放所有等待的线程；CyclicBarrier是加计数方式，计数达到构造方法中参数指定的值时释放所有等待的线程。Semaphore，每次semaphore.acquire()，获取一个资源，每次semaphore.acquire(n)，获取n个资源，当达到semaphore 指定资源数量时就不能再访问线程处于阻塞，必须等其它线程释放资源，semaphore.relase()每次资源一个资源，semaphore.relase(n)每次资源n个资源。
3. CountDownLatch当计数到0时，计数无法被重置；CyclicBarrier计数达到指定值时，计数置为0重新开始。
4. CountDownLatch每次调用countDown()方法计数减一，调用await()方法只进行阻塞，对计数没任何影响；CyclicBarrier只有一个await()方法，调用await()方法计数加1，若加1后的值不等于构造方法的值，则线程阻塞。CyclicBarrier的计数器计数器可以使用reset()方法重置。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次。
5. CountDownLatch、CyclikBarrier、Semaphore 都有一个int类型参数的构造方法。CountDownLatch、CyclikBarrier这个值作为计数用，达到该次数即释放等待的线程，而Semaphore 中所有acquire获取到的资源达到这个数，会使得其它线程阻塞。

应用场景：

1. 由于CountDownLatch有个countDown()方法并且countDown()不会引起阻塞，所以CountDownLatch可以应用于主线程等待所有子线程结束后再继续执行的情况。`await()`方法应用于主线程中。

2. 由于CyclicBarrier计数达到指定后会重新循环使用，所以CyclicBarrier可以用在所有子线程之间互相等待多次的情形，作用是让所有线程到达一个屏障是被阻塞，直到最后一个线程到达屏障时，屏障才会开门。比如在某种需求中，比如一个大型的任务，常常需要分配好多子任务去执行，只有当所有子任务都执行完成时候，才能执行主任务，这时候，就可以选择CyclicBarrier了。`await()`方法应用于子线程中，告诉已经到达屏障。

3. Semaphore可以用于做流量控制，特别公用资源有限的应用场景，比如数据库连接。示例：

   ```java
   public class SemaphoreCase {  
     
       private static final int THREAD_COUNT = 30;  
       private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);  
       private static Semaphore s = new Semaphore(10);  
       public static void main(String[] args) {  
           for (int i = 0; i < THREAD_COUNT; i++) {  
               threadPool.execute(new Runnable() {  
                   @Override  
                   public void run() {  
                       try {  
                           s.acquire();  
                           System.out.println("save data");  
                           s.release();  
                       } catch (InterruptedException e) {  
                       }  
                   }  
               });  
           }  
           threadPool.shutdown();  
       }  
     
   }
   ```

注意：

1. release函数和acquire并没有要求一定是同一个线程都调用，可以A线程申请资源，B线程释放资源；
2. 调用release函数之前并没有要求一定要先调用acquire函数。

##### 线程间交换数据Exchanger

`Exchanger`用于进行线程间的数据交换。它提供一个同步点，在这个同步点两个线程可以交换彼此的数据。

#### Executor框架

##### Executor

1. Executor是一个接口，它是Executor框架的基础。
2. ThreadPoolExecutor是线程池的核心实现类。
3. ScheduledThreadPoolExecutor是一个实现类，可以在给定的延迟后运行命令，或是定期执行命令。
4. Future接口和实现Future接口的FutureTask类，代表异步计算的结果。
5. Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或ScheduledThreadPoolExecutor执行。

ThreadPoolExecutor通常使用工厂类Executors来创建，可以创建3种ThreadPoolExecutor：

1. FixedThreadPool.//固定线程数

   ```java
   public static ExecutorService newFixedThreadPool(int nThreads) {
       return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
   }
   ```

2. SingleThreadExecutor.//固定一个线程。

   ```java
   public static ExecutorService newSingleThreadExecutor() {
       return new FinalizableDelegatedExecutorService
           (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
   }
   ```

3. CachedThreadPool.//弹性线程数，空闲线程存活时间默认为60s

   ```java
   public static ExecutorService newCachedThreadPool() {
       return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                     60L, TimeUnit.SECONDS,
                                     new SynchronousQueue<Runnable>());
   }
   ```

Executors可以创建2种类型的ScheduledThreadPoolExecutor：

1. ScheduledThreadPoolExecutor.//默认等待时间最长的先运行

   ```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
       return new ScheduledThreadPoolExecutor(corePoolSize);
   }
   
   public ScheduledThreadPoolExecutor(int corePoolSize) {
       super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
             new DelayedWorkQueue());
   }
   
   public ThreadPoolExecutor(int corePoolSize,
                             int maximumPoolSize,
                             long keepAliveTime,
                             TimeUnit unit,
                             BlockingQueue<Runnable> workQueue) {
       this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
            Executors.defaultThreadFactory(), defaultHandler);
   }
   ```

2. SingleThreadScheduledExecutor.

##### ExecutorService

ExecutorService是一个接口，继承了Executor，提供了submit()方法。

```java
ExecutorService service = Executors.newFixedThreadPool(5);
service.shutdown();//所有线程跑完才关闭
service.isShutdown();//马上关闭，不管跑没跑完
service.isTerminated()；//所有的线程是否都已经执行完了
service.isShutdown()；//线程池是否已经关闭。任务完成才关闭。
```



##### Executors

Executors是一个工具类。

##### FutureTask

FutureTask实现Future接口和Runnable接口，可以得到一个线程返回值。

```java
FutureTask<Integer> task = new FutureTask<>(()->{
    TimeUnit.MILLISECONDS.sleep(500);
    return 1000;
}); //new Callable () { Integer call();}

new Thread(task).start();

System.out.println(task.get()); //阻塞
```



##### WorkStealingPool

WorkStealingPool会主动地拉取任务。会根据CPU的核数产生一样的线程数。WorkStealingPool的线程是daemon线程。是通过new ForkJoinPool来实现的，是对ForkJoinPool进行了封装。

```java
public class T11_WorkStealingPool {
	public static void main(String[] args) throws IOException {
		ExecutorService service = Executors.newWorkStealingPool();
		System.out.println(Runtime.getRuntime().availableProcessors());

		service.execute(new R(1000));
		service.execute(new R(2000));
		service.execute(new R(2000));
		service.execute(new R(2000)); //daemon
		service.execute(new R(2000));
		
		//由于产生的是精灵线程（守护线程、后台线程），主线程不阻塞的话，看不到输出
		System.in.read(); 
	}

	static class R implements Runnable {

		int time;

		R(int t) {
			this.time = t;
		}

		@Override
		public void run() {
			
			try {
				TimeUnit.MILLISECONDS.sleep(time);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			
			System.out.println(time  + " " + Thread.currentThread().getName());
			
		}

	}
}
```



##### ForkJoinPool

一个用于并行执行任务的框架，是一个把大任务分割成若干个小任务，最终汇总每个小任务结果后得到大任务结果的框架。ForkJoinPool的线程是精灵线程。

如果任务执行类继承的是RecursiveAction，那么没有返回值，如果是RecursiveTask则可以返回结果用于汇总。