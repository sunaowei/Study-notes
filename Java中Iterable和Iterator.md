## Java中的Iterable 和 Iterator

> JDK版本：jdk1.8.0_121,本文中所有代码的编译和运行环境均依赖该版本的jdk和对应版本的jre

### 1.什么是Iterable

#### 1.1简介

Iterable是Java集合中的一个顶级接口，是*Collection*的父类，用于进行集合中元素的迭代。*Iterable*的方法定义如下：

```java
package java.lang;

import ...

public interface Iterable<T> {

    Iterator<T> iterator();

    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}

```

这里面包含三个方法，其中*forEach*和*spliterator*这两个Java 1.8新增的**default**方法，关于什么是**default**方法，参考[这里](http://www.importnew.com/7302.html),(PS:Java为了向下兼容也是煞费苦心啊:dog:)

#### 1.2 iterator()

*iterator()*方法是Java1.5新增的特性，用于生成*Iterator*迭代器，这里该方法只负责生成*Iterator*迭代器，并不包含任何迭代器的状态，例如“当前元素”等。

而*Iterator*迭代器是作为一个独立的接口存在的。

该遍历是**顺序遍历**

#### 1.3 forEach()

forEach 很熟悉 用来支持lambda表达式的，例如:

```java
list.forEach(item->{
    doSomething...;
})
```

我们可以看到 在*Iterable*接口中默认给出的实现是

```java
for (T t : this) {
    action.accept(t);
}
```

但是这个方法在子类中是可以被覆盖的，很多集合都对该方法进行了重写操作，以适应不同集合的特性。

#### 1.4 spliterator()

该接口是Java为了**并行**遍历数据源中的元素而设计的迭代器

该方法类似于*iterator()*这个方法，区别在于*iterator()*是顺序遍历，而*spliterator()*是并行遍历，具体可以参考这里[Java8里面的java.util.Spliterator接口有什么用？](https://segmentfault.com/q/1010000007087438)

------

### 2.什么是Iterator

#### 2.1 简介

*Iterator*是**具有迭代状态**的对象，即从该类的方法中我们可以获取当前迭代的元素，下一个元素等操作。它可以把访问逻辑从不同类型的集合类中抽象出来，从而避免向客户端暴露集合的内部结构。*Iterator*的定义如下：

```java
package java.util;

import java.util.function.Consumer;

public interface Iterator<E> {

    boolean hasNext();

    E next();

    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

#### 2.2 hasNext()

接口解释：用于判断迭代器(*iteration*)中是否还有更多的元素(elements*)。如果迭代器中还有更多的元素则返回*true。

#### 2.3 next()

接口解释：用于获取迭代器中的下一个元素(*next element*)，这里如果下一个元素不存在会抛出一个*NoSuchElementException*的异常，所以这个方法通常结合*hasNext()*一起使用。

#### 2.4 remove():star::star:

接口解释：用于从集合中移除迭代器返回的最后一个元素。**这个方法只能在*next()*方法之后进行调用*一次***

这个方法会抛出两个异常

* *UnsupportedOperationException:*表示当前迭代器不支持移除操作。
* *IllegalStateException*:表示*next()*方法尚未执行 或 *next()*方法后已经执行了*remove()*方法。


这里有两个例子，如何在集合中移除一个元素的操作。

**错误示范**:x::

```java
List<String> list = Lists.newArrayList("A","B","C","D");
for (String s : list) {
    if (Objects.equals("A",s)){
        list.remove(s);
    }
}
```

这里会抛出一个异常`java.util.ConcurrentModificationException`，表示 方法检测到对象的并发修改，但是这种修改并不被允许。

产生的原因就是迭代器是依赖于集合而存在，在判断成功后，集合的中移除了该元素，而迭代器却不知道，所以就报错了，这个错叫并发修改异常。

**正确操作✅:**

```java
List<String> list = Lists.newArrayList("A", "B", "C", "D");
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String str = it.next();
    if (Objects.equals("A", str)) {
        it.remove();
    }
}
```

在Java8中上面的代码可以用下面两行来完成：

```java
List<String> list = Lists.newArrayList("A", "B", "C", "D");
list.removeIf(str -> Objects.equals("A", str));
```

这两种方式都是可以正确的完成我们所需要的操作的。

#### 2.5 forEachRemaining()

这个方法是1.8新增的方法，对每个剩余的元素执行给定的操作，直到所有元素都执行完成或者抛出异常。

如果传入的操作(*Consumer<? super E> action*)为空的话，该方法会抛出一个*NullPointerException*

### 3.Iterator的实现 

我们可以看到*Iterator*提供了最简单的向后遍历的操作接口，如果我们需要执行向前遍历呢？或者执行某些特殊的遍历方式，那么这个时候就要看不同集合对*Iterator*的实现了，不同集合的对于*Iterator*的实现基本都是通过内部类来做的实现。

#### 3.1 ArrayList中Iterator的实现

##### 3.1.1 Itr

在**ArrayList**中，*iterator()*方法被返回了一个**Itr**的对象。

```java
// ArrArrayList.java
public Iterator<E> iterator() {
    return new Itr();
}
```

这个Itr对象就是对**ArrayList**对**Iterator**的实现。

```java
private class Itr implements Iterator<E> {
    // 被返回的节点的索引
    int cursor;       // index of next element to return、
    // 最后一个被返回的节点的索引
    int lastRet = -1; // index of last element returned; -1 if no such
    // expectedModCount 期望的被修改的次数，
    // modCount 被修改的次数,这个下面会用得到。定义modCount的注释翻译可以参考[这里](https://blog.csdn.net/qq_27093465/article/details/53116250)
    int expectedModCount = modCount;

    /**
     * 判断是否还有下一个元素,如果当前游标所在位置不等于集合的大小的时候都会返回true.
     */
    public boolean hasNext() {
        return cursor != size;
    }

    /**
     * next操作
     */
    public E next() {
	   // 校验是否被非法修改
        checkForComodification();
        int i = cursor;
        // 当前游标大于等于集合的大小的时候 抛出异常
        if (i >= size)
            throw new NoSuchElementException();
        // 存储ArrayList的数组缓冲区的大小
        Object[] elementData = ArrayList.this.elementData;
        // 如果游标大于等于数组缓冲区的大小则抛出异常
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        // 游标+1
        cursor = i + 1;
        // 获取对应下表的内容，进行强转， 并将当前访问的节点记录到lastRet中，
        return (E) elementData[lastRet = i];
    }
	
    /**
     * 移除操作
     */
    public void remove() {
        // lastRet小于0，即表示没有执行next操作，也就是不存在当前元素。
        if (lastRet < 0)
            throw new IllegalStateException();
	   // 校验是否被非法修改
        checkForComodification();

        try {
            // 将最后一次访问的节点移除
            ArrayList.this.remove(lastRet);
            // 更新 游标，😈我们直接执行list中的remove方法就是因为没有更新游标和expectedModCount，所以无法通过校验。
            cursor = lastRet;
            // 更改最后一次访问的索引为-1，此处因为最后一次访问的元素已经不存在了，所以更新为-1，同时也可以避免再次调用remove方法。
            lastRet = -1;
            // 更新修改次数
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            // 捕获remove过程中出现的数组下标越界异常，更改为非法修改异常。
            throw new ConcurrentModificationException();
        }
    }

    @Override
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }
	// final 防止这个方法被非法修改
    final void checkForComodification() {
        // 校验修改次数和期望的修改次数是否相同，remove是一个典型的例子。Itr中的remove方法更新了expectedModCount，而list中remove方法并没有，所以我们可以使用Iterator进行遍历删除。
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}

```

##### 3.1.1 ListItr

在ArrayList中，还有一个*listIterator()*方法被返回了一个**ListIterator**的对象。那么这个对象是干啥的呢？

```java
/**
 * 从指定位置开始返回一个迭代器，如果位置超过了List的大小，则抛出IndexOutOfBoundsException
 */
public ListIterator<E> listIterator(int index) {
    if (index < 0 || index > size)
        throw new IndexOutOfBoundsException("Index: "+index);
    return new ListItr(index);
}
/**
 * 返回一个从0开始的迭代器
 */
public ListIterator<E> listIterator() {
    return new ListItr(0);
}
```

这个方法在**Itr**基础上提供了一些List集合特有的遍历方法。

```java
// 继承了Itr，实现了ListIterator接口，ListIterator接口中定义了List集合一些特有的遍历方法
private class ListItr extends Itr implements ListIterator<E> {
    // 构造方法，可以指定迭代器的起始位置，即cursor的位置
    ListItr(int index) {
        super();
        cursor = index;
    }
	
    // 是否有前一个元素
    public boolean hasPrevious() {
        // 判断游标是否是在最前面
        return cursor != 0;
    }
	
    // 后一个元素的位置
    public int nextIndex() {
        return cursor;
    }

    // 前一个元素的位置
    public int previousIndex() {
        return cursor - 1;
    }

    // 前一个元素
    public E previous() {
        // 校验是否被非法修改(Itr中的方法)
        checkForComodification();
        // 确定前一个元素的位置
        int i = cursor - 1;
        // 前一个元素如果小于0则表示当前是起始节点，上一个节点不存在。
        if (i < 0)
            throw new NoSuchElementException();
      	// 剩下的和next()同理
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }
	
    // 替换当前元素
    public void set(E e) {
        // 判断当前元素是否存在，同理的类似于remove()
        if (lastRet < 0)
            throw new IllegalStateException();
        // 校验是否被非法修改
        checkForComodification();

        try {
            // 修改缓冲区数组的值
            ArrayList.this.set(lastRet, e);
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    // 增加一个值(原理同remove())
    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```

### 4.迭代器模式

**实现了Iterable接口的类是可迭代的；实现了Iterator接口的类是一个迭代器。**

这里算是一个**迭代器模式**，即提供一种方法访问一个容器对象中各个元素，而又不暴露该对象的内部细节。

#### 4.1 类图

![迭代器类图](http://orw70g1os.bkt.clouddn.com/1524325872.png)

#### 4.2迭代器模式的结构

* 抽象容器(Aggregate):通常是一个接口，提供iterator()方法，比如说Java中的Iterator接口，Collection接口，Set接口等。

* 具体容器(ConcreteAggregate):抽象容器的具体实现类，比如说**ArrayList**，**LinkList**，**HashSet**等。

* 抽象迭代器(Iterator):定义遍历元素所需要的方法，比如说：*next()*，*hasNext()*，*remove()*等方法。

* 迭代器实现(ConcreteIterator):实现迭代器接口中定义的方法，完成集合的迭代。