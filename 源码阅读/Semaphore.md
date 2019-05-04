# 1. 简介

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以保证合理的使用公共资源。

也就是说，它允许n个任务同时访问某个资源，可以将信号量看做是在向外分发使用资源的许可证，只有成功获取许可证，才能使用资源。

# 2. 应用场景

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。

假如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连接。这个时候，就可以使用Semaphore来做流量控制.

eg:

```Java

public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i< THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {}
                }
            });
        }
        threadPool.shutdown();
    }
}


```

# 3. 源码

## 3.1 acquire() 获取资源

```Java

public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

```

由上可以看出，这里是**响应中断获取资源**。当然，也有不响应中断获取资源的，暂时没看。

acquireSharedInterruptibly方法先检测中断。然后调用tryAcquireShared方法试图获取共享资源。这时公平模式和非公平模式的代码执行路径发生分叉，FairSync和NonfairSync各自重写tryAcquireShared方法。

### 非公平模式下的tryAcquireShared方法：

```Java

protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
	//CAS自旋
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

```

因为Semaphore是一个共享锁，可能有多个线程同时申请共享资源，因此CAS操作可能失败。

直到成功获取，返回剩余资源数目；或者发现没有剩余资源，返回负值代表申请失败。

有一个问题，为什么我们不在CAS操作失败后就直接返回失败呢？

- 因为这样做虽然不会导致错误，但会降低效率：在还有剩余资源的情况下，一个线程因为竞争导致CAS失败后被放入等待序列尾，一定在队列头部有一个线程被唤醒去试图获取资源，这比直接自旋继续获取多了操作等待队列的开销。

这里“非公平”的语义体现在：如果一个线程通过nonfairTryAcquireShared成功获取了共享资源，对于此时正在等待队列中的线程来说，可能是不公平的：队列中线程先到，却没能先获取资源。

如果tryAcquireShared没能成功获取，acquireSharedInterruptibly方法调用doAcquireSharedInterruptibly方法将当前线程放入等待队列并开始自旋检测获取资源：

```Java

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```

doAcquireSharedInterruptibly中，当一个线程从parkAndCheckInterrupt方法中被中断唤醒之后，直接抛出了中断异常。**这里就是及时响应中断。**

### 公平模式下的tryAcquireShared方法：

```Java

protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}

```

相比较非公平模式的nonfairTryAcquireShared方法，公平模式下的tryAcquireShared方法在试图获取之前做了一个判断:

如果发现等待对队列中有线程在等待获取资源，就直接返回-1表示获取失败。

当前线程会被上层的acquireSharedInterruptibly方法调用doAcquireShared方法放入等待队列中。这正是“公平”模式的语义：如果有线程先于我进入等待队列且正在等待，就直接进入等待队列，效果便是各个线程按照申请的顺序获得共享资源，具有公平性。

## 3.2 release() 释放资源

公平模式和非公平模式的释放资源操作是一样的：

```Java

public void release() {
	//调用AQS提供的releaseShared方法
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}


```

releaseShared方法首先调用我们重写的tryReleaseShared方法试图释放资源。然后调用doReleaseShared方法唤醒队列之后的等待线程。

```Java

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}

```

这个方法也是一个CAS自旋，原因是因为Semaphore是一个共享锁，可能有多个线程同时释放资源，因此CAS操作可能失败。最后方法总会成功释放并返回true（如果不出错的话）。