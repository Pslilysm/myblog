---
title: ArrayMap源码分析
date: 2020-10-31 10:15:00
tags:
    - Map
    - Java
    - Android
---
## 概述

在移动设备端内存资源很珍贵，HashMap为实现快速查询带来了很大内存的浪费。为此，2013年5月20日Google工程师Dianne Hackborn在Android系统源码中新增ArrayMap类，从Android源码中发现有不少提交专门把之前使用HashMap的地方改用ArrayMap，不仅如此，大量的应用开发者中广为使用。

ArrayMap是Android专门针对内存优化而设计的，用于取代Java API中的HashMap数据结构。为了更进一步优化key是int类型的Map，Android再次提供效率更高的数据结构SparseArray，可避免自动装箱过程。对于key为其他类型则可使用ArrayMap。HashMap的查找和插入时间复杂度为O(1)的代价是牺牲大量的内存来实现的，而SparseArray和ArrayMap性能略逊于HashMap，但更节省内存。

### 初始化ArrayMap
``` java
    Map<K,V> map = new ArrayMap<>();

    public class ArrayMap<K, V> extends SimpleArrayMap<K, V> implements Map<K, V> {
    
        public ArrayMap() {
            super();
        }

    }

    public class SimpleArrayMap<K, V> {
    
        private static final boolean CONCURRENT_MODIFICATION_EXCEPTIONS = true;

        private static final int BASE_SIZE = 4; // 缓存大小为4或8的数组
        private static final int CACHE_SIZE = 10; // 缓存数组的最多数量 大小4 和 8 的数组分别为10个，所以最多有20个数组缓存

        static @Nullable Object[] mBaseCache; // 长度为4的缓存
        static int mBaseCacheSize;
        static @Nullable Object[] mTwiceBaseCache; // 长度为8的缓存
        static int mTwiceBaseCacheSize;

        int[] mHashes;         // 由key的hashcode所组成的数组
        Object[] mArray;       // 由key-value对所组成的数组，是mHashes大小的2倍
        int mSize;             // key的个数

    }
```
1）ArrayMap的数据结构如下
* mHashes是一个记录所有key的hashcode值组成的数组，是从小到大的排序方式；
* mArray是一个记录着key-value键值对所组成的数组，是mHashes大小的2倍；
其中mSize记录着该ArrayMap对象中有多少对数据，执行put()或者append()操作，则mSize会加1，执行remove()，则mSize会减1。mSize往往小于mHashes.length，如果mSize大于或等于mHashes.length，则说明mHashes和mArray需要扩容。

2）ArrayMap类有两个非常重要的静态成员变量mBaseCache和mTwiceBaseCacheSize，用于ArrayMap所在进程的全局缓存功能：
* mBaseCache：用于缓存大小为4的ArrayMap，mBaseCacheSize记录着当前已缓存的数量，超过10个则不再缓存；
* mTwiceBaseCacheSize：用于缓存大小为8的ArrayMap，mTwiceBaseCacheSize记录着当前已缓存的数量，超过10个则不再缓存。

为了减少频繁地创建和回收Map对象，ArrayMap采用了两个大小为10的缓存队列来分别保存大小为4和8的Map对象。为了节省内存有更加保守的内存扩张以及内存收缩策略。 接下来分别说说缓存机制和扩容机制。

### 缓存
ArrayMap是专为Android优化而设计的Map对象，使用场景比较高频，很多场景可能起初都是数据很少，为了减少频繁地创建和回收，特意设计了两个缓存池，分别缓存大小为4和8的ArrayMap对象。要理解缓存机制，那就需要看看内存分配(allocArrays)和内存释放(freeArrays)。

