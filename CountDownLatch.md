# CountDownLatch

举个生活中的例子：家里来客人了,主要要给客人沏茶水喝，这时候我们可以看一下这个过程我们的核心任务是给客人沏茶水,但是我们需要烧热水，洗杯子，然后再泡茶水，我们可以在烧水的同时刷水杯，但是泡茶这个最终的任务我们只能等到水烧开，杯子洗好之后我们才能继续，所以这个过程要是用程序要表达的话 我们主任务需要等待烧水任务和洗杯子任务完成之后再继续泡茶任务,这个等待我们可以使用CountDownLatch来实现，

CountDownLatch 的作用是：当一个线程需要另外一个或多个线程完成后，再开始执行。比如主线程要等待一个子线程完成环境相关配置的加载工作，主线程才继续执行，就可以利用 CountDownLatch 来实现

上面的过程我们用程序表达如下，可以运行看一下效果。

```java
public static void main(String[] args) throws InterruptedException {

  System.out.println("泡茶开始执行");
  CountDownLatch countDownLatch = new CountDownLatch(2);

  //
  new Worker(countDownLatch, "烧水", 10).start();
  new Worker(countDownLatch, "洗杯子", 2).start();

  countDownLatch.await();
  System.out.println("泡茶任务继续执行");
}

static class Worker extends Thread {
  CountDownLatch countDownLatch;

  String taskName;

  long waitTime;

  public Worker(CountDownLatch countDownLatch, String taskName, long waitTime) {
    this.countDownLatch = countDownLatch;
    this.taskName = taskName;
    this.waitTime = waitTime;
  }

  @Override
  public void run() {
    System.out.println("子线程执行" + taskName + "任务...");
    try {
      // 我们用sleep来模拟工作时间
      TimeUnit.MINUTES.sleep(waitTime);
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      countDownLatch.countDown();
    }
  }
}
```

我们看到了CountDownLatch的简单用法 下面我们开始介绍其实现原理，首先我们先看CountDownLatch的构造方法 我们看到传入了一个参数count 那么我们这个参数代表的是什么意思呢 根据我们上面提到的例子 我们可以简单理解为：我们需要完成的子任务的个数

我们先看一下它有哪些主要的方法

```java
public class CountDownLatch 
    private final Sync sync;

		// 构造方法
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

	// 等待方法 会阻塞当前主线程
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

  // 等待方法 会阻塞当前主线程多长时间
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

  // 释放数量的线程 我们可以简单地理解为 我们完成一个子任务就打一个标记任务数量就减少1个 
    public void countDown() {
        sync.releaseShared(1);
    }

// 获取锁数量 我们可以简单的理解为 获取当前需要完成的子任务的数量
    public long getCount() {
        return sync.getCount();
    }
}
```

下面我们就一个个的去探索每个方法,识别一下庐山真面目

首先我们先从构造方法说起

```java
public CountDownLatch(int count) {
 // 判断一下传入的Count是否为大于0
  if (count < 0) throw new IllegalArgumentException("count < 0");
  // new Sync对象
  this.sync = new Sync(count);
}
```

上面的代码我们看到CountDownLatch的构造函数中主要就是创建了一个Sync的对象并把count参数传给了Sync对象，那么这个Sync到底是何方神圣呢

```java
// 我们看到Sync继承了AbstractQueuedSynchronizer(抽象队列同步器)
private static final class Sync extends AbstractQueuedSynchronizer {
  private static final long serialVersionUID = 4982264981922014374L;

  Sync(int count) {
    // 设置State的值
    setState(count);
  }

  int getCount() {
    // 获取State值
    return getState();
  }
  // 省略...
}

private volatile int state;
protected final void setState(int newState) {
  state = newState;
}
```

我们看到Sync继承了AbstractQueuedSynchronizer(抽象队列同步器),不错他就是我们经常所说的AQS,关于它的实现原理我们在后面的文章进行更新，上面的代码中还看到 state是一个int类型的被volatile修饰的整数，如果小伙伴对volatile不了解可以自行查阅资料,或者等待后面的文章更新

我们看到在CountDownLatch的构造函数中创建了一个Sync对象并且给state进行了赋值初始化,那他们具体是怎么应用的呢？

在上面的例子中我们的泡茶线程在等待烧水线程和洗杯子线程完成任务时调用了await()方法,我们来一窥究竟 看看他到底是怎么实现等待的呢

