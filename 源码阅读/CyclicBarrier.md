# 1. 简介

让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会
开门，所有被屏障拦截的线程才会继续运行。

CyclicBarrier是通过ReentrantLock(独占锁)和Condition来实现的。

```Java

public class CyclicBarrier {
	private static class Generation {
        boolean broken = false;
    }

	private final ReentrantLock lock = new ReentrantLock();
	private final Condition trip = lock.newCondition();
	private final int parties;//需要到达屏障的线程数
	private final Runnable barrierCommand;//全部线程到达屏障后要执行的指令
	private Generation generation = new Generation();//记录线程属于哪一代
	private int count;//处于等待状态的线程数
...

```

# 2. CyclicBarrier和CountDownLatch的区别

CountDownlatch的使用场景一般是一等多；CyclicBarrier的使用场景为多个线程互相等待。

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重
置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数
器，并让线程重新执行一次。

CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得Cyclic-Barrier
阻塞的线程数量。isBroken()方法用来了解阻塞的线程是否被中断。

# 3. 源码

## 3.1 构造函数

```Java

//调用 CyclicBarrier(int parties, Runnable barrierAction) 实现
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
	// parties表示“必须同时到达barrier的线程个数”
    this.parties = parties;
	// count表示“处在等待状态的线程个数”
    this.count = parties;
	// barrierCommand表示“parties个线程到达barrier时，会执行的动作”
    this.barrierCommand = barrierAction;
}

```

## 3.2 await

```Java

public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}

public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}

```
await() 和 await(long timeout, TimeUnit unit) 都是调用 dowait 实现。

dowait()的作用就是让当前线程阻塞，直到“有parties个线程到达barrier” 或 “当前线程被中断” 或 “超时”这3者之一发生，当前线程才继续执行。 

```Java

private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
	// 获取“独占锁(lock)”
    lock.lock();
    try {
		// 保存“当前的generation”,generation来代表每一轮的Cyclibarrier的运行状况
        final Generation g = generation;
		// 若“当前generation已损坏”，则抛出异常
        if (g.broken)
            throw new BrokenBarrierException();
		// 如果当前线程被中断，则通过breakBarrier()终止CyclicBarrier，唤醒CyclicBarrier中所有等待线程。
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
		// 将“count计数器”-1
        int index = --count;
		// 如果index=0，则意味着“有parties个线程到达barrier”
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
				// 如果barrierCommand不为null，则执行该动作
                if (command != null)
                    command.run();
                ranAction = true;
				// 唤醒所有等待线程，并更新generation
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 当前线程一直阻塞，直到“有parties个线程到达barrier” 或 “当前线程被中断” 或“超时”这3者之一发生,当前线程才继续执行。
		//在for(;;)循环中。timed是用来表示当前是不是“超时等待”线程。如果不是，则通过trip.await()进行等待；否则，调用awaitNanos()进行超时等待。
        for (;;) {
            try {
				// 如果不是“超时等待”，则调用awati()进行等待；否则，调用awaitNanos()进行等待
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
				// 如果等待过程中，线程被中断，则执行下面的函数
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    Thread.currentThread().interrupt();
                }
            }
			
			// 如果“当前generation已经损坏”，则抛出异常
            if (g.broken)
                throw new BrokenBarrierException();
			// 如果“generation已经换代”，则返回index
            if (g != generation)
                return index;
			// 如果是“超时等待”，并且时间已到，则通过breakBarrier()终止CyclicBarrier，唤醒CyclicBarrier中所有等待线程。
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}

```

## 3.3 其他

在CyclicBarrier中，同一批的线程属于同一代，即同一个Generation；CyclicBarrier中通过generation对象，记录属于哪一代。 当有parties个线程到达barrier，generation就会被更新换代。 

```Java

private static class Generation {
    boolean broken = false;
}

```

```Java

private void nextGeneration() {
    //调用signalAll()唤醒CyclicBarrier上所有的等待线程
    trip.signalAll();
    //重新初始化count
    count = parties;
	//更新generation
    generation = new Generation();
}

private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}

```