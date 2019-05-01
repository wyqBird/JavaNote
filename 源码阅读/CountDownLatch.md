# 1. 简介

允许一个或多个线程等待其他线程完成操作

# 2. 例子

```Java

public void startTestCountDownLatch() {
   int threadNum = 10;
   final CountDownLatch countDownLatch = new CountDownLatch(threadNum);
   for (int i = 0; i < threadNum; i++) {
       final int finalI = i + 1;
       new Thread(() -> {
           System.out.println("thread " + finalI + " start");
           Random random = new Random();
           try {
               Thread.sleep(random.nextInt(10000) + 1000);
           } catch (InterruptedException e) {e.printStackTrace();}

           System.out.println("thread " + finalI + " finish");

           countDownLatch.countDown();
       }).start();
   }

   try {
       countDownLatch.await();
   } catch (InterruptedException e) {e.printStackTrace();}

   System.out.println(threadNum + " thread finish");

}

```

主线程启动10个子线程后，阻塞在 await 方法，等待子线程全部执行完后，再执行。

CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完
成，这里就传入N。

当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个CountDownLatch的引用传递到线程里即可。

# 3. 源码

## 3.1 构造函数

```Java

public CountDownLatch(int count) {
	// count参数的含义为：在线程可以通过await之前必须调用countDown的次数，不能小于0
    if (count < 0) throw new IllegalArgumentException("count < 0");
	//构建同步器
    this.sync = new Sync(count);
}

```

## 3.2 Sync类

```Java

//同步器
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
	//构造器
    Sync(int count) {
        setState(count);
    }
	//获取当前同步状态
    int getCount() {
        return getState();
    }
	//尝试获取锁
	//返回负数：表示当前线程获取失败
	//返回零值：表示当前线程获取成功, 但是后继线程不能再获取了
	//返回正数：表示当前线程获取成功, 并且后继线程同样可以获取成功
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
	//尝试释放锁
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {//自旋减一
			//获取同步状态
            int c = getState();
            if (c == 0)//如果同步状态为0, 则不能再释放了
                return false;
            int nextc = c-1;//否则的话就将同步状态减1
            if (compareAndSetState(c, nextc))//使用CAS方式更新同步状态
                return nextc == 0;//nextc 为零时才返回真
        }
    }
}

```

CountDownLatch内部使用Sync继承AQS构造函数很简单地传递计数值给Sync，并且设置了state。这个state对于CountDownLatch来说，则表示计数值的大小。

## 3.3 await如何阻塞线程？

```Java

public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())//判断是否被中断，中断就抛出异常
        throw new InterruptedException();
	//尝试去获取锁
    if (tryAcquireShared(arg) < 0)
		//如果获取失败则进人该方法
        doAcquireSharedInterruptibly(arg);
}

private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
	//将当前线程相关的节点将入链表尾部
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {//无限循环
            final Node p = node.predecessor();//获得它的前驱节点
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {//唯一的退出条件，也就是await()方法返回的条件很重要！！
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) //线程由该函数来阻塞
                throw new InterruptedException();
        }
    } finally {
        if (failed)//如果失败或出现异常，取消该节点，以便唤醒后续节点
            cancelAcquire(node);
    }
}

```

调用await方法会阻塞当前线程的原理：

- 当线程调用await方法时其实是调用到了AQS的acquireSharedInterruptibly方法，该方法是以响应线程中断的方式来获取锁的。

- acquireSharedInterruptibly方法首先会去调用tryAcquireShared方法尝试获取锁。

- Sync里面重写的tryAcquireShared方法的逻辑，方法的实现逻辑很简单，就是判断当前同步状态是否为0，如果为0则返回1表明可以获取锁，否则返回-1表示不能获取锁。

- 如果tryAcquireShared方法返回1则线程能够不必等待而继续执行，如果返回-1那么后续就会去调用doAcquireSharedInterruptibly方法让线程进入到同步队列里面等待。

### 3.3.1 addWaiter  将线程变为节点添加到链表

```Java

private Node addWaiter(Node mode) {
	//首先new一个节点，该节点维持一个线程引用
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;//获取tail节点，tail是volatile型的
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {//利用CAS设置，允许失败，后面有补救措施
            pred.next = node;
            return node;
        }
    }
    enq(node);//设置失败，表明是第一个创建节点，或者是已经被别的线程修改过了会进入这里
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize 第一个创建时，节点为空
            if (compareAndSetHead(new Node()))
                tail = head;//初始化时 头尾节点相等
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {//注意这里，只有设置成功才会退出，所以该节点一定会被添加
                t.next = node;
                return t;
            }
        }
    }
}

```

## 3.4 countDown 如何唤醒阻塞线程？

```Java

public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
	//尝试去释放锁
    if (tryReleaseShared(arg)) {
		如果释放成功就唤醒其他线程
        doReleaseShared();
        return true;
    }
    return false;
}

//唤醒线程
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
				//头节点状态如果SIGNAL，则状态重置为0，并调用unparkSuccessor唤醒下个节点。
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
			//被唤醒的节点状态会重置成0，在下一次循环中被设置成PROPAGATE状态，代表状态要向后传播。
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}

```

countDown方法里面调用了releaseShared方法，该方法同样是AQS里面的方法。

releaseShared方法里面首先是调用tryReleaseShared方法尝试释放锁。

tryReleaseShared方法如果返回true表示释放成功，返回false表示释放失败，只有当将同步状态减1后该同步状态恰好为0时才会返回true，其他情况都是返回false。

么当tryReleaseShared返回true之后就会马上调用doReleaseShared方法去唤醒同步队列的所有线程。