# ArrayList

arrayList底层是使用的数组 Object[] 来实现的存储

## 构造方法

```java
public ArrayList() {
  // 默认创建一个空数组
  //  private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
  this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

public ArrayList(int initialCapacity) {
  if (initialCapacity > 0) {
    // 容量大于0 创建一个容量为initialCapacity的Object数组
    this.elementData = new Object[initialCapacity];
  } else if (initialCapacity == 0) {
    // 创建一个空数组
    // private static final Object[] EMPTY_ELEMENTDATA = {}
    this.elementData = EMPTY_ELEMENTDATA;
  } else {
    // 抛出异常
    throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
  }
}
```

## add

```java
public boolean add(E e) {
  // 判断数组是否需要扩容,造成扩容的情况 
  // 1.初始化的时候没有传初始化容量 则使用默认的初始化容量(10)进行扩容
  // 2.数组中的元素存满了则需要进行扩容
  ensureCapacityInternal(size + 1);  // Increments modCount!!
  // index位置赋值
  elementData[size++] = e;
  return true;
}
```

### ensureCapacityInternal

```java
private void ensureCapacityInternal(int minCapacity) {
  // 计算容量 并判断是否需要扩容
  ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

#### calculateCapacity

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
  if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    // 数组还没有进行初始化 DEFAULT_CAPACITY=10 则返回最大的
    return Math.max(DEFAULT_CAPACITY, minCapacity);
  }
  return minCapacity;
}
```

#### ensureExplicitCapacity

```java
private void ensureExplicitCapacity(int minCapacity) {
  modCount++;
	// 判断是否需要扩容
  // 容器中添加1个元素之后如果长度大于数组的长度则进行扩容
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
```

##### grow

```java
private void grow(int minCapacity) {
  int oldCapacity = elementData.length;
  // 1.5倍扩容
  int newCapacity = oldCapacity + (oldCapacity >> 1);
  if (newCapacity - minCapacity < 0)
    newCapacity = minCapacity;
  // MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8
  if (newCapacity - MAX_ARRAY_SIZE > 0)
    newCapacity = hugeCapacity(minCapacity);
  // 数组的copy 调用的 System.arraycopy
  elementData = Arrays.copyOf(elementData, newCapacity);
}
```

###### hugeCapacity

```java
private static int hugeCapacity(int minCapacity) {
  if (minCapacity < 0) // overflow
    throw new OutOfMemoryError();
  // 最大容量为 Integer.max_value
  return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

## get

```java
public E get(int index) {
  // 检查index是否越界
  rangeCheck(index);
  // 获取index位置的元素
  return elementData(index);
}
```

