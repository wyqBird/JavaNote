# 一、多线程技能

## 1 多线程知识点

四种实现方法

- （1）extends Thread，重写 run()
- （2）implements Runnable，重写 run()
- （3）implements Callable，重写 call()
- （4）线程池实现

区别：

- 用 Thread 单一继承，用 Runnable 可实现多继承效果
- Callable 有返回值， Runnable 没有返回值；
- Callable 中的 call 可抛异常， Runnable 中的 run 不可；

Thread 中的 start 只是通知“线程规划器”此线程准备就绪，等待调用线程对象的 run，执行 start 方法的顺序不代表线程启动的顺序。

非线程安全主要指多线程对同一个对象中同一个实例变量进行操作时会出现值被更改、值不同步的情况，进而影响程序的执行流程。

## 2 常用方法

- currentThread()：返回代码段正在被哪个线程调用的信息；
- isAlive()：判断当前线程是否处于活动状态；
- sleep()：在指定毫秒数内让当前“正在执行的线程”暂停执行，这个“正在执行的线程”是指 this.currentThread() 返回的线程；
- getId()：取得线程的唯一标识
- yield()：放弃当前 CPU 资源，但放弃的时间不确定。

注意：

在自定义线程类时，如果线程类是继承java.lang.Thread的话，那么线程类就可以使用this关键字去调用继承自父类Thread的方法，this就是当前的对象。

另一方面，Thread.currentThread()可以获取当前线程的引用，一般都是在没有线程对象又需要获得线程信息时通过Thread.currentThread()获取当前代码段所在线程的引用。

尽管this与Thread.currentThread() 都可以获取到Thread的引用，但是在某种情况下获取到的引用是有差别的，下面进行举例说明：

```Java

MyThread myThread = new MyThread();
myThread.setName("A");
myThread.start();

```

这种情况下，开始运行 run 后：

- Thread.currentThread().getName() = A
- this.getName() = A

这两个是同一个引用。

```Java

MyThread myThread = new MyThread();
// 将线程对象以构造参数的方式传递给Thread对象进行start（）启动线程
Thread newThread = new Thread(myThread);		
newThread.setName("A");		
newThread.start();

```

这种情况下，开始运行 run 后：

- Thread.currentThread().getName() = A
- this.getName() = Thread-0

这两个不是同一个引用。

看一下源码：

```Java

public Thread() {
	init(null, null, "Thread-" + nextThreadNum(), 0);
}
public Thread(Runnable target) {
	init(null, target, "Thread-" + nextThreadNum(), 0);
}

```

解释：

- 将线程对象以构造参数的方式传递给Thread对象进行start（）启动线程，我们直接启动的线程实际是newThread，而作为构造参数的myThread，赋给Thread类中的属性target，之后在Thread的run方法中调用target.run()；
- 此时Thread.currentThread()是Thread的引用newThread, 而this依旧是MyThread的引用，所以是不一样的，打印的内容也不一样


## 3 停止线程

- Thread.stop()：可停止一个线程，但不安全，已弃用；
- Thread.interrupt()：不会终止一个正在执行的线程，需加入一个判断才能完成线程的停止。

三种方法终止正在运行的线程：

- 使用退出标志，使线程正常退出，也就是当 run 执行完毕后线程终止；
- 使用 stop 强行停止，不推荐；
- 使用 interrupt 中断线程。

说明：

- 调用 interrupt() 方法仅仅是在当前线程中打了一个停止的标记，并不是真正停止线程；

判断线程是否是停止状态：

- this.interrupted()：测试当前线程是否已经是中断状态，执行后具有将状态标志清除为 false 的功能。
- this.isInterrupted()：测试线程 Thread 对象是否已经是中断状态，但不清除状态标志。

使用 return 停止线程：

- 将 interrupt() 与 return 结合使用也能实现停止线程的效果。
- 但这种方法，使得 return 后，异常不会抛出，线程停止的事件无法传播。
- 所以建议使用 “抛异常”的方法来实现线程的停止，因为在 catch 块中还可以将异常上抛，使线程停止的事件得以传播。

## 4 暂停线程

suspend()：暂停线程

resume()：恢复线程

缺点：

- 独占：但这两个方法极易造成对同步对象的独占，使得其他线程无法访问公共同步对象。
- 不同步：容易出现线程暂停导致数据不同步的现象。

