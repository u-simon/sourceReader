# ReentrantLock源码解析

synchronized是jvm级别的可重入锁的实现,而ReenytrantLock则是java代码实现的可重入锁, ReentrantLock主要是使用CAS+AQS队列实现的

CAS: compare and swap  比较并交换 Java中主要运用的底层的Unsafe 类来实现

AQS: AbstractQueuedSynchronizer

reentrantLock支持非公平锁和公平锁

## 非公平锁

```java
static final class NonfairSync extends Sync {
  private static final long serialVersionUID = 7316153563782823691L;

  /**
   * Performs lock.  Try immediate barge, backing up to normal
   * acquire on failure.
   */
  final void lock() {
    if (compareAndSetState(0, 1))
      setExclusiveOwnerThread(Thread.currentThread());
    else
      acquire(1);
  }

  protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
  }
}
```

## 公平锁

```java
static final class FairSync extends Sync {
  private static final long serialVersionUID = -3000897897090466540L;

  final void lock() {
    acquire(1);
  }
  /**
   * Fair version of tryAcquire.  Don't grant access unless
   * recursive call or no waiters or is first.
   */
  protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
      if (!hasQueuedPredecessors() &&
          compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
      }
    }
    else if (current == getExclusiveOwnerThread()) {
      int nextc = c + acquires;
      if (nextc < 0)
        throw new Error("Maximum lock count exceeded");
      setState(nextc);
      return true;
    }
    return false;
  }
}
```

## 构造方法

```java
public ReentrantLock() {
  // 默认使用非公平锁
  sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```

## lock

先从非公平锁的实现开始说起

```java
final void lock() {
   // 之所以成为非公平锁, 就是他不会去关心队列中是否有线程等待 刚开始就先争抢锁
    if (compareAndSetState(0, 1))
      // 争抢锁成功 则把锁的拥有者 设置为当前线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
      // 获取锁失败
        acquire(1);
}
```

公平锁的lock

```java
final void lock() {
  // 公平锁在获取锁的时候 不会先出尝试获取锁
  acquire(1);
}
```

### acquire 