#### SimpleArrayMap.freeArrays()
``` java
    private static void freeArrays(final int[] hashes, final Object[] array, final int size) {
        if (hashes.length == (BASE_SIZE*2)) {
            // 如果要缓存大小为8的数组
            synchronized (SimpleArrayMap.class) {
                if (mTwiceBaseCacheSize < CACHE_SIZE) {
                    // 当前缓存容量小于上限，继续缓存
                    array[0] = mTwiceBaseCache; // 将原来的缓存头放在第0位，所以可以通过缓存头的第0位获取第二个缓存
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        // 清空数据
                        array[i] = null;
                    }
                    mTwiceBaseCache = array; // 将缓存头指向新的缓存
                    mTwiceBaseCacheSize++;
                    if (DEBUG) System.out.println(TAG + " Storing 2x cache " + array
                            + " now have " + mTwiceBaseCacheSize + " entries");
                }
            }
        } else if (hashes.length == BASE_SIZE) {
            // 如果要缓存大小为4的数组
            synchronized (SimpleArrayMap.class) {
                if (mBaseCacheSize < CACHE_SIZE) {
                    array[0] = mBaseCache;
                    array[1] = hashes;
                    for (int i=(size<<1)-1; i>=2; i--) {
                        array[i] = null;
                    }
                    mBaseCache = array;
                    mBaseCacheSize++;
                    if (DEBUG) System.out.println(TAG + " Storing 1x cache " + array
                            + " now have " + mBaseCacheSize + " entries");
                }
            }
        }
    }
```
缓存池为一个单链表结构，每个缓存数据的第0位都代表着下一个缓存。

freeArrays()触发时机:
* 当执行removeAt()移除最后一个元素的情况
* 当执行clear()清理的情况
* 当执行ensureCapacity()在当前容量小于预期容量的情况下, 先执行allocArrays,再执行freeArrays
* 当执行put()在容量满的情况下, 先执行allocArrays, 再执行freeArrays

#### SimpleArrayMap.allocArrays()
``` java
    /**
     * 申请分配数组给mHashes和mArray
     * @param size 分配的大小
     */
    private void allocArrays(final int size) {
        if (size == (BASE_SIZE*2)) {
            // 如果要分配的大小为8
            // 去缓存池里找数组
            synchronized (SimpleArrayMap.class) {
                if (mTwiceBaseCache != null) {
                    // 当前缓冲池有缓存
                    final Object[] array = mTwiceBaseCache;
                    mArray = array; // 将缓存赋值给mArray
                    // array[0] 为下一个缓存, // array[1]为一个int[8]
                    mTwiceBaseCache = (Object[])array[0]; // 将缓存头指向下一个缓存
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;  // 对mArray清空数据
                    mTwiceBaseCacheSize--;
                    if (DEBUG) System.out.println(TAG + " Retrieving 2x cache " + mHashes
                            + " now have " + mTwiceBaseCacheSize + " entries");
                    return;
                }
            }
        } else if (size == BASE_SIZE) {
            // 如果要分配的大小为4
            // 去缓存池里找数组
            synchronized (SimpleArrayMap.class) {
                if (mBaseCache != null) {
                    final Object[] array = mBaseCache;
                    mArray = array;
                    mBaseCache = (Object[])array[0];
                    mHashes = (int[])array[1];
                    array[0] = array[1] = null;
                    mBaseCacheSize--;
                    if (DEBUG) System.out.println(TAG + " Retrieving 1x cache " + mHashes
                            + " now have " + mBaseCacheSize + " entries");
                    return;
                }
            }
        }

        // 没有缓存可用， 直接申请内存
        mHashes = new int[size];
        mArray = new Object[size<<1];
    }
```
allocArrays触发时机：
* 当执行ArrayMap(int capacity)的构造函数的情况
* 当执行removeAt()在满足容量收紧机制的情况
* 当执行ensureCapacity()在当前容量小于预期容量的情况下, 先执行allocArrays,再执行freeArrays
* 当执行put()在容量满的情况下, 先执行allocArrays, 再执行freeArrays

