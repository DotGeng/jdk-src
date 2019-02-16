# jdk-src
jdk 源码阅读

# TreeMap 源码阅读
&emsp;&emsp;TreeMap是用红黑树实现的，插入数据后添加红黑树节点并进行红黑树的调整，删除数据后也同样进行红黑树的调整，查询数据时，不是用hashcode来进行查询，而是使用红黑树查询。TreeMap没有HashMap的扩容机制，所有的数据都存在一棵红黑树上。红黑树的数据节点是封装数据的Entry对象。
## Entry定义
```
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left;
    Entry<K,V> right;
    Entry<K,V> parent;
    boolean color = BLACK;

    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
    // ... 省略其他方法
}
```
其中Entry就是红黑树的数据节点。

## 构造函数与成员变量
## 成员变量
```
// 比较器
private final Comparator<? super K> comparator;

// 根节点
private transient Entry<K,V> root;

// 大小
private transient int size = 0;

```
## 构造函数
```
// 默认构造，比较器采用key的自然比较顺序
public TreeMap() {
    comparator = null;
}

// 指定比较器
public TreeMap(Comparator<? super K> comparator) {
    this.comparator = comparator;
}

// 从Map集合导入初始数据
public TreeMap(Map<? extends K, ? extends V> m) {
    comparator = null;
    putAll(m);
}

// 从SortedMap导入初始数据
public TreeMap(SortedMap<K, ? extends V> m) {
    comparator = m.comparator();
    try {
        buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }
}
```
一共四个构造函数，均没有像HashMap那样指定大小的构造函数，这里也能猜测到TreeMap没有resize 的过程。

## 增加一个元素
红黑树最复杂的地方就在于增删了，我们就从增加一个元素开始分析：
```
public V put(K key, V value) {
    // 暂存根节点
    Entry<K,V> t = root;

    // 根节点空，就是还没有元素
    if (t == null) {
        compare(key, key); // type (and possibly null) check
        // 新建一个元素，默认颜色黑色
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }

    // 根节点不为空，有元素时的情况
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    // 初始化时指定了comparator比较器
    if (cpr != null) {
        do {
            // 把t暂存到parent中
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                // 比较小，往左侧插入
                t = t.left;
            else if (cmp > 0)
                // 比较大，往右侧插入
                t = t.right;
            else
                // 一样大，所以就是更新当前值
                return t.setValue(value);
        } while (t != null);
    }
    else {
    // 使用key的比较器，while循环原理和上述一致
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }

    // 不断的比较，找到了没有相应儿子的节点
    //（cmp<0就是没有左儿子，cmp>0就是没有右儿子）
    Entry<K,V> e = new Entry<>(key, value, parent);
    // 把数据插入
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;

    // 新插入的元素破坏了红黑树规则，需要调整
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```
## 获取元素
TreeMap中的元素是有序的，当使用中序遍历时就可以得到一个有序的Set集合，所以获取元素可以采用二分法：
```
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```
除了获取某个元素外，还可以获取它的前一个元素与后一个元素：
```
// 获取前一个元素
static <K,V> Entry<K,V> predecessor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.left != null) {
        // 左侧最右边元素就是
        Entry<K,V> p = t.left;
        while (p.right != null)
            p = p.right;
        return p;
    } else {
        // 右侧最左边元素就是
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.left) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}

// 获取后一个元素
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    //...
}
```
## 总结
主要分析了TreeMap的put方法和get方法。

