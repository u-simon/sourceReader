# HashMap源码解析

JDK8 对HashMap底层进行了优化 由原先的 Node数组 + 链表的结构 变成了Node数组 + 链表 + 红黑树 1.8也解决了死循环的情况

## Node类

```java
// Node 类 实现了 Entry接口
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  V value;
  Node<K,V> next;

  Node(int hash, K key, V value, Node<K,V> next) {
    this.hash = hash;
    this.key = key;
    this.value = value;
    this.next = next;
  }

  public final K getKey()        { return key; }
  public final V getValue()      { return value; }
  public final String toString() { return key + "=" + value; }

  public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
  }

  public final V setValue(V newValue) {
    V oldValue = value;
    value = newValue;
    return oldValue;
  }

  public final boolean equals(Object o) {
    if (o == this)
      return true;
    if (o instanceof Map.Entry) {
      Map.Entry<?,?> e = (Map.Entry<?,?>)o;
      if (Objects.equals(key, e.getKey()) &&
          Objects.equals(value, e.getValue()))
        return true;
    }
    return false;
  }
}
```

## 构造方法

```java
// 无参
public HashMap() {
  // 无参构造器只初始化了默认的加载因子 0.75
   this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
// 1个参数  初始化容量
public HashMap(int initialCapacity) {
  this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
// 2个参数 初始化容量+加载因子
public HashMap(int initialCapacity, float loadFactor) {
  if (initialCapacity < 0)
    throw new IllegalArgumentException("Illegal initial capacity: " +
                                       initialCapacity);
  if (initialCapacity > MAXIMUM_CAPACITY)
    // 容量最大 就是 1 << 30
    initialCapacity = MAXIMUM_CAPACITY;
  if (loadFactor <= 0 || Float.isNaN(loadFactor))
    throw new IllegalArgumentException("Illegal load factor: " +
                                       loadFactor);
  this.loadFactor = loadFactor;
  // 保证hashMap的容量是2^n
  this.threshold = tableSizeFor(initialCapacity);
}
```

### tableSizeFor方法

