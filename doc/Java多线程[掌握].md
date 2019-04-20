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

## 2 synchronized 同步代码块

## 3 volatile

# 三、线程间通信

## 1 等待/通知机制

## 2 join方法

## 3 ThreadLocal类

## 4 InheritableThreadLocal类

# 四、Lock

## 1 ReentrantLock 类

## 2 ReentrantReadWriteLock 类

# 五、定时器 Timer

# 六、单例模式

# 七、其他