```java
public void await() throws InterruptedException {
  // 这里调用了sync#acquireSharedInterruptibly我们看看他要获取什么
  sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
  throws InterruptedException {
  // 我们先判断线程是否被打断 因为这里我们的泡茶任务可能被中断 比如说突然停电了 我们也没必要在等待烧水完成了
  if (Thread.interrupted())
    throw new InterruptedException();
  
  // 如果没有被打断 则去判断任务是否执行完成
  if (tryAcquireShared(arg) < 0) 
    // 如果任务没有完成 则我们还应该等待子任务完成
    doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
  // 这里就是获取state的值 来判断子任务是否执行完成了
  return (getState() == 0) ? 1 : -1;
} 

// 子任务还没有完成 我们的主任务应该继续等待子任务完成
private void doAcquireSharedInterruptibly(int arg)
  throws InterruptedException {
  // 主线程等待 创建一个SHARE节点加入到队列尾部
  final Node node = addWaiter(Node.SHARED);
  boolean failed = true;
  try {
    // 自旋
    for (;;) {
      // 这里获取node的前一个节点
      final Node p = node.predecessor();
      // 判断node的前一个节点是不是头结点head
      if (p == head) {
        // 如果node的前置节点是头结点则再次判断子任务是否执行完成
        int r = tryAcquireShared(arg);
        if (r >= 0) {// 如果子任务已经完成
          setHeadAndPropagate(node, r);
          p.next = null; // help GC
          failed = false;
          return;
        }
      }
      // 如果当前节点的头结点不是head节点或者子任务还没有完成 则判断当前节点是否可以挂起
      if (shouldParkAfterFailedAcquire(p, node) &&
          // 挂起线程并且检查终端 LockSupport#park()
          parkAndCheckInterrupt())
        throw new InterruptedException();
    }
  } finally {
    if (failed) // 失败
      // 取消节点的登等待
      cancelAcquire(node);
  }
}

private void setHeadAndPropagate(Node node, int propagate) {
  
  Node h = head; // Record old head for check below
  // 设置head = node节点 并且设置node标记的线程为null 前置节点为null
  setHead(node);

  // propagate > 0 说明子任务执行完成了则应该唤醒阻塞的线程去继续执行
  if (propagate > 0 || h == null || h.waitStatus < 0 ||
      (h = head) == null || h.waitStatus < 0) {
    Node s = node.next;
      // 只有一个节点 或者是共享模式 释放所有等待线程 各自尝试抢占锁
    if (s == null || s.isShared())
      doReleaseShared();
  }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  // 节点状态说明
  //static final int CANCELLED =  1; 线程已经被取消 这种状态的节点会被忽略并移除队列
  //static final int SIGNAL    = -1; 表示当前线程已被挂起 并且后继节点可以尝试抢占锁
  //static final int CONDITION = -2; 线程正在等待某些条件
  //static final int PROPAGATE = -3; 共享模式下 无条件所有等待线程尝试抢占锁

  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    return true;
  // 如果节点被取消 节点向前查找没有取消获取的线程
  if (ws > 0) {
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 如果没有被取消则把waitStatue修改为-1
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

上面的整体流程为:

1.判断当前线程是否中断 中断则抛出终端异常

2.判断子任务是否完成如果子任务完成则不需要等待 继续向下执行 如果子任务没完成则把当前线程放到CLH队列中

3.自旋判断当前节点的前置节点是否为head节点如果是head则尝试判断子任务是否完成 如果子任务完成则唤醒CLH队列中的等待线程继续执行

4.如果未完成则判断当前线程是否可以挂起 如果可以则调用LockSupport#park方法挂起当前线程 等待子任务完成唤醒挂起的线程



在例子中看到子任务完成之后会调用countDown方法，它的方法里面到底做了什么操作下面我们来看一下

```java
public void countDown() {
  // 调用了 sync#releaseShare我们看一下他到底释放了什么
  sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
  // 首先我们调用tryReleaseShared 尝试释放什么呢 
  // 我们继续分析 tryReleaseShared(1)方法
  if (tryReleaseShared(arg)) { 
    // 如果所有的子任务都完成了 则唤醒CLH队列中阻塞的线程
    doReleaseShared();
    return true;
  }
  return false;
}

// 我们来具体分析一下tryReleaseShared方法
protected boolean tryReleaseShared(int releases) {
  // Decrement count; signal when transition to zero
  // 自旋
  for (;;) {
    // 获取state的值
    int c = getState();
    // 判断state的值是否为0 为0则返回false
    if (c == 0)
      return false;
    // 如果state的值不为0 我们则把state进行减1
    int nextc = c-1;
    // 然后通过cas重新给state赋值 其实这里的意思就是通过cas把state的值减少1
    if (compareAndSetState(c, nextc))
      // 如果成功则判断 state减1之后是否为0
      return nextc == 0;
  }
}

private void doReleaseShared() {
  for (;;) {
    Node h = head;
    // 如果头结点不为空 并且头结点和尾结点不相等
    if (h != null && h != tail) {
      // 如果队列中有两个或以上个node，那么检查局部变量h的状态：
     
    //如果状态为PROPAGATE，直接判断head是否变化。
      int ws = h.waitStatus;
      if (ws == Node.SIGNAL) { 
   
        // 如果节点状态为SIGNAL，则他的next节点也可以尝试被唤醒
        // 如果状态为SIGNAL，说明h的后继是需要被通知的 通过对CAS操作结果取反将compareAndSetWaitStatus(h,Node.SIGNAL,0)
        // 和unparkSuccessor(h)绑定在了一起。说明了只要head成功得从SIGNAL修改为0，那么head的后继的代表线程肯定会被唤醒了。
     
        if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
          continue;            // loop to recheck cases
        unparkSuccessor(h);
      }
      // 如果状态为0，说明h的后继所代表的线程已经被唤醒或即将被唤醒，并且这个中间状态即将消失要么由于acquire thread获取锁失再
      // 次设置head为SIGNAL并再次阻塞,要么由于acquire thread获取锁成功而将自己（head后继）设置为新head并且只要head后继不是
      // 队尾，那么新head肯定为SIGNAL,所以设置这种中间状态的head的status为PROPAGATE,让其status又变成负数,这样可能被被唤醒
      // 线程检测到。
      else if (ws == 0 &&
                // 将节点状态设置为PROPAGATE，表示要向下传播，依次唤醒
               !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
        continue;                // loop on failed CAS
    }
    if (h == head)                   // loop if head changed
      break;
  }
}
```

上述代码的流程为：

1.当前子任务完成则尝试将sate值减1 并且判断是否所有的子任务都完成

2.如果所有的子任务都完成则需要唤醒CLH队列中阻塞的线程继续执行后面的操作