### 对容量的调整
#### 扩张
``` java
    public V put(K key, V value) {
        final int osize = mSize;
        ...
        if (osize >= mHashes.length) {
            // 如果 osize >= 8 则将 n * 1.5
            // 如国 4 <= osize < 8 则 n = 8，否则为4
            final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                    : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);
            allocArrays(n);
        }
        ...
    }
```
当mSize大于或等于mHashes数组长度时则扩容，完成扩容后需要将老的数组拷贝到新分配的数组，并释放老的内存。
* 当map个数满足条件 osize<4时，则扩容后的大小为4；
* 当map个数满足条件 4<= osize < 8时，则扩容后的大小为8；
* 当map个数满足条件 osize>=8时，则扩容后的大小为原来的1.5倍；

#### 收缩
``` java
    public V removeAt(int index) {
        final int osize = mSize;
        final int nsize;
        if (osize <= 1) {
            ...
        }else {
            nsize = osize - 1;
            if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
                final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);
                ...
                // 分配更小的内存
                allocArrays(n);
            }
        }
    }
```
当数组内存的大小大于8，且已存储数据的个数mSize小于数组空间大小的1/3的情况下，需要收紧数据的内容容量，分配新的数组，老的内存靠虚拟机自动回收。
* 如果mSize<=8，则设置新大小为8；
* 如果mSize> 8，则设置新大小为mSize的1.5倍。

### 基本方法
#### put()
``` java
    public V put(K key, V value) {
        final int osize = mSize;
        final int hash;
        int index;
        if (key == null) {
            hash = 0;
            index = indexOfNull();
        } else {
            hash = key.hashCode();
            // 二分法搜索key
            // 下面有对该方法的注释
            index = indexOf(key, hash);
        }
        // 如果当前map里已有相同的key
        if (index >= 0) {
            index = (index<<1) + 1; // 将index转化为mArray中的index，做法为 * 2 + 1；
            final V old = (V)mArray[index];
            mArray[index] = value;
            return old;
        }

        // 取反获得预计要插入的位置
        index = ~index;
        if (osize >= mHashes.length) {
            final int n = osize >= (BASE_SIZE*2) ? (osize+(osize>>1))
                    : (osize >= BASE_SIZE ? (BASE_SIZE*2) : BASE_SIZE);

            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            // 扩容
            allocArrays(n);

            if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                throw new ConcurrentModificationException();
            }

            if (mHashes.length > 0) {
                // 将原数组的数据copy过去
                System.arraycopy(ohashes, 0, mHashes, 0, ohashes.length);
                System.arraycopy(oarray, 0, mArray, 0, oarray.length);
            }
            // 将原数组释放到缓存池
            freeArrays(ohashes, oarray, osize);
        }

        if (index < osize) {
            // 如果要插入的位置在当前数组范围内，则将index(include)之后的元素往后移一位
            System.arraycopy(mHashes, index, mHashes, index + 1, osize - index);
            System.arraycopy(mArray, index << 1, mArray, (index + 1) << 1, (mSize - index) << 1);
        }

        if (CONCURRENT_MODIFICATION_EXCEPTIONS) {
            if (osize != mSize || index >= mHashes.length) {
                throw new ConcurrentModificationException();
            }
        }

        mHashes[index] = hash;
        mArray[index<<1] = key;
        mArray[(index<<1)+1] = value;
        mSize++;
        return null;
    }

    /**
     * 返回正数说明找到了该key，返回负数说明没找到
     */
    int indexOf(Object key, int hash) {
        final int N = mSize;
        ...
        int index = binarySearchHashes(mHashes, N, hash);
        if (index < 0) {
            return index;
        }

        if (key.equals(mArray[index<<1])) {
            return index;
        }

        // 看下index + 1 的hash会不会相等
        int end;
        for (end = index + 1; end < N && mHashes[end] == hash; end++) {
            if (key.equals(mArray[end << 1])) return end;
        }

        // 看下index - 1 的hash会不会相等
        for (int i = index - 1; i >= 0 && mHashes[i] == hash; i--) {
            if (key.equals(mArray[i << 1])) return i;
        }

        // 返回负数
        return ~end;
    }
    
    static int binarySearch(int[] array, int size, int value) {
        int lo = 0;
        int hi = size - 1;

        while (lo <= hi) {
            int mid = (lo + hi) >>> 1;
            int midVal = array[mid];

            if (midVal < value) {
                lo = mid + 1;
            } else if (midVal > value) {
                hi = mid - 1;
            } else {
                return mid;  // value found
            }
        }
        return ~lo;  // 返回负数，说明没找到
    }
```
put()设计巧妙地将修改已有数据对(key-value) 和插入新的数据对合二为一个方法，主要是依赖indexOf()过程中采用的二分查找法， 当找到相应key时则返回正值，但找不到key则返回负值，按位取反所对应的值代表的是需要插入的位置index。

