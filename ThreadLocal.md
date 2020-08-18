# ThreadLocal源码解析

threadLocal 线程局部变量, 线程独有的副本,是每个线程独自拥有的并且其他线程不能访问,天然的线程安全

## set(value)

```java
public void set(T value) {
  // 先获取当前线程,因为是线程独立的 所以要获取调用者线程
  Thread t = Thread.currentThread();
  // 获取线程的ThreadLocalMap 因为一个线程可存在多个ThreadLocal对象,所以使用ThreadLocalMap来存储
  ThreadLocalMap map = getMap(t);
  if (map != null)
    // 把value添加到ThreadLocalMap中
    map.set(this, value);
  else
    // 先初始化map
    createMap(t, value);
}
```

### getMap(thread)

```java
ThreadLocalMap getMap(Thread t) {
  // ThreadLocal.ThreadLocalMap threadLocals
  return t.threadLocals;
}
```

### createMap(thread, value)

```java
void createMap(Thread t, T firstValue) {
  // new一个threadLocalMap对象
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

## get()

```java
public T get() {
  // 获取当前线程
  Thread t = Thread.currentThread();
  // 获取当前线程的ThreadLocalMap
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    // 调用map的getEntry方法获取
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  // 如果 map还没有初始化 则返回一个初始化值
  return setInitialValue();
}
```

### setInitialValue

```java
private T setInitialValue() {
  T value = initialValue();
  Thread t = Thread.currentThread();
  // 创建threadlocal 放一个初始值进去
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
  return value;
}
```



```java
// 默认值为null 可以被重写
protected T initialValue() {
  return null;
}
```

## remove()

```java
// 调用ThreadLocalMap的remove方法
public void remove() {
  ThreadLocalMap m = getMap(Thread.currentThread());
  if (m != null)
    m.remove(this);
}
```

## ThreadLocalMap

### 构造方法

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
  // 创建一个初始化容量为16的数组
  table = new Entry[INITIAL_CAPACITY];
  // 计算index
  int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
  // 创建entry实例 放入 index位置
  table[i] = new Entry(firstKey, firstValue);
  size = 1;
  // 设置threshold = len * 2 / 3  容量的 2/3
  setThreshold(INITIAL_CAPACITY);
}
```

