# jdk-src
jdk 源码阅读

# TreeMap 源码阅读
&emsp;&emsp;TreeMap是用红黑树实现的，插入数据后添加红黑树节点并进行红黑树的调整，删除数据后也同样进行红黑树的调整，查询数据时，不是用hashcode来进行查询，而是使用红黑树查询。TreeMap没有HashMap的扩容机制，所有的数据都存在一棵红黑树上。红黑树的数据节点是封号数据的Entry对象。
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