## 5 优先级 

在操作系统中，线程可划分优先级，优先级较高的线程得到的 CPU 资源较多，优先执行。

Java 中优先级分为 1~10，如果 < 1 || > 10，则 throw new IllegalArgumentException().

- MIN_PRIORITY = 1;
- NORM_PRIORITY = 5;
- MAX_PRIORITY = 10;

Java 线程优先级特性：

- 继承性：A线程启动B线程，则B线程和A线程优先级相同；
- 规则性：CPU 尽量将执行资源让给优先级比较高的线程；
- 随机性：优先级高的线程不一定每次都先执行完；

守护线程：

- 当进程中不存在非守护线程时，守护线程自动销毁
- eg：垃圾回收线程
- 守护线程主要就是为其他线程的运行提供便利服务

# 二、对象及变量的并发访问

## 1 synchronized 同步方法

- “非线程安全”问题存在于“实例变量”中，若是方法内部的私有变量，则不存在“非线程安全”问题，这是方法内部的变量是私有的特性造成的。
- synchronized 取得的锁都是对象锁。当一个线程执行的代码出现异常时，其所持有的锁会自动释放。
- synchronized 可重入：当一个线程得到一个对象的锁后，再次请求此对象锁时是可以再次得到该对象的锁的，如果不可重入的话，会造成死锁。
- 可冲入锁也支持在父子类继承的环境中：当存在父子类继承关系时，子类是完全可以通过“可重入锁”调用父类的同步方法。
- 同步不可用继承：子类无法继承父类的同步方法。

## 2 synchronized 同步代码块

synchronized 同步方法在某些情况下因锁住的资源较多，耗时较长，可改用 synchronized 同步代码块，实现功能的同时提高效率。

- 当一个线程访问某个对象的一个 synchronized 同步代码块时，另一个线程仍可以访问该对象中的非 synchronized(this) 同步代码块。不在同步代码块中就是异步，在同步代码块中就是同步。

- 当一个线程访问对象的一个 synchronized(this) 同步代码块时，其他线程对同一个对象中所有其他 synchronized(this) 同步代码块的访问将被阻塞，因为 synchronized 使用的“对象监视器”是同一个。

和 synchronized 方法一样，synchronized(this) 代码块也是锁定当前对象的。

- 多个线程调用同一个对象中的不同名称的 synchronized 同步方法或 synchronized(this) 同步代码块时，是按顺序进行，同步阻塞的。

- synchronized 同步方法：对其他 synchronized 同步方法或 synchronized(this) 同步代码块调用呈阻塞状态；同一时间只有一个线程可以执行 synchronized 同步方法中的代码。

- synchronized(this) 同步代码块：对其他 synchronized 同步方法或 synchronized(this) 同步代码块呈阻塞状态；同一时间只有一个线程可以执行 synchronized(this) 同步代码块中的代码。

将任意对象作为对象监视器：synchronized(非 this 对象)

- 在多个线程持有“对象监视器”为同一个对象的前提下，同一时间只有一个线程可以执行 synchronized(非 this 对象 x) 同步代码块中的代码。

- 当持有“对象监视器”为同一个对象的前提下，同一时间只有一个线程可以执行 synchronized(非 this 对象 x) 同步代码块中的代码。

锁非 this 对象的优点：若在一个类中有很多个 synchronized 方法，这是虽然能实现同步，但会收到阻塞，所以影响效率；但若使用同步代码块锁非 this 对象，则 synchronized(非 this) 代码块中的程序与同步方法是异步的，不与其他锁 this 同步方法争抢 this 锁，可大大提高效率。

synchronized(非 this 对象 x) 总结：

- 当多个线程同时执行 synchronized(x){} 同步代码块时呈同步效果；
- 当其他线程执行 x 对象中 synchronized 同步方法时呈同步效果；
- 当其他线程执行 x 对象方法里面的 synchronized(this) 代码块时也呈现同步效果。
- 若其他线程调用不加 synchronized 关键字的方法时，还是异步调用。

## 3 静态同步 synchronized 方法与 synchronized(class) 代码块

synchronized 用在 static 静态方法上，是对当前的 *.java 文件对应的 Class 类进行加锁，而 synchronized 关键字加到非静态方法上是给对象加锁。Class锁可以对类的所有对象实例起作用。

