# CopyOnWriteArrayList

**CopyOnWriteArrayList** (_写时复制数组_，简称 **COW** ) 是一个并发容器类，在某些情况下可用于替换同步 ArrayList 或 Vector ，以提供更优的并发性能。接下来按下面顺序介绍 COW。
1. **同步容器的问题**
2. **COW 读写原理**
3. **COW 使用时机**


## 同步容器的问题

`Vector` 和 `Collections.synchronized(ArrayList)` 都是线程安全的容器类，类中提供的方法都是线程安全的。
然而，对于许多「复合操作」，由于单独的一个线程安全的方法无法保证操作的「原子性」，使用者还是需要在使用容器的时候额外加锁。
常见的「复合操作」有：
1. 遍历（迭代）。
2. 条件运算，如若没有则添加操作。

由于同步容器无法保证复合操作的原子性，在多线程同时操作数组时会导致数据不一致，常见的是某个线程在迭代遍历（如 `for-each` ）中，另一个线程改变了数组的数量，会抛出 `ConcurrentModificationException` 异常，这种行为被称为「及时失败」( fail - fast )。我们看 Vector 迭代器的源码也能够清晰看到这点：

当我们使用 for-each 时，会 new 一个 Iterator ，创建了一个 Itr 实例。
```java

public synchronized Iterator<E> More ...iterator() {
        return new Itr();
}
```

Itr 是 Iterator 接口的具体实现类，也是 Vector 中的内部类，在 new 的时候记录当前容器中的元素数量 `expectedModCount = modCount` , 重点看一下 `checkForComodification()` 方法。

```java

private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            // Racy but within spec, since modifications are checked
            // within or after synchronization in next/previous
            return cursor != elementCount;
        }

        public E next() {
            synchronized (Vector.this) {
                checkForComodification();
                int i = cursor;
                if (i >= elementCount)
                    throw new NoSuchElementException();
                cursor = i + 1;
                return elementData(lastRet = i);
            }
        }

        public void remove() {
            if (lastRet == -1)
                throw new IllegalStateException();
            synchronized (Vector.this) {
                checkForComodification();
                Vector.this.remove(lastRet);
                expectedModCount = modCount;
            }
            cursor = lastRet;
            lastRet = -1;
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }

```

该方法里的内容很简单，就是判断了一下当前容器元素数量是否与创建迭代器时记录的数量相等，若不相等，则抛出 `ConcurrentModificationException` 异常，可见在 `next()` 和 `remove()` 方法中在执行具体操作前都会先执行这个检查，并且在检查到容器元素数量改变时抛出异常，这就解释了为什么多线程操作数组时，同步容器会有不一致性的风险。所以，为了保证一致性，使用者常需额外对整个迭代过程加锁加锁：

```java

synchronize(mVector){
    for(Object obj : mVector){
        //do something for everobj
    }
}

```
> 注意：同步容器要实现一边遍历一边修改数组时，要使用 iterator 中对应的修改方法，如 add(), remove() 等，否则若直接使用容器类中的修改方法，同理会报 ConcurrentModificationException 异常。

## COW 读写原理
COW 被称为「写时复制」数组，任何修改操作( add, remove 等)都会通过新建一个底层数组的拷贝来实现。
下面我们从**读取**和**修改**两方面看看 COW 的实现原理。

### 1. COW 迭代

COW 的迭代和其他容器是类似的，都是先获取迭代器。不过这里返回的是一个 `COWIterator` ，同时，创建该迭代器时，需传入实际存放元素的底层数组的一个「snapshot」（译为「快照」）

```java

public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
         /** The array, accessed only via getArray/setArray. */
        private volatile transient Object[] array;
        
        final Object[] getArray() {
            return array;
        }
        
        public Iterator<E> iterator() {
            return new COWIterator<E>(getArray(), 0);
        }
    }
```

下面仔细看看 COWIterator 的实现，并与前面 Vector 的迭代器对比看有啥不同。
```java

    private static class COWIterator<E> implements ListIterator<E> {
        /** Snapshot of the array **/
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        public boolean hasNext() {
            return cursor < snapshot.length;
        }

        public boolean hasPrevious() {
            return cursor > 0;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (! hasPrevious())
                throw new NoSuchElementException();
            return (E) snapshot[--cursor];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; <tt>remove</tt>
         *         is not supported by this iterator.
         */
        public void remove() {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; <tt>set</tt>
         *         is not supported by this iterator.
         */
        public void set(E e) {
            throw new UnsupportedOperationException();
        }

        /**
         * Not supported. Always throws UnsupportedOperationException.
         * @throws UnsupportedOperationException always; <tt>add</tt>
         *         is not supported by this iterator.
         */
        public void add(E e) {
            throw new UnsupportedOperationException();
        }
    }
```
相比 Vector 的迭代器， COW 的迭代器主要有以下不同：
- 初始化时传入容器底层数组引用值作为「快照」。
- 遍历时对该「快照」进行索引取值。
- 迭代器不支持 add(), set(), remove() 方法，调用会抛出 UnsupportedOperationException 。

可注意到，COW 的迭代器在调用遍历方法 next() 时，已经*不需要*调用类似 Vector 迭代中当中的 checkForComodification() 方法来检查遍历中途有无数组修改并在修改时抛出 ConcurrentModificationException 了。

这是为什么呢？难道 COW 在迭代过程中，不会受数组修改的影响吗？接着看 COW 的元素修改源码来一探究竟。

### 2. COW 修改

我们看一下 COW 的 add(E e) 方法
```java

 /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

先直接看 try 里面的内容，代码很清晰的做了 3 件事：
- 创建一个新的数组 newElements, 并使用 Arrays 的 copyOf 方法将容器底层数组的值赋值到该新数组，注意这里的复制是值复制，是浅拷贝。
- 在新数组的末尾插入新元素 e。
- 使用 setArray 方法将容器底层数组指向新数组。

这样就很清楚了，原来 COW 每次 add() 的时候，会新创建复制一个数组，等修改完成后，会将容器底层数组的引用指向新数组。

而迭代时只读取迭代器创建时底层数组的快照，并且该快照在在整个迭代器生命周期不会更改，所以迭代本身与数组的修改互不影响，读的操作不需同步也不会造成异常。但这样将导致无法保证数组的「实时一致性」，不过能保证「最终一致性」。

另，在 try 的前面和 finally 里面分别加锁和释放锁，再观察其他修改数组的操作如 remove() 等，均发现有加锁和释放锁的操作，因此可确定 COW 修改数组的操作均为线程同步的。

## COW 使用时机

从上文可知，COW 的**缺点**明显，每次修改数组均需新建一个底层数组拷贝，这显然会对性能有消耗。

相对而言，COW 的**优点**为遍历时不需对遍历过程加锁，遍历效率高且拓展性强。

因此，COW 更适用于「**读比写多**」且不需要「**实时一致性**」的情况。