# HashMap 源码阅读
HashMap 是一个key value类型的集合，是线程不安全的。它底层基于hash表，用数据+链表的结构存储数据。JDK1.8中，还是用红黑树来管理冲突的Entry链表，从而提高从HashMap拿数据的效率。
HashMap的结构如下所示：
![图1](https://img-blog.csdnimg.cn/201902161708260.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MTg4MjU1,size_16,color_FFFFFF,t_70)

## HashMap的元素
我们要先看下其数据单元，因为HashMap有普通的元素，还有红黑树的元素，所以其数据单元定义有两个：
```
// 普通节点
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
    // ...
}

// 树节点，继承自LinkedHashMap.Entry
// 这是因为LinkedHashMap是HashMap的子类，也需要支持树化
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    // ...
}

// LinkedHashMap.Entry的实现
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
TreeNode定义了一些相关操作的方法,我们在使用的时候进行分析。
## 成员变量和构造函数

## 成员变量
```
// capacity初始值，为16，必须为2的次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// capacity的最大值，为2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// load factor，是指当容量被占满0.75时就需要rehash扩容
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 链表长度到8，就转为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 树大小为6，就转回链表
static final int UNTREEIFY_THRESHOLD = 6;

// 至少容量到64后，才可以转为树
static final int MIN_TREEIFY_CAPACITY = 64;

// 保存所有元素的table表
transient Node<K,V>[] table;

// 通过entrySet变量，提供遍历的功能
transient Set<Map.Entry<K,V>> entrySet;

// 下一次扩容值
int threshold;

// load factor
final float loadFactor;
```
# 构造函数
HashMap有多个构造函数，主要支持配置容量capacity和load factor，以及从其他Map集合获取初始化数据。
```
public HashMap(int initialCapacity, float loadFactor) {
    // ... 参数校验    
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
这里涉及到两个方法：
tableSizeFor 和 putMapEntries，putMapEntries是依次插入元素的，所以这里我们先不做分析，后续分析put方法时，自然就明白了。现在先看下tableSizeFor方法。
```
    /**
     * Returns a power of two size for the given target capacity.
     * 找到距离cap参数最近的2的次幂
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
```
即找到距离cap参数最近的2的次幂.

## 重要方法
```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
其中 key与其高16位异或，降低hash碰撞，提高put效率。
## put方法
put方法的具体实现在putVal中，源码如下：
```
// 参数onlyIfAbsent表示是否替换原值
// 参数evict我们可以忽略它，它主要用来区别通过put添加还是创建时初始化数据的
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 空表，需要初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // resize()不仅用来调整大小，还用来进行初始化配置
        n = (tab = resize()).length;
    // (n - 1) & hash这种方式也熟悉了吧？都在分析ArrayDeque中有体现
    //这里就是看下在hash位置有没有元素，实际位置是hash % (length-1)
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 将元素直接插进去
        tab[i] = newNode(hash, key, value, null);
    else {
        //这时就需要链表或红黑树了
        // e是用来查看是不是待插入的元素已经有了，有就替换
        Node<K,V> e; K k;
        // p是存储在当前位置的元素
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p; //要插入的元素就是p，这说明目的是修改值
        // p是一个树节点
        else if (p instanceof TreeNode)
            // 把节点添加到树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 这时候就是链表结构了，要把待插入元素挂在链尾
            for (int binCount = 0; ; ++binCount) {
                //向后循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表比较长，需要树化，
                    // 由于初始即为p.next，所以当插入第9个元素才会树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到了对应元素，就可以停止了
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 继续向后
                p = e;
            }
        }
        // e就是被替换出来的元素，这时候就是修改元素值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 默认为空实现，允许我们修改完成后做一些操作
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // size太大，达到了capacity的0.75，需要扩容
    if (++size > threshold)
        resize();
    // 默认也是空实现，允许我们插入完成后做一些操作
    afterNodeInsertion(evict);
    return null;
}
```
上述源代码非常依赖与resize 方法，resize方法源码如下：
```
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 大小超过了2^30，就不会进行扩容了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 新容量必须小于最大容量，且就容量也必须大于默认的初始化容量，才会进行扩容
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        // 如果是初始化，则设置newCap = oldThr;
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            // 全部设置为默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
                // 扩容，重新初始化一个Node 数组
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 扩容完成，开始复制就数据到新的空间中
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        // 这是只有一个值的情况，则直接复制到新列表即可
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        // 如果已经是红黑树，则重新规划树，如果树的size很小，默认为6，就退化为链表
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 复制链表中的数据到新数组中
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //oldCap是2的次幂，所以除了最高位为1以外其他位都是0
                            // 所以和它按位与的结果为0，说明hash比它小，原表有这个位置
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
                        // 挂在与原表相同的位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        // 挂在扩容后的空间中
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
从上述代码可以看到，Jdk1.8的resize方法，不会再像Jdk1.7那样，再多线程的情况下出现死循环（https://www.cnblogs.com/dongguacai/p/5599100.html）。

## remove 方法

```
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

和插入一样，其实际的操作在removeNode方法中完成，我们看下其实现：
```
// matchValue是说只有value值相等时候才可以删除，我们是按照key删除的，所以可以忽略它。
    // movable是指是否允许移动其他元素，这里是和TreeNode相关的
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            // 不同情况下获取待删除的node节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // 根据找到的node节点进行删除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    // 从红黑树中删除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    // 从链表头部删除
                    tab[index] = node.next;
                else
                    // 从链表中删除
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```
## 获取一个元素
```
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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
逻辑和我们分析的增删很类似，再读起来就很简单了。

## 总结

HashMap是比较复杂的集合类了。合理的使用它能够在增删改查等方面都有很好的表现。在使用时要需要注意以下几点：
1 设计的key对象一定要实现hashCode方法，并尽可能保证均匀少重复。

2 由于树化过程会依次通过hash值、比较值和对象的hash值进行排序，所以key还可以实现Comparable，以方便树化时进行比较。

3 如果可以预先估计数量级，可以指定initial capacity，以减少rehash的过程。

4 虽然HashMap引入了红黑树，但它的使用是很少的，如果大量出现红黑树，说明数据本身设计的不合理，我们应该从数据源寻找优化方案。





