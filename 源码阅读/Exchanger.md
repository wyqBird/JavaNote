# 1.简介

Exchanger（交换者）是一个用于线程间协作的工具类。

Exchanger用于进行线程间的数据交换。

它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。

这两个线程通过exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产出来的数据传递给对方。

# 2.应用场景

Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。

Exchanger也可以用于校对工作，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致，eg。

eg：

```Java

public class ExchangerTest {

    private static final Exchanger<String> exgr = new Exchanger<>();
    private static ExecutorService threadPool = Executors.newFixedThreadPool(2);

    public static void main(String[] args) {
        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String A = "银行流水A";//A录入银行流水数据
                    exgr.exchange(A);
                } catch (InterruptedException e) { }
            }
        });

        threadPool.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    String B = "银行流水B";//B录入银行流水数据
                    String A = exgr.exchange(B);
                    System.out.println("A和B数据是否一致：" + A.equals(B) + ", A录入的是：" + A + ", B录入的是：" + B);
                } catch (InterruptedException e){}
            }
        });

        threadPool.shutdown();
    }
}

```

如果两个线程有一个没有执行exchange()方法，则会一直等待，如果担心有特殊情况发生，避免一直等待，可以使用exchange（V x，longtimeout，TimeUnit unit）设置最大等待时长。

# 3.源码 exchange

Exchanger 有单槽位和多槽位之分，单个槽位在同一时刻只能用于两个线程交换数据，这样在竞争比较激烈的时候，会影响到性能，多个槽位就是多个线程可以同时进行两个的数据交换，彼此之间不受影响，这样可以很好的提高吞吐量。 

我暂时只看了单槽交换。

```Java

public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    if ((arena != null ||
         (v = slotExchange(item, false, 0L)) == null) &&
        ((Thread.interrupted() || // disambiguates null return
          (v = arenaExchange(item, false, 0L)) == null)))
        throw new InterruptedException();
    return (v == NULL_ITEM) ? null : (V)v;
}

//单槽交换
private final Object slotExchange(Object item, boolean timed, long ns) {
    Node p = participant.get();// 获取当前线程携带的Node
    Thread t = Thread.currentThread();// 当前线程
    if (t.isInterrupted()) // 保留中断状态，以便调用者可以重新检查，Thread.interrupted() 会清除中断状态标记
        return null;

    for (Node q;;) {
        if ((q = slot) != null) {// slot不为null, 说明已经有线程在这里等待了
            if (U.compareAndSwapObject(this, SLOT, q, null)) {// 将slot重新设置为null, CAS操作
                Object v = q.item;// 取出等待线程携带的数据
                q.match = item;// 将当前线程的携带的数据交给等待线程
                Thread w = q.parked;// 可能存在的等待线程（可能中断，不等了）
                if (w != null)
                    U.unpark(w);// 唤醒等待线程
                return v;// 返回结果，交易成功
            }
            // CPU的个数多于1个，并且bound为0时创建 arena，并将bound设置为SEQ大小
            if (NCPU > 1 && bound == 0 &&
                U.compareAndSwapInt(this, BOUND, 0, SEQ))
                arena = new Node[(FULL + 2) << ASHIFT];// 根据CPU的个数估计Node的数量
        }
        else if (arena != null)
            return null; // 如果slot为null， 但arena不为null， 则转而路由到arenaExchange方法
        else {// 最后一种情况，说明当前线程先到，则占用此slot
            p.item = item;// 将携带的数据卸下，等待别的线程来交易
            if (U.compareAndSwapObject(this, SLOT, null, p))// 将slot的设为当前线程携带的Node
                break;// 成功则跳出循环
            p.item = null;// 失败，将数据清除，继续循环
        }
    }

    // 当前线程等待被释放， spin -> yield -> block/cancel
    int h = p.hash;// 伪随机，用于自旋
    long end = timed ? System.nanoTime() + ns : 0L;// 如果timed为true，等待超时的时间点； 0表示没有设置超时
    int spins = (NCPU > 1) ? SPINS : 1;// 自旋次数
    Object v;
    while ((v = p.match) == null) {// 一直循环，直到有线程来交易
        if (spins > 0) {// 自旋，直至spins不大于0
            h ^= h << 1; h ^= h >>> 3; h ^= h << 10;// 伪随机算法， 目的是等h小于0（随机的）
            if (h == 0)// 初始值
                h = SPINS | (int)t.getId();
            else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                Thread.yield();// 等到h < 0, 而spins的低9位也为0（防止spins过大，CPU空转过久），让出CPU时间片，每一次等待有两次让出CPU的时机（SPINS >>> 1）
        }
        else if (slot != p)// 别的线程已经到来，正在准备数据，自旋等待一会儿，马上就好
            spins = SPINS;
		// 如果线程没被中断，且arena还没被创建，并且没有超时
        else if (!t.isInterrupted() && arena == null &&
                 (!timed || (ns = end - System.nanoTime()) > 0L)) {
            U.putObject(t, BLOCKER, this);// 设置当前线程将阻塞在当前对象上
            p.parked = t;// 挂在此结点上的阻塞着的线程
            if (slot == p)
                U.park(false, ns);// 阻塞， 等着被唤醒或中断
            p.parked = null;// 醒来后，解除与结点的联系
            U.putObject(t, BLOCKER, null);// 解除阻塞对象
        }
        else if (U.compareAndSwapObject(this, SLOT, p, null)) {// 超时或其他（取消），给其他线程腾出slot
            v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
            break;
        }
    }
	// 归位
    U.putOrderedObject(p, MATCH, null);
    p.item = null;
    p.hash = h;
    return v;
}

```

总结：

1. 检查slot是否为空（null），不为空，说明已经有线程在此等待，尝试占领该槽位，如果占领成功，与等待线程交换数据，并唤醒等待线程，交易结束，返回。

2. 如果占领槽位失败，创建arena，但要继续【步骤1】尝试抢占slot，直至slot为空，或者抢占成功，交易结束返回。

3. 如果slot为空，则判断arena是否为空，如果arena不为空，返回null，重新路由到arenaExchange方法

4. 如果arena为空，说明当前线程是先到达的，尝试占有slot，如果成功，将slot标记为自己占用，跳出循环，继续【步骤5】，如果失败，则继续【步骤1】

5 当前线程等待被释放，等待的顺序是先自旋（spin），不成功则让出CPU时间片（yield），最后还不行就阻塞（block），spin -> yield -> block

6. 如果超时（设置超时的话）或被中断，则退出循环。

7. 最后，重置数据，下次重用，返回结果，结束。