```java
public final void acquire(int arg) {
  // 先尝试索取锁 获取失败则添加到队列中自旋
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

#### tryAcquire

非公平锁

```java
protected final boolean tryAcquire(int acquires) {
  return nonfairTryAcquire(acquires);
}
```

##### nonfairTryAcquire

```java
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  // 查看 state的值
  int c = getState();
  if (c == 0) {
    // 尝试获取锁
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    // 获取锁的线程和拥有锁的线程相同 可重入特性
    int nextc = c + acquires; // 重入的数量
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

公平锁

```java
protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    // 公平锁 tryAcquire 尝试获取锁和非公平锁的区别是 会先去查看自己是不是队列的第一个
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

##### hasQueuedPredecessors

```java
public final boolean hasQueuedPredecessors() {
  Node t = tail; // Read fields in reverse initialization order
  Node h = head;
  Node s;
  // 队列中已经存在等待的线程 并且自己不是head的下一个节点
  return h != t &&
    ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

#### addWaiter

```java
private Node addWaiter(Node mode) {
  // 创建新节点
  Node node = new Node(Thread.currentThread(), mode);
  // Try the fast path of enq; backup to full enq on failure
  Node pred = tail;
  // 如果之前队列中有线程, 则把线程新线程添加到队列尾
  if (pred != null) {
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  // 尾结点为空 说明队列还没有进行性初始化 需要先进行初始哈
  enq(node);
  return node;
}
```

##### enq

```java
// 初始化队列并添加新节点
private Node enq(final Node node) {
  for (;;) {
    Node t = tail;
    if (t == null) { // Must initialize
      // tail为空则初始化一个新的head 并且 tail执行head
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      node.prev = t;
      // tail不为空 则把新节点入队
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

#### acquireQueued

```java
// 入队的线程则自旋尝试获取锁
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true; // 标记线程是否获取锁
  try {
    // 标记线程是否被中断
    boolean interrupted = false;
    for (;;) {
      // 获取先驱节点
      final Node p = node.predecessor();
      // 如果先驱节点是head,即该节点称为第二个节点,则尝试获取锁
      if (p == head && tryAcquire(arg)) {
        // 获取成功则把当前节点设置为head
        setHead(node);
        // 原head节点出列
        p.next = null; // help GC
        // 获取锁成功
        failed = false;
        // 返回是否被中断过
        return interrupted;
      }
      // 获取锁失败之后 判断线程是否可以被挂起
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
        // 线程若被中断 设计标记为true
        interrupted = true;
    }
  } finally {
    if (failed)
      // 取消获取锁
      cancelAcquire(node);
  }
}
```

##### shouldParkAfterFailedAcquire

```java
// 线程入队能挂起的前提是 前置节点的状态为signal 含义是当前一个节点获取锁并且出队之后 唤醒当前线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  // 节点状态说明
  /** waitStatus value to indicate thread has cancelled */
  //static final int CANCELLED =  1;
  /** waitStatus value to indicate successor's thread needs unparking */
  //static final int SIGNAL    = -1;
  /** waitStatus value to indicate thread is waiting on condition */
  //static final int CONDITION = -2;
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    // 先驱节点的状态为signal 则返回true
    return true;
  if (ws > 0) {
    // 先驱节点的状态为cancelled 则从对列尾向前寻找第一个状态不为cancelled的节点
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 将先驱节点的状态这是为signal
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

##### parkAndCheckInterrupt

```java
// 挂起当前线程,返回线程的中断状态并重置
private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  return Thread.interrupted();
}
```

## tryLock

### tryLock()

```java
public boolean tryLock() {
  // nofairTryAcquire方法在前面介绍了
  return sync.nonfairTryAcquire(1);
}
```

- [tononfairTryAcquire](#nonfairTryAcquire)

### tryLock(timeout, unit)

```java
public boolean tryLock(long timeout, TimeUnit unit)
  throws InterruptedException {
  return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

#### tryAcquireNanos

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
  throws InterruptedException {
  if (Thread.interrupted())
    throw new InterruptedException();
  // 尝试获取锁 或者 有超时的获取
  return tryAcquire(arg) ||
    doAcquireNanos(arg, nanosTimeout);
}
```

- [toTryAcquire](#tryAcquire)

##### doAcquireNanos

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
  throws InterruptedException {
  if (nanosTimeout <= 0L)
    return false;
  // 计算过期时间
  final long deadline = System.nanoTime() + nanosTimeout;
  // 添加到队列中
  final Node node = addWaiter(Node.EXCLUSIVE);
  boolean failed = true;
  try {
    for (;;) {
      final Node p = node.predecessor();
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return true;
      }
      nanosTimeout = deadline - System.nanoTime();
      // 如果到规定时间都没获取锁则放弃
      if (nanosTimeout <= 0L)
        return false;
      // 判断是不是可挂起
      if (shouldParkAfterFailedAcquire(p, node) &&
          nanosTimeout > spinForTimeoutThreshold)
        // 带超时的挂起
        LockSupport.parkNanos(this, nanosTimeout);
      if (Thread.interrupted())
        throw new InterruptedException();
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

## unlock

```java
public void unlock() {
  sync.release(1);
}
```

### release

```java
public final boolean release(int arg) {
  // 尝试释放锁
  if (tryRelease(arg)) {
    Node h = head;
    // 判断head的waitStatus
    if (h != null && h.waitStatus != 0)
      //唤醒后面的节点
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

#### tryRelease

```java
protected final boolean tryRelease(int releases) {
  int c = getState() - releases;
  // 释放锁的线程不是当前拥有锁的线程则抛出IllegalMonitorStateException异常
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  boolean free = false;
  if (c == 0) {
    // 释放锁成功
    free = true;
    // 抹除所得拥有线程
    setExclusiveOwnerThread(null);
  }
  // 说明线程多次获取锁 需要多次 unlock
  setState(c);
  return free;
}
```

#### unparkSuccessor

```java
private void unparkSuccessor(Node node) {
  int ws = node.waitStatus;
  if (ws < 0)
    // head的节点小于0 先设置为0
    compareAndSetWaitStatus(node, ws, 0);
  
  // 获取head的下一个节点
  Node s = node.next;
  if (s == null || s.waitStatus > 0) {
    // 说明当前节点的状态为cancelled
    s = null;
    // 从根节点开始寻找第一个不为cancelled状态的节点
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  if (s != null)
    // 唤醒节点
    LockSupport.unpark(s.thread);
}
```

## newCondition

```java
final ConditionObject newCondition() {
  // 创建一个ConditionObject对象
  return new ConditionObject();
}
```

### await

```java
// AQS中await的实现
public final void await() throws InterruptedException {
  // 线程中断则抛出InterceptedException
  if (Thread.interrupted())
    throw new InterruptedException();
  // 将线程添加到等待队列
  Node node = addConditionWaiter();
  // 释放线程持有的锁
  int savedState = fullyRelease(node);
  int interruptMode = 0;
  // 判断线程是否在同步队列
  while (!isOnSyncQueue(node)) {
    // 挂起线程
    LockSupport.park(this);
    // 检查中断状态
    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
      break;
  }
  // 已经在同步队列则自旋获取锁
  if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
     // 线程被中断但是线程没有退出等待 则重新中断
    interruptMode = REINTERRUPT;
  if (node.nextWaiter != null) // clean up if cancelled
    // 线程下一个节点不为空则清理清理队列
    unlinkCancelledWaiters();
  if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
}
```

#### addConditionWaiter

```java
private Node addConditionWaiter() {
  Node t = lastWaiter;
  // lastWaiter 取消了 则清理队列
  if (t != null && t.waitStatus != Node.CONDITION) {
    unlinkCancelledWaiters();
    t = lastWaiter;
  }
  // node的waitStatus 为 condition
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  if (t == null)
    // 队列尾空则添加到firstWaiter
    firstWaiter = node;
  else
    // 队列不为空则添加到对列尾
    t.nextWaiter = node;
  // 对列尾重新赋值
  lastWaiter = node;
  return node;
}
```

#### fullyRelease

```java
final int fullyRelease(Node node) {
  boolean failed = true;
  try {
    int savedState = getState();
    // 全部释放 包括重入次数
    if (release(savedState)) {
      // 释放成功
      failed = false;
      return savedState;
    } else {
      // 失败则抛出异常
      throw new IllegalMonitorStateException();
    }
  } finally {
    if (failed)
      node.waitStatus = Node.CANCELLED;
  }
}
```

#### isOnSyncQueue

```java
final boolean isOnSyncQueue(Node node) {
  if (node.waitStatus == Node.CONDITION || node.prev == null)
    // 节点的状态为condition 或者前置节点为空 则不存在队列中
    return false;
  if (node.next != null) 
    // 节点存在next节点 则说明在队列中
    return true;
 // 从尾部开始查找节点
  return findNodeFromTail(node);
}
```

##### findNodeFromTail

```java
/**从尾部开始寻找节点**/
private boolean findNodeFromTail(Node node) {
  Node t = tail;
  for (;;) {
    if (t == node)
      return true;
    if (t == null)
      return false;
    t = t.prev;
  }
}
```

#### checkInterruptWhileWaiting

```java
private int checkInterruptWhileWaiting(Node node) {
  // 判断线程是否终端并抹除状态
  return Thread.interrupted() ?
    // 转换线程的状态
    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
  0;
}
```

##### transferAfterCancelledWait

```java
final boolean transferAfterCancelledWait(Node node) {
  if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
    // 中断退出
    enq(node);
    return true;
  }
  while (!isOnSyncQueue(node))
    Thread.yield();
  return false;
}
```

- [toAcquireQueued](#acquireQueued)

#### unlinkCancelledWaiters

```java
private void unlinkCancelledWaiters() {
  Node t = firstWaiter;
  Node trail = null;
  // 遍历队列 清理那些取消的节点
  while (t != null) {
    Node next = t.nextWaiter;
    if (t.waitStatus != Node.CONDITION) {
      t.nextWaiter = null;
      if (trail == null)
        firstWaiter = next;
      else
        trail.nextWaiter = next;
      if (next == null)
        lastWaiter = trail;
    }
    else
      trail = t;
    t = next;
  }
}
```

#### reportInterruptAfterWait

```java
private void reportInterruptAfterWait(int interruptMode)
  throws InterruptedException {
  if (interruptMode == THROW_IE)
    // 线程被中断抛出中断异常
    throw new InterruptedException();
  else if (interruptMode == REINTERRUPT)
    selfInterrupt();
}
```

### signal

```java
public final void signal() {
  //判断当前线程是否是线程的的拥有者
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  if (first != null)
    // 唤醒第一个等待者
    doSignal(first);
}
```

#### doSignal

```java
private void doSignal(Node first) {
  do {
    // 设置first的next节点为firstWaiter
    if ( (firstWaiter = first.nextWaiter) == null)
      lastWaiter = null;
    first.nextWaiter = null;
  } while (!transferForSignal(first) &&
           (first = firstWaiter) != null);
}
```

##### transferForSignal

```java
final boolean transferForSignal(Node node) {
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
    // 该节点可能被取消
    return false;
  // 从条件队列添加到AQS的同步队列
  Node p = enq(node);
  int ws = p.waitStatus;
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
    // 前节点被取消 唤醒线程
    LockSupport.unpark(node.thread);
  return true;
}
```

### signalAll

```java
public final void signalAll() {
  if (!isHeldExclusively())
    throw new IllegalMonitorStateException();
  Node first = firstWaiter;
  if (first != null)
    doSignalAll(first);
}
```

#### doSignalAll

```java
private void doSignalAll(Node first) {
  lastWaiter = firstWaiter = null;
  // 循环遍历唤醒
  do {
    Node next = first.nextWaiter;
    first.nextWaiter = null;
    transferForSignal(first);
    first = next;
  } while (first != null);
}
```

