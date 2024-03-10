# Java 8 ArrayList源码

> 原文引用 - [Cloud Li](http://blog.cloudli.top/) - [Java ArrayList](http://blog.cloudli.top/posts/Java-ArrayList/)

ArrayList 继承于 AbstractList ，实现了 List、RandomAccess、Cloneable、Serializable 接口。
ArrayList 的底层数据结构是数组，元素超出容量时会进行扩容操作。
## ArrayList 中的属性
```java
private static final int DEFAULT_CAPACITY = 10;
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
transient Object[] elementData;
private int size;
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
protected transient int modCount = 0;
```

- `DEFAULT_CAPACITY`：默认容量为 10。
- `EMPTY_ELEMENTDATA`：通过构造方法指定的容量为 0 时，使用该空数组。
- `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`：调用无参构造方法时，使用该数组。
- `elementData`：保存添加的元素，在 ArrayList 被序列化时，该属性不会被序列化。
- `size`：ArrayList 保存的元素个数。
- `MAX_ARRAY_SIZE`：ArrayList 能够容纳的最大长度，2^31 - 1 - 8。
- `modCount`:  这个列表被结构修改的次数。
## ArrayList 的构造方法

- `ArrayList()`：构造一个容量为 10 的空列表。
- `ArrayList(int initialCapacity)`：构造一个指定容量的空列表。
- `ArrayList(Collection<? extends E> c)`：构造一个列表，将给定集合中的元素按照迭代器返回的顺序复制到列表中。

使用无参构造方法创建 ArrayList 时，`elementData` 的长度是 0，当第一次添加元素时，它会扩容到 `DEFAULT_CAPACITY` 。
使用指定容量构造 ArrayList 时，`elementData` 的长度为指定的长度。
### ArrayList()
```java
public ArrayList() {
	this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
### ArrayList(int initialCapacity)
```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```
### ArrayList(Collection<? extends E> c)
```java
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a;
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        // replace with empty array.
        elementData = EMPTY_ELEMENTDATA;
    }
}
```
集合转换为数组后，如果不是 Object[] 类型，会使用 Arrays.copyOf 方法返回一个新的 Object 数组。
## 元素操作的基本方法
| 方法名 | 说明 | 时间复杂度 |
| --- | --- | --- |
| E get(int index) | 返回指定索引的元素 | O(1) |
| E set(int index, E element) | 替换指定索引的元素，返回旧元素 | O(1) |
| boolean add(E e) | 向列表的末尾添加元素 | O(1) |
| void add(int index, E element) | 在指定位置插入元素 | O(N) |
| E remove(int index) | 删除指定位置的元素，返回旧元素 | O(N) |
| boolean remove(Object o) | 删除列表中与给定对象相同的元素 | O(N) |
| boolean contains(Object o) | 判断列表中是否包含指定元素 | O(N) |
| int indexOf(Object o) | 返回指定元素在列表中的索引 | O(N) |

## 对列表操作的方法
### clear()
清空列表，将所有元素的引用指向 null，等待垃圾回收器回收，此操作不减小数组容量。
```java
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```
### subList(int fromIndex, int toIndex)
返回列表的指定区域（子列表），对返回的列表做修改会影响整个列表。
具体看 java.util.ArrayList.SubList 源码
```java
public List<E> subList(int fromIndex, int toIndex) {
    subListRangeCheck(fromIndex, toIndex, size);
    return new SubList(this, 0, fromIndex, toIndex);
}
```
## 迭代器方法
### iterator()
返回一个迭代器，如果在迭代器遍历过程中调用了列表的 add，remove 等方法，会抛出 `ConcurrentModificationException` 。
```java
public Iterator<E> iterator() {
	return new Itr();
}
```
### listIterator()
返回一个迭代器，该迭代器可以向前后两个方向遍历元素。ListIter 继承于 Iter 。
```java
public ListIterator<E> listIterator() {
	return new ListItr(0);
}
```
### listIterator(int index)
返回一个从指定位置开始的迭代器。
```java
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}
```
## 扩容方法
ArrayList 有两个与扩容有关的方法：

- `void grow(int minCapacity)`
- `int hugeCapacity(int minCapacity)`
### grow(int minCapacity)
```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```
minCapacity 为添加新元素后数组的长度（此时还未添加）。
首先将新长度（newCapacity）设置为原来的 1.5 倍，然后计算新的长度是否能够达到 minCapacity，如果不能，则把新长度设置为 minCapacity 的大小。
如果新长度比 ArrayList 能够容纳的最大长度还要大，调用 `hugeCapacity` 方法来计算。否则，将列表长度调整为新长度。
### hugeCapacity(int minCapacity)
```java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
这里为什么要判断 minCapacity < 0 ？
假如列表中已经有 oldCapacity = 2^31 - 1 个元素，在添加新元素的时候，minCapacity 为原来的长度 + 1，溢出变为 minCapacity = -2^31，newCapacity 扩大到 1.5 倍也溢出为负数（绝对值 < minCapacity），此时 `grow` 方法中 newCapacity - minCapacity > 0（绝对值小的负数 - 绝对值大的负数），最后调用该方法时传递的 minCapacity 就是 -231，将抛出异常。
最后如果 minCapacity 大于 ArrayList 能容纳的最大长度，就返回整型最大值，否则返回 `MAX_ARRAY_SIZE`。
## 函数式接口方法
ArrayList 有 4 个参数为函数式接口的方法：
### 1. forEach(Consumer<? super E> action)
消费列表中的元素，可以对列表中的元素作出处理。以下代码打印列表中所有的偶数：
```java
List<Integer> list = Arrays.asList(1, 2, 3, 4);
list.forEach(e -> {
    if (e % 2 == 0) {
        System.out.printf("%d ", e);
    }
});
```
### 2. removeIf(Predicate<? super E> filter)
删除列表中满足给定条件的元素。以下代码删除列表中所有偶数元素：
```java
List<Integer> list = Arrays.asList(1, 2, 3, 4);
list.removeIf(e -> e % 2 == 0);
```
### 3. replaceAll(UnaryOperator operator)
替换列表中满足给定条件的元素。以下代码将所有偶数元素替换为 0 ：
```java
list.replaceAll(e -> e % 2 == 0 ? 0 : e);
```
### 4. sort(Comparator<? super E> c)
使用给定的规则对列表元素排序。如果元素的类型已经实现了 Comparable 接口，可以直接传递方法引用。
以下代码对列表进行升序排序：
```java
list.sort(Integer::compareTo);
```
逆序可以使用 Comparator.reverseOrder ：
```java
list.sort(Comparator.reverseOrder());
```
也可以在 lambda 表达式中写判断逻辑。
