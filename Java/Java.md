## 一、HashMap的常用方法解析


### get()


```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```


### getNode()


```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 哈希表可用且查询K对应的哈希表下标处不为空
    if ((tab = table) != null && (n = tab.length) > 0 && (first = tab[(n - 1) & hash]) != null) {
        // always check first node
        // 始终检查第一个节点，当容量大小合理时，发生冲突的情况较少
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 存在下一个节点
        if ((e = first.next) != null) {
            // 判断节点的类型，使用对应的查找方法
            if (first instanceof TreeNode)
                // 使用树结构形式的查找方法
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 循环判断
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```


### put()


```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```


### putVal()


```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 判断哈希表是否可用
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 判断PUT节点在哈希表的下标处是否已经有值
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 直接赋值
        tab[i] = newNode(hash, key, value, null);
    else {
        // 已存在值，接下来的工作主要是定位PUT节点的位置
        Node<K,V> e; K k;
        // 当前节点与PUT节点的K是否等价
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            // 定位到
            e = p;
        // 判断当前节点是否为树节点
        else if (p instanceof TreeNode)
            // 以树结构的方式去处理
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 以链表结构的方式去处理
            for (int binCount = 0; ; ++binCount) {
                // 判断下一个节点是否为空
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 节点长度超过8个，将链表结构转化为树结构
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 当前节点与PUT节点的K是否等价
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 判断之前有没有值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 判断是否覆盖值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    ++modCount;
    // 已存储的数据量大于阈值则进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```


### resize()


```java
final Node<K,V>[] resize() {
    // 旧哈希表
    Node<K,V>[] oldTab = table;
    // 旧哈希表容量
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    // 旧哈希表阈值
    int oldThr = threshold;
    // 新哈希表容量 新哈希表阈值
    int newCap, newThr = 0;
    // 旧哈希表可用
    if (oldCap > 0) {
        // 旧哈希表已达到最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 将阈值设为最大值
            threshold = Integer.MAX_VALUE;
            // 返回旧的哈希表，不扩容
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
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
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
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


## 二、特点
### 线程不安全
#### JDK 1.7
多线程操作的时候，扩容引发的线程不安全，`resize()`方法，可能会导致死循环
1.8已解决该问题，使用尾插法
#### JDK1.8
多线程操作的时候可能会导致数据被覆盖
### Key和Value可以为空
```java
map.put(null,null);
```
## 三、HashMap的线程安全版本


### Collections.synchronizedMap(Map m)


#### 使用


```java
Map<String, Object> objectObjectMap = Collections.synchronizedMap(new HashMap<>());
Map<String, Object> objectObjectMap = Collections.synchronizedMap(new HashMap<>(),new Object());
```


### java.util.Collections.SynchronizedMap：


使用Sychronize对所有涉及读写的方法加锁，支持自定义锁对象。


##### 构造方法


```java
private static class SynchronizedMap<K,V>
    implements Map<K,V>, Serializable {
    private static final long serialVersionUID = 1978198479659022715L;

    // 实际存储的Map
    private final Map<K,V> m;
    // 用于同步的对象
    final Object      mutex;

    SynchronizedMap(Map<K,V> m) {
        this.m = Objects.requireNonNull(m);
        mutex = this;
    }

    // mutx 同步的对象，默认是本对象
    SynchronizedMap(Map<K,V> m, Object mutex) {
        this.m = m;
        this.mutex = mutex;
    }
}
```


##### 其他方法


```java
public V get(Object key) {
    synchronized (mutex) {return m.get(key);}
}

public V put(K key, V value) {
    synchronized (mutex) {return m.put(key, value);}
}
```


### Hashtable


通过在所有涉及读写Map的方法中使用`Synchronize`进行修饰来进行同步。


##### 使用


```java
Map<String, Object> objectObjectMap = new Hashtable<>();
```


### ConcurrentHashMap


通过CAS，Synchronize来进行同步


##### 使用


```java
ConcurrentHashMap<String, Object> objectObjectMap = new ConcurrentHashMap<>();
```


##### 原理