#### Entry

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
  /** The value associated with this ThreadLocal. */
  Object value;
  // entry是一个key为ThreadLocal的对象 并且他的key是一个弱引用 但是value是一个强引用 所以这就可能会导致内存泄露
  // 弱引用的特点是 gc的时候会被回收
  Entry(ThreadLocal<?> k, Object v) {
    super(k);
    value = v;
  }
}
```

### set(threadloca, value)

```java
private void set(ThreadLocal<?> key, Object value) {
  Entry[] tab = table;
  int len = tab.length;
  // 先计算插入的index
  int i = key.threadLocalHashCode & (len-1);

  // 从位置向后查找
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    // 获取key
    ThreadLocal<?> k = e.get();
    if (k == key) {
      // 存在则替换
      e.value = value;
      return;
    }

    if (k == null) {
      // 可能key被gc回收了 直接替换旧值
      replaceStaleEntry(key, value, i);
      return;
    }
  }

  // 不存在
  tab[i] = new Entry(key, value);
  int sz = ++size;
  // 清理失败并且 数量达到threshold(length 的 2/3)
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    // 扩容
    rehash();
}
```

#### replaceStaleEntry

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                               int staleSlot) {
  // 备份以便检查
  Entry[] tab = table;
  int len = tab.length;
  Entry e;
  // 记录当前失效的节点的下标
  int slotToExpunge = staleSlot;
  
  // prevIndex -> ((i - 1 >= 0) ? i - 1 : len - 1)
  for (int i = prevIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = prevIndex(i, len))
    // 从staleSlot位置向前遍历 i--
    // 这样的目的是为了把前面所有已经被垃圾回收(key被回收)的也释放掉
    
    // 记录prev第一个为空的entry 到staleSlot位置之间key过期最小的index
    if (e.get() == null)
      slotToExpunge = i;
 
  // 从staleSlot位置向后查找 i++
  // 这两个遍历就是为了在左边遇到的第一个entry为空的key到右边第一个为空的entry之间被回收的key全都清理掉
  for (int i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();  
    
    // 如果存在相同的key则进行替换 并且i位置和staleSlot位置进行交换 然后清理slotToExpunge到i之间的被回收的数据
    if (k == key) {
      // 存在相同的key
      e.value = value;
      // 交换i和staleSlot位置的元素
      // 这里交换的目的是为了不出现重复的key
     // 假设statleSolt = 5,i = 10 tab[i]=new Entry(15, 15) 假如此时set了一个(15, 20)计算的index = 5 因为 5位置的key已经被垃圾回收掉了 所以会(15, 20)会替换掉index=5位置的旧数据 但是其实数组中已经存在你一个key=15的数据了 此时数组中就存在重复的key
      tab[i] = tab[staleSlot];
      tab[staleSlot] = e;
      
      // i--的for循环没有找到key被回收的entry
      // 因为 staleSolt和i位置的数据已经互换 所以需要清理的位置是i而不是staleSlot
      if (slotToExpunge == staleSlot)
        slotToExpunge = i;
       /**
         * 在调用cleanSomeSlots进行启发式清理之前
         * 会先调用expungeStaleEntry方法从slotToExpunge到table下标所在为null的连续段进行一次清理
         * 返回值便是table[]为null的下标
         * 然后以该下标--len进行一次启发式清理
         * 最终里面的方法实际上还是调用了expungeStaleEntry
         * 可以看出expungeStaleEntry方法是ThreadLocal核心的清理函数
         */
      cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
      return;
    } 
    // 如果在i--遍历的时候没有找到被回收的key,那么我们应该把SlotToExpunge设置成i++遍历的第一个被回收key的index
    // 如果整个数组中存在相同的key则会把数据交换到staleSlot的位置
    // 所以不管怎样 staleSlot位置存放的都是有效数据 所以不需要清理
    if (k == null && slotToExpunge == staleSlot)
      slotToExpunge = i;
  }
  // key不存在则把value也值为空 help gc
  // 创建一个新的entry放到该位置
  tab[staleSlot].value = null;
  tab[staleSlot] = new Entry(key, value);
  // 在i--/i++的遍历中如果存在被回收的key则清理掉
  if (slotToExpunge != staleSlot)
    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

##### expungeStaleEntry(staleSlot)

```java
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;
  // value和entry都设置为null help gc
  tab[staleSlot].value = null;
  tab[staleSlot] = null;
  size--;

  Entry e;
  int i;
  for (i = nextIndex(staleSlot, len);
       (e = tab[i]) != null;
       i = nextIndex(i, len)) {
    // 从当前位置往后遍历查找 遇到entry=null停止
    ThreadLocal<?> k = e.get();
    if (k == null) {
      // 如果key已经被回收 则吧value和entry都设置为null help gc 防止内存泄露
      e.value = null;
      tab[i] = null;
      size--;
    } else {
      // 因为threadLocal解决hash冲突采用的是开放地址法,所以删除的元素可能是冲突中的一个,所以需要从当天位置向后寻找到冲突的元素将他前移
      int h = k.threadLocalHashCode & (len - 1);
      if (h != i) {
        tab[i] = null;
        while (tab[h] != null)
          h = nextIndex(h, len);
        tab[h] = e;
      }
    }
  }
  return i;
}
```

##### cleanSomeSlots(i , length)

```java
private boolean cleanSomeSlots(int i, int n) {
  boolean removed = false;
  Entry[] tab = table;
  int len = tab.length;
  do {
    // 从i位置向后寻找,查找被回收的key进行清理
    i = nextIndex(i, len);
    Entry e = tab[i];
    if (e != null && e.get() == null) {
      n = len;
      removed = true;
      i = expungeStaleEntry(i);
    }
  } while ( (n >>>= 1) != 0);
  return removed;
}
```

#### rehash()

```java
private void rehash() {
  // 先进行回收key的清理
  expungeStaleEntries();
  // 清理之后的size > thredShold 的 3/4 则进行扩容 因为theshold的大小为 length的2/3 所以 length * 2/3 * 3/4 = length/2 所以当table中的数据量达到 length/2时候就需要进行扩容
  if (size >= threshold - threshold / 4)
    resize();
}
```

##### expungeStaleEntries

```java
private void expungeStaleEntries() {
  Entry[] tab = table;
  int len = tab.length;
  // 清理tab中所有已经回收的key
  for (int j = 0; j < len; j++) {
    Entry e = tab[j];
    if (e != null && e.get() == null)
      expungeStaleEntry(j);
  }
}
```

##### resize

```java
private void resize() {
  Entry[] oldTab = table;
  int oldLen = oldTab.length;
  // 新table的长度是旧table的2倍
  int newLen = oldLen * 2;
  Entry[] newTab = new Entry[newLen];
  int count = 0;
  // 遍历旧table把所有的元素重新计算index放到新table中
  for (int j = 0; j < oldLen; ++j) {
    Entry e = oldTab[j];
    if (e != null) {
      ThreadLocal<?> k = e.get();
      if (k == null) {
        // 如果key被回收了 则把value置为null help gc
        e.value = null; // Help the GC
      } else {
        // 重新计算index 
        int h = k.threadLocalHashCode & (newLen - 1);
        while (newTab[h] != null)
         // 重复则查找下一个插入点
          h = nextIndex(h, newLen);
        newTab[h] = e;
        count++;
      }
    }
  }

  setThreshold(newLen);
  size = count;
  table = newTab;
}
```

### get()

```java
public T get() {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null) {
    ThreadLocalMap.Entry e = map.getEntry(this);
    if (e != null) {
      @SuppressWarnings("unchecked")
      T result = (T)e.value;
      return result;
    }
  }
  return setInitialValue();
}
```

### getEntry()

```java
private Entry getEntry(ThreadLocal<?> key) {
  // 计算index
  int i = key.threadLocalHashCode & (table.length - 1);
  Entry e = table[i];
  if (e != null && e.get() == key)
    // 当前位置的元素不为空并且key相等
    return e;
  else
    return getEntryAfterMiss(key, i, e);
}
```

#### getEntryAfterMiss

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
  Entry[] tab = table;
  int len = tab.length;
  // 从i位置向后查找相同的key 如果存在则返回 不存在则返回null
  while (e != null) {
    ThreadLocal<?> k = e.get();
    if (k == key)
      // key相等 则返回
      return e;
    if (k == null)
      // key已经被gc回收
      expungeStaleEntry(i);
    else
      // 计算下一个index
      i = nextIndex(i, len);
    e = tab[i];
  }
  return null;
}
```



### remove()

```java
private void remove(ThreadLocal<?> key) {
  Entry[] tab = table;
  int len = tab.length;
  // 计算index 
  int i = key.threadLocalHashCode & (len-1);
  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    // 从index位置开始向后寻找相同的key来进行清理
    if (e.get() == key) {
      // 清理掉数据
      e.clear();
      expungeStaleEntry(i);
      return;
    }
  }
}
```

