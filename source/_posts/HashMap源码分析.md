---
title: HashMap源码分析
date: 2020-10-31 15:52:00
tags:
    - Map
    - Java
---
## 概述

HashMap是查询非常快的Map，他的查询只需要通过一次计算就能获取key的位置，他是如何实现的呢，他的数据结构又是怎样的呢？今天便来一探究竟。

### 初始化HashMap
``` java
    public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

        static final float DEFAULT_LOAD_FACTOR = 0.75f;
        final float loadFactor;

        /**
         * The next size value at which to resize (capacity * load factor).
         */
        int threshold;
    
        public HashMap(int initialCapacity, float loadFactor) {
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal initial capacity: " +
                                                    initialCapacity);
            if (initialCapacity > MAXIMUM_CAPACITY)
                initialCapacity = MAXIMUM_CAPACITY;
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal load factor: " +
                                                    loadFactor);
            this.loadFactor = loadFactor;
            this.threshold = tableSizeFor(initialCapacity);
        }

        public HashMap(int initialCapacity) {
            this(initialCapacity, DEFAULT_LOAD_FACTOR);
        }

        /**
         * Constructs an empty <tt>HashMap</tt> with the default initial capacity (16) and the default load factor (0.75).
         */
        public HashMap() {
            this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
        }

        /**
         * Returns a power of two size for the given target capacity.
         */
        static final int tableSizeFor(int cap) {
            int n = cap - 1;
            n |= n >>> 1;
            n |= n >>> 2;
            n |= n >>> 4;
            n |= n >>> 8;
            n |= n >>> 16;
            return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
        }

    }
```
我们一般使用的都是无参的构造方法，无参的构造方法只会对loadFactor赋值。HashMap(int initialCapacity, float loadFactor)这个构造方法会通过tableSizeFor方法计算threshold，第一次看到这个方法时可能会有点懵，不急，先传几个参数模拟下他的步骤。
传入8，也就是0b1000， n = cap-1 -> 0b0111
接下来的步骤就是把第一个1后面的全补为1，最后return n+1。也就是 0b0111 + 1 = 8；如果你传入3，那么返回0b0011 + 1 =  4；如果传入13，那么返回0b1111+1 = 16。
那么为什么一进来要将cap - 1呢，因为如果传进来的数刚好是2的幂次方的话，这个方法就应该返回传进来的值，所以要去-1，不然返回的就是cap * 2。
有没有发现很神奇，这个方法返回的值一定是2的幂次方，而且这个2的幂次方刚好比传进来的值大。

### putVal()
``` java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next; // Node为单链表结构，通过next链接下一个Node

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            // 当前tab为空数组，进行初始化
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 通过hash获取node的index
            // 将该tab[index]中的数据赋值给p，如果p == null，则直接newNode并存放进去
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果p的hash等于传入的hash并且key也相等
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // hash或者key不相等，向p的末尾插入node
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // p.next == null 达到末尾
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            // 单链表的长度大于等于8 将tab[i]转化为红黑树
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 在单链表中找到了相同key hash的Node
                        break;
                    // e = p.next
                    // p = e -> p.next
                    // 所以这里是将p指向下一个Node
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                // 设置新value到node中
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
    
```
#### 红黑树
``` java
    static final int MIN_TREEIFY_CAPACITY = 64;

    final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 当前table容量小于64 去扩容
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            // hd也就是head 红黑树的头
            // tl为上一次循环的TreeNode
            TreeNode<K,V> hd = null, tl = null;
            do {
                // 将Node转化为TreeNode
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    // 第一次do tl为空 把第一个Node赋值给hd
                    hd = p;
                else {
                    // 第二次do 进入
                    p.prev = tl; // 将上一个TreeNode赋值给p的prev
                    tl.next = p; // 将p赋值给上一个TreeNode的next
                }
                tl = p;
            } while ((e = e.next) != null); // 循环Node的单链表
            // 将红黑树的head赋值给tab[index]
            if ((tab[index] = hd) != null)
                // 对head进行树化
                hd.treeify(tab);
        }
    }

    final void treeify(Node<K,V>[] tab) {
        TreeNode<K,V> root = null;
        for (TreeNode<K,V> x = this, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;  // x = next = x.next 也就是在下个循环x成了x的下一个TreeNode
            x.left = x.right = null;
            if (root == null) {
                // 第一次循环进入
                x.parent = null;
                x.red = false;
                root = x;
            }
            else {
                K k = x.key; // 当前TreeNode的 k
                int h = x.hash;  // 当前TreeNode的 hash
                Class<?> kc = null;
                for (TreeNode<K,V> p = root;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        // 如果root的hash 大于 当前TreeNode的 hash， dir = -1; 将会把x放在树的左边
                        // 可见树的循序是根据hash的大小来排的
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                                (kc = comparableClassFor(k)) == null) ||
                                (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);

                    TreeNode<K,V> xp = p;
                    // 通过dir来判断是在树的左边还是右边                   
                    // 第三次循环 如果x应该在的方向和上次循环一样，那么不会进入if
                    // 直到方向不一样将root的左右填满才会再次进入if
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        // 为空，可以将元素放入
                        x.parent = xp; 
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        // 从新计算root
                        root = balanceInsertion(root, x);
                        // 得到root = root，进入第三次循环
                        break;
                    }
                }
            }
        }
        moveRootToFront(tab, root);
    }

    static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                    TreeNode<K,V> x) {
        x.red = true;
        for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
            // 不进入
            if ((xp = x.parent) == null) {
                x.red = false;
                return x;
            }
            // xpp = xp.parent = x.parent.parent = root.parent = null
            // 所以返回root 继续看treeify
            else if (!xp.red || (xpp = xp.parent) == null)
                return root;
            if (xp == (xppl = xpp.left)) {
                if ((xppr = xpp.right) != null && xppr.red) {
                    xppr.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.right) {
                        root = rotateLeft(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateRight(root, xpp);
                        }
                    }
                }
            }
            else {
                if (xppl != null && xppl.red) {
                    xppl.red = false;
                    xp.red = false;
                    xpp.red = true;
                    x = xpp;
                }
                else {
                    if (x == xp.left) {
                        root = rotateRight(root, x = xp);
                        xpp = (xp = x.parent) == null ? null : xp.parent;
                    }
                    if (xp != null) {
                        xp.red = false;
                        if (xpp != null) {
                            xpp.red = true;
                            root = rotateLeft(root, xpp);
                        }
                    }
                }
            }
        }
    }

```