put()在插入时，如果当前数组内容已填充满时，则会先进行扩容，再通过System.arraycopy来进行数据拷贝，最后在相应位置写入数据。

#### removeAt()
``` java
    public V removeAt(int index) {
        final Object old = mArray[(index << 1) + 1];
        final int osize = mSize;
        final int nsize;
        // 如果现在的大小<=1
        if (osize <= 1) {
            // 将mHashes和mArray释放到缓存池
            freeArrays(mHashes, mArray, osize);
            mHashes = ContainerHelpers.EMPTY_INTS;
            mArray = ContainerHelpers.EMPTY_OBJECTS;
            nsize = 0;
        } else {
            nsize = osize - 1;
            // 如果容量太大
            if (mHashes.length > (BASE_SIZE*2) && mSize < mHashes.length/3) {
                // 收缩内存
                final int n = osize > (BASE_SIZE*2) ? (osize + (osize>>1)) : (BASE_SIZE*2);

                final int[] ohashes = mHashes;
                final Object[] oarray = mArray;
                allocArrays(n);

                if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
                    throw new ConcurrentModificationException();
                }

                if (index > 0) {
                    // 将index(exclude)之前的数据copy到新数组
                    System.arraycopy(ohashes, 0, mHashes, 0, index);
                    System.arraycopy(oarray, 0, mArray, 0, index << 1);
                }
                if (index < nsize) {
                    // 将index(exclude)之后的数据copy到新数组
                    System.arraycopy(ohashes, index + 1, mHashes, index, nsize - index);
                    System.arraycopy(oarray, (index + 1) << 1, mArray, index << 1,
                            (nsize - index) << 1);
                }
            } else {
                if (index < nsize) {
                    System.arraycopy(mHashes, index + 1, mHashes, index, nsize - index);
                    System.arraycopy(mArray, (index + 1) << 1, mArray, index << 1,
                            (nsize - index) << 1);
                }
                mArray[nsize << 1] = null;
                mArray[(nsize << 1) + 1] = null;
            }
        }
        if (CONCURRENT_MODIFICATION_EXCEPTIONS && osize != mSize) {
            throw new ConcurrentModificationException();
        }
        mSize = nsize;
        return (V)old;
    }
```
remove()过程：通过二分查找key的index，再根据index来选择移除动作；当被移除的是ArrayMap的最后一个元素，则释放该内存，否则只做移除操作，这时会根据容量收紧原则来决定是否要收紧，当需要收紧时会创建一个更小内存的容量。

#### clear()
``` java
    public void clear() {
        if (mSize > 0) {
            final int[] ohashes = mHashes;
            final Object[] oarray = mArray;
            final int osize = mSize;
            mHashes = ContainerHelpers.EMPTY_INTS;
            mArray = ContainerHelpers.EMPTY_OBJECTS;
            mSize = 0;
            // 将mHashes和mArray释放到缓存池
            freeArrays(ohashes, oarray, osize);
        }
        if (CONCURRENT_MODIFICATION_EXCEPTIONS && mSize > 0) {
            throw new ConcurrentModificationException();
        }
    }
```
## 总结
ArrayMap在Android中能节省内存的主要原因还是他的缓存机制，注意以下几点来高效利用ArrayMap：
* 实例化ArrayMap的过程中如果预计容量为8以下，那么通过8或者4来构造ArrayMap。
* 在一个ArrayMap对象不会被继续使用时将他clear。
* 千万不可并发操作ArrayMap，ArrayMap对并发异常的检测不是很完善，所以因为并发导致的其他问题很难找到源头。