```java
//保证hashMap的容量为 2^n  目的是为了计算index的时候尽量的保证更分散 保证计算结果和length尽量的无关
static final int tableSizeFor(int cap) {
  // 先进行减1的目的是 方式 传入的cap本身就是2^n 导致结果扩大2倍
  int n = cap - 1;
  n |= n >>> 1;
  n |= n >>> 2;
  n |= n >>> 4;
  n |= n >>> 8;
  n |= n >>> 16;
  return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

## put方法

```java
public V put(K key, V value) {
  return putVal(hash(key), key, value, false, true);
}
```

### hash

```java
static final int hash(Object key) {
  int h;
  // key为null的时候 hash值为0 否则进行 hashCode的高低16位异或
  return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### putVal方法

```java
	final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
  Node<K,V>[] tab; Node<K,V> p; int n, i;
  if ((tab = table) == null || (n = tab.length) == 0)
    // hashMap在new的时候没有进行初始化 而是在第一次put的时候进行的延迟初始化
    n = (tab = resize()).length;
  if ((p = tab[i = (n - 1) & hash]) == null)
    // 计算的index位置如果没有元素则创建node存进去
    tab[i] = newNode(hash, key, value, null);
  else {
    // Hash冲突的情况
    Node<K,V> e; K k;
    if (p.hash == hash &&
        ((k = p.key) == key || (key != null && key.equals(k))))
      // hash值相等 key也相同 则进行覆盖
      e = p;
    else if (p instanceof TreeNode)
      // 链表已经转换成红黑树
      e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
    else {
      for (int binCount = 0; ; ++binCount) {
        // 遍历列表
        if ((e = p.next) == null) {
          // 链表中不存在这个key则创建新节点
          // 1.8 这里用的尾插法
          p.next = newNode(hash, key, value, null);
          // 判断链表长度 达到8的时候进行树化
          if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            // 树化
            treeifyBin(tab, hash);
          break;
        }
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          // 链表中存在这个key
          break;
        p = e;
      }
    }
    if (e != null) { // existing mapping for key
      // 新值替换旧值
      V oldValue = e.value;
      if (!onlyIfAbsent || oldValue == null)
        e.value = value;
      afterNodeAccess(e);
      // 返回旧值
      return oldValue;
    }
  }
  ++modCount;
  if (++size > threshold)
    // 判断进行扩容
    resize();
  afterNodeInsertion(evict);
  return null;
```

#### putTreeVal 

```java
// 建议先去了解下 红黑树的特性 在看关于红黑树的这一段
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
  Class<?> kc = null;
  boolean searched = false;
  // 从根节点开始找 适合插入新数据的位置
  TreeNode<K,V> root = (parent != null) ? root() : this;
  for (TreeNode<K,V> p = root;;) {
    int dir, ph; K pk;
    // 比较插入数据的hash值和当前节点的hash值 判断是往左节点还是往右节点
    if ((ph = p.hash) > h)
      // 左节点
      dir = -1;
    else if (ph < h)
      // 右节点
      dir = 1;
    else if ((pk = p.key) == k || (k != null && k.equals(pk)))
      // key存在 直接返回 以便替换
      return p;
    else if ((kc == null &&
              (kc = comparableClassFor(k)) == null) ||
             (dir = compareComparables(kc, k, pk)) == 0) {
       //进入当前的判断的原因
       //1、当前节点与待插入节点　key　不同,　hash 值相同
       //2、ｋ是不可比较的，即ｋ并未实现　comparable<K>　接口
             //(若 k 实现了comparable<K>　接口，comparableClassFor（k）返回的是ｋ的class,而不是　null）　
      //或者　compareComparables(kc, k, pk)　返回值为 0
      			//(pk 为空　或者　按照 k.compareTo(pk) 返回值为０，
      			//返回值为０可能是由于ｋ的compareTo 方法实现不当引起的，compareTo 判定相等，而上个 else if　中　equals 判定不等)
      if (!searched) {
        TreeNode<K,V> q, ch;
        searched = true;
        // 判断是否存在重复的key 存在则覆盖以value
        if (((ch = p.left) != null &&
             (q = ch.find(h, k, kc)) != null) ||
            ((ch = p.right) != null &&
             (q = ch.find(h, k, kc)) != null))
          return q;
      }
      // 既然k无法比较则自定义比较方式
      dir = tieBreakOrder(k, pk);
    }

    TreeNode<K,V> xp = p;
    // 根据 前面计算的dir的值来判断是left 还是right
    if ((p = (dir <= 0) ? p.left : p.right) == null) {
      Node<K,V> xpn = xp.next;
      TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
      if (dir <= 0)
        // dir<= 0  则为左孩子
        xp.left = x;
      else
        // dir > 0  则为右孩子
        xp.right = x;
      xp.next = x;
      x.parent = x.prev = xp;
      if (xpn != null)
        ((TreeNode<K,V>)xpn).prev = x;
      // 插入节点之后进行二叉树的平衡
      moveRootToFront(tab, balanceInsertion(root, x));
      return null;
    }
  }
}
```

##### tieBreakOrder

```java
static int tieBreakOrder(Object a, Object b) {
  int d;
  // System.identityHashCode实际上是使用内存地址来进行比较
  if (a == null || b == null ||
      (d = a.getClass().getName().
       compareTo(b.getClass().getName())) == 0)
    d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
         -1 : 1);
  return d;
}
```

#### treeifyBin

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
  int n, index; Node<K,V> e;
  // 如果数据总量还没达到64 则先进行扩容
  if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
    resize();
  else if ((e = tab[index = (n - 1) & hash]) != null) {
    TreeNode<K,V> hd = null, tl = null;
    // 先把单向链表转换成双向链表(tree node)
    do {
      TreeNode<K,V> p = replacementTreeNode(e, null);
      if (tl == null)
        hd = p;
      else {
        p.prev = tl;
        tl.next = p;
      }
      tl = p;
    } while ((e = e.next) != null);
    if ((tab[index] = hd) != null)
      // 树化操作
      hd.treeify(tab);
  }
}
```

##### treeify

```java
final void treeify(Node<K,V>[] tab) {
  TreeNode<K,V> root = null;
  // 从链表的头节点开始转换成树结构
  for (TreeNode<K,V> x = this, next; x != null; x = next) {
    next = (TreeNode<K,V>)x.next;
    x.left = x.right = null;
    if (root == null) {
      x.parent = null;
      // 红黑树 根节点为黑色
      x.red = false;
      root = x;
    }
    else {
      K k = x.key;
      int h = x.hash;
      Class<?> kc = null;
      // 和插入类似 从根节点开始寻找插入点
      for (TreeNode<K,V> p = root;;) {
        int dir, ph;
        K pk = p.key;
        if ((ph = p.hash) > h)
          dir = -1;
        else if (ph < h)
          dir = 1;
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0)
          dir = tieBreakOrder(k, pk);

        TreeNode<K,V> xp = p;
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
          x.parent = xp;
          if (dir <= 0)
            xp.left = x;
          else
            xp.right = x;
          root = balanceInsertion(root, x);
          break;
        }
      }
    }
  }
  // 进行平衡操作
  moveRootToFront(tab, root);
}
```

#### balanceInsertion()

```java
/**
 * 红黑树的特性
 * 1.节点是黑色或者是红色
 * 2.根节点是黑色
 * 3.每个叶子节点(NIL节点,空节点)是黑色
 * 4.每个红色叶子节点的两个子节点都是黑色(从每个叶子到根的所有路径上不能有连续的两个红色节点)
 * 5.从任一节点到其每一个叶子节点的路径上包含的黑色节点数量都相同
 */
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                            TreeNode<K,V> x) {
  // 默认当前插入的节点为红节点
  x.red = true;
  // xp:当前节点的父节点 xpp:当前节点的爷爷节点 xppl:当前节点的左叔叔节点 xppr:当前节点的右叔叔节点
  for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
    if ((xp = x.parent) == null) {// 当前节点的父节点为null 也就是x节点就是根节点
     // x节点置黑
      x.red = false;
      // 返回根节点
      return x;
    }
    // 父节点为黑节点 或者 爷爷节点为空
    else if (!xp.red || (xpp = xp.parent) == null)
      return root;
    if (xp == (xppl = xpp.left)) { // 父节点是爷爷节点的左孩子
      if ((xppr = xpp.right) != null && xppr.red) { // 爷爷节点的右孩子不为空 并且是红色
        // 无旋转的情况 x节点为红色 父节点为红色破坏特性4  则需要进行重新染色
        xppr.red = false;// 右叔叔节点置黑
        xp.red = false; //父节点置黑
        xpp.red = true; // 爷爷节点置红
        x = xpp; // x节点设置为爷爷节点 进行下一轮循环
      }
      else {// 爷爷节点的右孩子为空或者是黑色
        if (x == xp.right) {// 如果x是右孩子节点
          // 父节点进行左旋
          root = rotateLeft(root, x = xp);
          xpp = (xp = x.parent) == null ? null : xp.parent; // 获取爷爷节点
        }
        if (xp != null) { // 如果父节点不为空
          xp.red = false; // 置黑(其实就是把x节点置黑)
          if (xpp != null) { // 爷爷节点不为空
            xpp.red = true; // 置红
            root = rotateRight(root, xpp);// 爷爷节点右旋
          }
        }
      }
    }
    else { // 父节点是爷爷节点的右孩子节点
      if (xppl != null && xppl.red) { // 左叔叔节点不为空并且是红节点
        xppl.red = false;// 左叔叔节点置黑
        xp.red = false; // 父节点置黑
        xpp.red = true; // 爷爷节点置红
        x = xpp; // x节点设置为爷爷节点 ，进行下一轮循环
      }
      else {// 左叔叔节点为空或者为黑节点
        if (x == xp.left) { // x为父节点的左孩子节点
          // 父节点进行右旋
          root = rotateRight(root, x = xp);
          xpp = (xp = x.parent) == null ? null : xp.parent;// 获取爷爷节点
        }
        if (xp != null) { // 如果父节点不为空
          xp.red = false; // 置黑
          if (xpp != null) { // 爷爷节点不为空
            xpp.red = true; // 置红
            root = rotateLeft(root, xpp); // 爷爷节点进行左旋
          }
        }
      }
    }
  }
}
```

#### rotateLeft()

```java
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,
                                      TreeNode<K,V> p) {
  
  TreeNode<K,V> r, pp, rl; // r:左旋节点的右孩子节点 rl:左旋节点的右孩子节点的左孩子节点 pp:左旋节点的父节点
  if (p != null && (r = p.right) != null) { // 旋转的节点以及右孩子节点不为空
    
    if ((rl = p.right = r.left) != null) //  r的左节点赋值给左旋节点的右孩子节点
      rl.parent = p; // rl和左旋节点的父子关系
    
    if ((pp = r.parent = p.parent) == null)// r的父节点设置为p的父节点 将r升一级 此时父节点如果为null则说明是顶层节点，应该作为root标记为黑色
      (root = r).red = false;
    else if (pp.left == p) // 如果p是做孩子节点 则把r设置为p的父节点的左孩子节点
      pp.left = r;
    else // 要左旋的节点是个右孩子
      pp.right = r;
    
    r.left = p; // r提升为父节点 则把p设置为r的左节点
    p.parent = r; // r设置为p的父节点
    
  }
  return root;
}
```

#### rotateRight()

```java
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,
                                       TreeNode<K,V> p) {
  TreeNode<K,V> l, pp, lr; // l:右旋节点的左孩子节点 pp:右旋节点的父节点 lr : 右旋节点的左孩子节点的右孩子节点
  if (p != null && (l = p.left) != null) {// 右旋节点已经它的左孩子节点不为空
    
    if ((lr = p.left = l.right) != null)// l的右节点赋值给右旋节点p的左孩子节点
      lr.parent = p; // 设置lr的父节点为p
    
    if ((pp = l.parent = p.parent) == null) // l的父节点为p的父节点 将l节点提升一级此时父节点如果为null则说明是顶层节点，应该作为root标记为黑色 
      (root = l).red = false;
    else if (pp.right == p)// 如果p为父节点的右孩子节点 设把l设置为pp的右孩子节点 否则设置为左孩子节点
      pp.right = l;
    else    
      pp.left = l;  
    l.right = p; // l的右孩子节点为p 这里就是l和p换了个位置
    p.parent = l; // p的父节点为l
  }
  return root;
}
```

#### resize

```java
final Node<K,V>[] resize() {
  Node<K,V>[] oldTab = table;
  // HashMap的懒加载创建
  int oldCap = (oldTab == null) ? 0 : oldTab.length;
  int oldThr = threshold;
  int newCap, newThr = 0;
  if (oldCap > 0) {
    // HashMap的最大容量为 1 << 30
    if (oldCap >= MAXIMUM_CAPACITY) {
      threshold = Integer.MAX_VALUE;
      return oldTab;
    }
    else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
             oldCap >= DEFAULT_INITIAL_CAPACITY)
      // 两倍扩容
      newThr = oldThr << 1; // double threshold
  }
  else if (oldThr > 0) // initial capacity was placed in threshold
    newCap = oldThr;
  else {               // zero initial threshold signifies using defaults
   // 第一次初始化的时候 hashMap 容量为 16 threshold为 12
    newCap = DEFAULT_INITIAL_CAPACITY;
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
  }
  if (newThr == 0) {
    float ft = (float)newCap * loadFactor;
    //最大的threshold 1<<30
    newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
              (int)ft : Integer.MAX_VALUE);
  }
  threshold = newThr;
  @SuppressWarnings({"rawtypes","unchecked"})
  Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
  table = newTab;
  // 表中不为空的时候 则把数据迁移到扩容后的 hashMap中
  if (oldTab != null) {
    // 遍历旧HashMap中的数据
    for (int j = 0; j < oldCap; ++j) {
      Node<K,V> e;
      if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)
          // 此位置只有一个数据则重新hash计算新表的位置 插入
          newTab[e.hash & (newCap - 1)] = e;
        else if (e instanceof TreeNode)
          // 此处为红黑树 后面介绍
          ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        else { // preserve order
          // 此处为链表
          Node<K,V> loHead = null, loTail = null; // 插入新表的位置和旧表的index相同
          Node<K,V> hiHead = null, hiTail = null; // 插入新表的位置为 旧表的cap + 在旧表的index
          Node<K,V> next;
          do {
            next = e.next;
            // 这里有一个很有意思的优化 用node的hash值和旧表的容量进行&运算 
            // 如果为0 则表示 在新表的位置和旧表的数据一样
            // 如果结果为1 则表示 新表的位置为 旧表的cap + 在旧表的index
            if ((e.hash & oldCap) == 0) {
              if (loTail == null)
                loHead = e;
              else
                loTail.next = e;
              loTail = e;
            }
            else {
              if (hiTail == null)
                hiHead = e;
              else
                hiTail.next = e;
              hiTail = e;
            }
          } while ((e = next) != null);
          if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
          }
          if (hiTail != null) {
            hiTail.next = null;
            newTab[j + oldCap] = hiHead;
          }
        }
      }
    }
  }
  return newTab;
}
```

##### split

```java
// treeNode 的扩容实现
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
  // 最好能明白this的用法 不然可能看不懂可能不知道这个this代表的含义
  TreeNode<K,V> b = this;
  // Relink into lo and hi lists, preserving order
  TreeNode<K,V> loHead = null, loTail = null; // 插入新表的位置和旧表的index相同
  TreeNode<K,V> hiHead = null, hiTail = null; // 插入新表的位置为 旧表的cap + 在旧表的index
  int lc = 0, hc = 0;
  // 实现思想和链表的类似
  for (TreeNode<K,V> e = b, next; e != null; e = next) {
    next = (TreeNode<K,V>)e.next;
    e.next = null;
    // 和链表一样的优化
    if ((e.hash & bit) == 0) {
      if ((e.prev = loTail) == null)
        loHead = e;
      else
        loTail.next = e;
      loTail = e;
      ++lc;
    }
    else {
      if ((e.prev = hiTail) == null)
        hiHead = e;
      else
        hiTail.next = e;
      hiTail = e;
      ++hc;
    }
  }

  if (loHead != null) {
    // 这里判断插入新表数据的大小 是转化成链表 还是转换成红黑树
    if (lc <= UNTREEIFY_THRESHOLD)
      tab[index] = loHead.untreeify(map);
    else {
      tab[index] = loHead;
      if (hiHead != null) // (else is already treeified)
        loHead.treeify(tab);
    }
  }
  if (hiHead != null) {
     // 这里判断插入新表数据的大小 是转化成链表 还是转换成红黑树
    if (hc <= UNTREEIFY_THRESHOLD)
      tab[index + bit] = hiHead.untreeify(map);
    else {
      tab[index + bit] = hiHead;
      if (loHead != null)
        hiHead.treeify(tab);
    }
  }
}
```

##### untreeify

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
  Node<K,V> hd = null, tl = null;
  for (Node<K,V> q = this; q != null; q = q.next) {
   // 双向链表转换为单向链表
    Node<K,V> p = map.replacementNode(q, null);
    if (tl == null)
      hd = p;
    else
      tl.next = p;
    tl = p;
  }
  return hd;
}
```

