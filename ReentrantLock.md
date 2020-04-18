## ReentrantLock源码解析

synchronized是jvm级别的可重入锁的实现,而ReenytrantLock则是java代码实现的可重入锁, ReentrantLock主要是使用CAS+AQS队列实现的

CAS: compare and swap  比较并交换 Java中主要运用的底层的Unsafe 类来实现

AQS: AbstractQueuedSynchronizer

reentrantLock支持非公平锁和公平锁

### 非公平锁

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

### 公平锁

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

### 构造方法

```java
public ReentrantLock() {
  // 默认使用非公平锁
  sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```

### lock方法 

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

#### acquire 

```java
public final void acquire(int arg) {
  // 先尝试索取锁 获取失败则添加到队列中自旋
  if (!tryAcquire(arg) &&
      acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

##### tryAcquire

非公平锁

```java
protected final boolean tryAcquire(int acquires) {
  return nonfairTryAcquire(acquires);
}
```

###### nonfairTryAcquire

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

###### hasQueuedPredecessors

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

##### addWaiter

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

###### enq

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

##### accquireQueued

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

###### shouldParkAfterFailedAcquire

```java
// 线程入队能挂起的前提是 前置节点的状态为signal 含义是当前一个节点获取锁并且出队之后 唤醒当前线程
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
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

###### parkAndCheckInterrupt

```java
// 挂起当前线程,返回线程的中断状态并重置
private final boolean parkAndCheckInterrupt() {
  LockSupport.park(this);
  return Thread.interrupted();
}
```

#### 