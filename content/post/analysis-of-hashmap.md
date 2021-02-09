+++
title = "Analysis of Hashmap"
date = "2018-07-02"
slug = "2018/07/02/analysis-of-hashmap"
Categories = ["java", "algorithm", "notes"]
+++

``HashMap``是存储key-value的集合，底层采用``Node<K,V>[] table``实现，初始大小为2^4即16。

<!-- more -->

```java
public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }
```

``put``操作首先根据``key``的hash值来找到要插入的``index``，如果存在相同``key``的元素则替换，否则插入。

```java
final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
```

``hash``方法采用的是``sun.misc.Hashing``中的方法。

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
        if ((size >= threshold) && (null != table[bucketIndex])) {
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }
```

```java
void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }
```

添加元素的时候会检查是否需要扩容，如果需要会将数组大小增大一倍，同时进行``rehash``来将之前的元素转移到现在的数组中来。

如果不需要扩容，直接添加元素

```java
void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }
```

将``table``数组``index``位置元素指向插入元素，插入元素作为链表头。

``get``操作和``put``操作差不多，根据``key``查找``Entry``

```java
final Entry<K,V> getEntry(Object key) {
        if (size == 0) {
            return null;
        }

        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

``table``数组的长度是2^n，这样2^n-1的二进制表示每一位都是1，方便进行index计算。

```java
static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```

``LinkedHashMap``的``Entry``除了有一个``next``来处理冲突，还有``before``和``after``来将所有元素连接成一个双向循环链表。

```java
// LinkedHashMap
void createEntry(int hash, K key, V value, int bucketIndex) {
        HashMap.Entry<K,V> old = table[bucketIndex];
        Entry<K,V> e = new Entry<>(hash, key, value, old);
        table[bucketIndex] = e;
        e.addBefore(header);
        size++;
    }
```

```java
/**
         * Inserts this entry before the specified existing entry in the list.
         */
        private void addBefore(Entry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        /**
         * This method is invoked by the superclass whenever the value
         * of a pre-existing entry is read by Map.get or modified by Map.set.
         * If the enclosing Map is access-ordered, it moves the entry
         * to the end of the list; otherwise, it does nothing.
         */
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }
```

``Entry``的``addBefore``将元素添加至双向循环链表的尾部，``recordAccess``将元素从双向循环链表原来的位置移除，重新添加到链表尾部。如果key元素已经存在Map中，在``put``时会替换value，同时``recordAccess``，``recordAccess``在``HashMap``的``Entry``中是空实现，在``LinkedHashMap``中进行移除到链表尾部的操作。``recordAccess``还在``LinkedHashMap``的``get``方法中被调用，这样每次执行``get``操作返回元素的同时将``Entry``移动到链表尾部。

``WeakHashMap``的``Entry``是``WeakReference``的子类，创建的时候和``ReferenceQueue``进行关联，referent是key，当key被回收时将移除key对应的entry。

```java
/**
     * Expunges stale entries from the table.
     */
    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                @SuppressWarnings("unchecked")
                    Entry<K,V> e = (Entry<K,V>) x;
                int i = indexFor(e.hash, table.length);

                Entry<K,V> prev = table[i];
                Entry<K,V> p = prev;
                while (p != null) {
                    Entry<K,V> next = p.next;
                    if (p == e) {
                        if (prev == e)
                            table[i] = next;
                        else
                            prev.next = next;
                        // Must not null out e.next;
                        // stale entries may be in use by a HashIterator
                        e.value = null; // Help GC
                        size--;
                        break;
                    }
                    prev = p;
                    p = next;
                }
            }
        }
```

key被回收时``Entry``会被放入``ReferenceQueue``中。在调用``size()``和``resize()``方法时会调用``expungeStaleEntries``方法。

一般情况下，一个对象被标记为垃圾（并不代表被回收了）后会被加入引用队列。

对于虚引用来说，它指向的对象只有被回收后才会加入引用队列，所以可以作为记录该引用指向的对象是否被回收。

## reference

+ [Android Hashing.java](https://android.googlesource.com/platform/libcore/+/8f9c9cae00ad906c39891890f7b9d7a0bc453c0a%5E2..8f9c9cae00ad906c39891890f7b9d7a0bc453c0a/)
+ [JDK Hashing.java](http://hg.openjdk.java.net/jdk7u/jdk7u6/jdk/file/8c2c5d63a17e/src/share/classes/sun/misc/Hashing.java)
+ [Reference Queues](http://learningviacode.blogspot.com/2014/02/reference-queues.html)