## get

```java
public V get(Object key) {
  Node<K,V> e;
  // 实际上是调用的getNode方法
  return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

### getNode

```java
final Node<K,V> getNode(int hash, Object key) {
  Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (first = tab[(n - 1) & hash]) != null) {
    // table不为空并且table.length > 0 并且 index位置有元素
    if (first.hash == hash && // always check first node
        ((k = first.key) == key || (key != null && key.equals(k))))
      // 查看first位置的key是否相等 相等则返回firstNode
      return first;
    if ((e = first.next) != null) {
      if (first instanceof TreeNode)
        // 当前位置为红黑树
        return ((TreeNode<K,V>)first).getTreeNode(hash, key);
      // 当前位置为链表 则遍历链表查找节点
      do {
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
          return e;
      } while ((e = e.next) != null);
    }
  }
  return null;
}
```

#### getTreeNode

```java
final TreeNode<K,V> getTreeNode(int h, Object k) {
  return ((parent != null) ? root() : this).find(h, k, null);
}
```

#### find

```java
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
  TreeNode<K,V> p = this;
  do {
    int ph, dir; K pk;
    // pl为左节点 pr为右节点
    TreeNode<K,V> pl = p.left, pr = p.right, q;
    if ((ph = p.hash) > h)
      // 当前节点的hash大于查找的key的hash值 则查找左节点
      p = pl;
    else if (ph < h)
      // 查找右节点
      p = pr;
    else if ((pk = p.key) == k || (k != null && k.equals(pk)))
     // 当前节点的key和查找的key相同 返回当前节点
      return p;
    else if (pl == null)
      // 没有左节点
      p = pr;
    else if (pr == null)
      // 没有右节点
      p = pl;
    else if ((kc != null ||
              (kc = comparableClassFor(k)) != null) &&
             (dir = compareComparables(kc, k, pk)) != 0)
      p = (dir < 0) ? pl : pr;
    else if ((q = pr.find(h, k, kc)) != null)
      return q;
    else
      p = pl;
  } while (p != null);
  return null;
}
```