synchronized(class)代码块的作用其实和 synchronized static 方法的作用一样。

在将任何数据类型作为同步锁时，需要注意的是，是否有多个线程同时持有锁对象，若同时持有相同锁对象，则这些线程之间是同步的；若分别获得锁对象，这些线程之间是异步的。

注意String的常量池特性（缓存），所导致的例外。在大多数情况下，同步 synchronized 代码块都不使用 String 作为锁对象。

## 4 volatile

volatile：强制从公共堆栈中取得变量的值，而不是从线程私有数据栈中取得变量的值。

volatile和synchronized：

- volatile是线程同步的轻量级实现，性能比synchronized稍好；
- volatile只能修饰变量，synchronized可修饰方法、代码块；
- 多线程访问 volatile 不发生阻塞，而 synchronized 会出现阻塞；
- volatile只能保证数据可见性，synchronized 可保证原子性和可见性，它会将私有内存和公共内存做数据同步；
- volatile解决的是变量在多个线程之间的可见性，synchronized解决的是多线程之间访问资源的同步性。

# 三、线程间通信

## 1 等待/通知机制

wait()：

- 使当前执行代码的线程进行等待
- Object 类的方法
- 将当前线程置入“预执行队列”，并在 wait 所在的代码行处停止执行，直到接到通知或被中断为止
- 调用 wait 前，线程必须获得该对象的锁，只能在同步方法/代码块中调用 wait 方法
- 执行完 wait 方法后，当前线程释放锁
- wait 返回前，线程与其他线程竞争重新获锁
- 若调用 wait 时没有持有适当的锁，则抛出 IllegalMonitorStateException，它是 RuntimeException 的一个子类，不需要 try-catch 捕获异常
- 在执行同步代码块的过程中，遇到异常而导致线程终止，锁也会被释放

notify()：

- 通知那些可能等待该对象的对象锁的其他线程，如果有多个线程等待，则由线程规划器随机挑选出其中一个呈 wait 状态的线程，对其发送通知 notify，并使它等待获取该对象的对象锁
- 只是随机叫醒一个线程
- 在同步方法/代码块中调用
- 调用前需获得该对象的对象级别锁
- 若调用 notify 时没有持有适当的锁，则抛出IllegalMonitorStateException
- 执行 notify 方法后，当前线程不会马上释放该对象锁，呈 wait 状态的线程也并不能马上获取该对象锁，要等到执行 notify 方法的线程将程序执行完，也就是退出同步代码块之后，当前线程释放锁，此时呈 wait 状态所在的线程才能获取该对象锁

notifyAll()：

- 使所有正在等待队列中等待同一共享资源的“全部”线程从等待状态退出，进入可执行状态。此时优先级最高的那个线程最先执行，但也有可能随机执行。

当 wait() 方法被执行后，锁被自动释放，但执行完 notify 方法后，锁不自动释放。必须等到 notify 所在的同步代码块执行完毕后，锁才释放。

## 2 join方法

- 等待线程对象销毁
- 使所属的线程对象 x 正常执行 run() 方法中的任务，而使当前线程 z 进行无限期的阻塞，等待线程 x 销毁后再继续执行线程 z 后面的代码
- 具有使线程排队运行的作用
- 与 synchronized 区别：join 在内部使用 wait 方法进行等待，而 synchronized 关键字使用的是“对象监视器”原理做同步
- join 过程中，若当前线程对象被中断，则当前线程出现异常
- join 和 sleep 区别：join 在内部使用 wait 方法实现，所以具有释放锁的特点；Thread.sleep()方法不释放锁。

## 3 ThreadLocal类

- ThreadLocal 解决的是变量在不同线程间的隔离性，也就是不同线程拥有自己的值，不同线程中的值是可以放入 ThreadLocal 类中进行保存的。

## 4 InheritableThreadLocal类

- 使用 InheritableThreadLocal 可以在子线程中取得父线程继承下来的值
- 若子线程在取得值得同时，主线程将 InheritableThreadLocal 中的值进行修改，那么子线程取到的值还是旧值

# 四、Lock

## 1 ReentrantLock 类

## 2 ReentrantReadWriteLock 类

# 五、定时器 Timer

# 六、单例模式

# 七、其他